+++
title = 'Zero-Downtime Deployments on Kubernetes: Probes, Rolling Updates and Helm'
description = "How we got zero downtime on every backend release using /healthz and /readyz endpoints, Kubernetes probes, RollingUpdate, and Helm's atomic deploys, release history and Capabilities."
date = 2026-10-05T13:00:00+08:00
draft = false
categories = ["DevOps"]
tags = ["kubernetes", "helm", "zero-downtime", "health-checks"]
+++

> **Disclaimer:** This post is AI-assisted: I used AI to help. All the code, manifests and scripts in it are examples, also generated with the help of AI to explain the ideas. They're not the actual code from my previous company, so please adjust them before using them anywhere real. The screenshots and outputs come from a small demo I rebuilt on my laptop with [kind](https://kind.sigs.k8s.io/), not from production.

At my previous company, I initiated an effort to improve how we deployed our backend, with one goal in mind, to achieve zero downtime for all our microservices during every new release. Kubernetes already had pretty much everything we needed. The missing piece was on the developer/application side: our code never told Kubernetes when it was actually ready to take traffic.

In this blog I want to go through it from a software engineer's point of view. What the backend exposes (`/healthz` and `/readyz`), how Kubernetes uses them through probes, how RollingUpdate builds on top of that, how the old pod shuts down without dropping requests, how Helm wraps the whole deploy so a bad release rolls itself back, and the scripts and runbook that made it the normal way to deploy.

# Background

Our backend has approximately 10 microservices, built on Express.js and Python running on an AKS cluster. Before this, every release came with a short window of errors.

To show what that window looked like, I rebuilt a small version of the old setup on my laptop. It's a single-node kind cluster running a demo `order-service` that talks to Postgres and Redis and takes about 8 seconds to start up (standing in for a real service connecting to its database and queues), plus a load generator sending it around 150 requests per second. The load generator prints one line per second: how many requests succeeded (`ok`), how many failed (`err`), which version answered, and what went wrong.

This is one deploy done the old way, with no readiness probe and no graceful shutdown:

![Load generator output during a deploy without probes or graceful shutdown: one line per second, with red error lines from 04:11:55 to 04:12:03 showing HTTP 503s, conn closed mid-request, conn refused, conn reset and timeouts, and seven seconds where no request succeeded.](/images/zero-downtime-deployments-kubernetes/before-unsafe-deploy.png)

In 9 seconds, 760 of 902 requests failed, and for 7 seconds in a row not a single request succeeded. Don't let the two white `ok=0 err=0` lines fool you. Nothing was healthy there: every request was stuck waiting on a pod that was already gone, and they all timed out together a second later.

So what was actually going wrong? As soon as a new pod's container started, Kubernetes considered it ready, and the Service started sending it requests. But the app wasn't ready at all. It was still busy connecting to the database, Redis, RabbitMQ and Kafka, so the first few requests it got just failed. Meanwhile, the old pod was being killed, sometimes with requests still in flight.

You can see both in the screenshot. The wall of `HTTP 503`s is the new pod answering before it was connected to anything. The `conn closed mid-request`, `conn reset`, `conn refused` and `timeout` errors at the start are the old pod dying with requests still in flight, or still being sent to it.

The thing is, Kubernetes has no idea when your app is ready unless the app tells it. That's what [probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) are for.

# Liveness vs Readiness

Kubernetes has two probes that matter here. They sound similar, but they ask different questions:

|  | `livenessProbe` → `/healthz` | `readinessProbe` → `/readyz` |
|---|---|---|
| Question | Is the process alive? | Can this pod serve a request right now? |
| What it checks | Nothing external | DB, Redis, RabbitMQ, Kafka |
| When it fails | Kubernetes restarts the container | Pod is removed from the Service and traffic goes to the other pods, no restart |
| Typical reason to fail | Deadlock, stuck event loop | Still starting up, dependency down |

In simple words: liveness is "should I restart you?" and readiness is "should I send you traffic?".

The one rule I'd tattoo somewhere: **liveness must not check dependencies.** Imagine the database has a 30-second blip and `/healthz` checks the database. Every pod of every service fails liveness at the same time, and Kubernetes restarts all of them. You just turned a small DB blip into a full outage, and restarting the pods doesn't fix the database anyway. The Kubernetes docs warn about exactly this: liveness probes should only indicate an unrecoverable failure like a deadlock, otherwise they can lead to cascading failures.

Readiness is the right place for dependency checks, because failing readiness only stops traffic. The pod keeps running, keeps retrying its connections, and gets added back to the Service by itself once the dependency is back.

# /healthz: Is the Process Alive?

`/healthz` does as little as possible. If the HTTP server can answer, the process is alive. That's it.

Our services are Express.js and Python, but I'll use Go in the examples for brevity. Nothing here is Go-specific.

```go
// Healthz only answers "is the process alive?". It must not touch any dependency.
func Healthz(w http.ResponseWriter, _ *http.Request) {
	w.WriteHeader(http.StatusOK)
	w.Write([]byte("ok"))
}
```

# /readyz: Can This Pod Serve Traffic?

`/readyz` is where the real work happens. It checks every dependency the service needs to handle a request. If any of them fails, it returns `503` and Kubernetes keeps the pod out of the Service.

```go
// Check is one dependency the service needs before it can serve traffic.
type Check struct {
	Name string
	Fn   func(ctx context.Context) error
}

// Readyz answers "can this pod serve a request right now?".
func Readyz(checks []Check) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// Keep this below the probe's timeoutSeconds.
		ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
		defer cancel()

		failed := map[string]string{}
		for _, c := range checks {
			if err := c.Fn(ctx); err != nil {
				failed[c.Name] = err.Error()
			}
		}

		w.Header().Set("Content-Type", "application/json")
		if len(failed) > 0 {
			w.WriteHeader(http.StatusServiceUnavailable)
			json.NewEncoder(w).Encode(map[string]any{"status": "not ready", "failed": failed})
			return
		}
		json.NewEncoder(w).Encode(map[string]any{"status": "ready"})
	}
}
```

Each service registers the dependencies it actually uses:

```go
checks := []Check{
	{"postgres", func(ctx context.Context) error { return db.PingContext(ctx) }},
	{"redis", func(ctx context.Context) error { return rdb.Ping(ctx).Err() }},
	{"rabbitmq", func(ctx context.Context) error {
		if amqpConn.IsClosed() {
			return errors.New("connection closed")
		}
		return nil
	}},
	{"kafka", func(ctx context.Context) error { return kafkaClient.Ping(ctx) }},
}

mux := http.NewServeMux()
mux.HandleFunc("GET /healthz", Healthz)
mux.HandleFunc("GET /readyz", Readyz(checks))
```

Here `db` is a `*sql.DB`, `rdb` is a go-redis client, `amqpConn` is an `amqp091-go` connection and `kafkaClient` is a franz-go client. The idea is the same in any language: ping each dependency with a short timeout and report which ones failed.

Kubernetes only looks at the status code, so the JSON body with the failed dependencies isn't for Kubernetes. It's for us. When a pod refuses to go Ready, you `kubectl port-forward` to it, `curl localhost:8080/readyz`, and it tells you straight away which dependency is the problem. Saves a lot of guessing. (`kubectl exec` plus `curl` works too, but only if the image has `curl`, and slim or distroless images usually don't.)

Here's what it said in the demo, for a release I deployed with a wrong database password (more on releases like this in the Helm section):

```json
{
  "failed": {
    "postgres": "failed to connect to `user=orders database=orders`: 10.96.230.7:5432 (postgres): failed SASL auth: FATAL: password authentication failed for user \"orders\" (SQLSTATE 28P01)"
  },
  "status": "not ready"
}
```

No digging through logs: it's Postgres, and it's the password.

# Wiring the Probes into the Deployment

Now we tell Kubernetes about the two endpoints:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  # No replicas here: HPA owns the replica count (see "Things to Watch Out For").
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: order-service
          image: registry.example.com/order-service:1.4.0
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["sleep", "5"]
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
```

A few of these values matter more than they look:

- **`timeoutSeconds: 3` on readiness.** The default is only 1 second. `/readyz` makes several network calls, so with the default, one slightly slow Kafka ping fails the probe even though everything is fine. The 2-second timeout inside `/readyz` stays under this.
- **`failureThreshold: 3` on liveness.** That's already the default, but it's worth knowing why it matters: one slow response shouldn't get a pod restarted. With `periodSeconds: 10`, the process has to be stuck for about 30 seconds before Kubernetes gives up on it.

# RollingUpdate: maxSurge 1, maxUnavailable 0

When you deploy a new version, a Deployment with the `RollingUpdate` strategy doesn't replace all of its pods at once. It swaps them gradually, and two settings decide how: `maxSurge` (how many extra pods it may create on top of the desired count) and `maxUnavailable` (how many pods may be missing below it).

There's no one right value for these. They have to match how your Deployment runs. In our case, each backend service runs a single pod most of the time, and only gets more when HPA scales it up under load. With just one pod, the order of the swap is everything:

- **`maxUnavailable: 0`** so Kubernetes never kills the old pod before its replacement is Ready. With one pod, killing it first means zero pods, and that's downtime.
- **`maxSurge: 1`** so Kubernetes is allowed to spawn the new pod next to the old one. The two can't both be 0 (Kubernetes rejects that), because then the rollout would have no way to make progress.

With exactly one pod, the defaults (25% each) actually work out to the same thing, because `maxSurge` rounds up to 1 and `maxUnavailable` rounds down to 0. We still set both explicitly, because once HPA scales a service to 4 or more pods, the default `maxUnavailable` becomes 1 or more, and Kubernetes is allowed to take old pods away before their replacements are Ready.

```yaml
spec:
  # No replicas: HPA owns it. Most of the time that's 1 pod (minReplicas: 1).
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
```

In other words: start the new pod first, wait for it to be Ready, and only then remove the old one. A deploy looks like this:

![Rolling update timeline: the v1.1.2 pod stays Ready and serving traffic while the new v1.1.3 pod initializes and is not ready. Once the new pod is Ready, traffic routes to v1.1.3 and the old pod stops.](/images/zero-downtime-deployments-kubernetes/rolling-update-timeline.png)

Here v1.1.2 is the old version and v1.1.3 is the new one.

1. v1.1.2 is running and Ready.
2. We deploy v1.1.3. `maxSurge: 1` lets Kubernetes create a v1.1.3 pod next to v1.1.2. `maxUnavailable: 0` means v1.1.2 has to stay.
3. v1.1.3 starts connecting to the DB, Redis, RabbitMQ and Kafka. `/readyz` returns `503`, so v1.1.3 gets no traffic. Everything still goes to v1.1.2.
4. Once every dependency is connected, `/readyz` returns `200` and v1.1.3 is added to the Service, so requests start coming in.
5. Only now does Kubernetes terminate v1.1.2.

Here's what that looks like in the demo, deploying 1.1.3 over 1.1.1 with the load generator running:

![Load generator output during a rolling update with probes: every line shows err=0. Requests go to 1.1.1, then for two seconds both 1.1.1 and 1.1.3 answer, then everything goes to 1.1.3.](/images/zero-downtime-deployments-kubernetes/rolling-update-loadgen.png)

Not a single failed request. For about two seconds both versions answered at the same time (that's steps 4 and 5), and then everything moved to 1.1.3.

This is exactly where `/readyz` pays off. Without a readiness probe, step 3 doesn't exist: v1.1.3 counts as Ready the moment the container starts, Kubernetes kills v1.1.2 straight away, and traffic goes to a pod that can't handle it yet. That was our old deploy in a nutshell, and it's the wall of `HTTP 503`s in the first screenshot.

And what if v1.1.3 never becomes Ready? Maybe a bad config, a wrong secret, or a dependency it can't reach. Then the rollout just gets stuck with v1.1.2 still serving. The broken release never takes a single request. Even better, Helm can roll a broken release like this back automatically. More on that in the Helm section below.

# Shutting Down the Old Pod Gracefully

`/readyz` fixes the first half of our original problem: new pods only get traffic once they can handle it. But remember the other half, the old pod getting killed with requests still in flight. Probes don't help there. That's what the `preStop` hook and `terminationGracePeriodSeconds` in the Deployment are for.

When Kubernetes terminates a pod (step 5 above), two things happen at the same time:

1. The pod is removed from the Service's endpoints, and that change has to reach kube-proxy on every node and the ingress controller.
2. The kubelet starts shutting the container down: it runs the `preStop` hook, then sends `SIGTERM`.

Because these run in parallel, and the endpoint removal usually takes a moment longer, an app that exits as soon as it gets `SIGTERM` still has requests being routed to it, and those fail. The `preStop` sleep gives the rest of the cluster a few seconds to stop sending new requests before the app even hears about the shutdown.

After that, the app has to shut down properly by itself: stop accepting new connections, finish the requests it's already handling, and then exit. In Go that's `http.Server.Shutdown`:

```go
srv := &http.Server{Addr: ":8080", Handler: mux}
go func() {
	if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
		log.Fatal(err)
	}
}()

// Wait for SIGTERM, which the kubelet sends after the preStop hook finishes.
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, os.Interrupt)
defer stop()
<-ctx.Done()

// Finish in-flight requests, but stay inside terminationGracePeriodSeconds.
shutdownCtx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
defer cancel()
srv.Shutdown(shutdownCtx)
```

`terminationGracePeriodSeconds: 30` covers the whole thing: 5 seconds of `preStop` sleep plus up to 20 seconds of draining, with some room to spare. If the container is still running after 30 seconds, it gets `SIGKILL`.

The demo shows both sides. Without any of this, the old pod in the first screenshot exited the moment it got `SIGTERM`, and that's where its `conn closed`, `conn reset`, `conn refused` and `timeout` errors came from. With the `preStop` sleep and graceful shutdown in place, here's the whole life of the old 1.1.1 pod from the rolling update above:

```
2026/10/05 03:55:14 order-service 1.1.1 starting on order-service-5dbbbf7cb6-g7xvm
2026/10/05 03:55:14 listening on :8080
2026/10/05 03:55:14 startup: warming up for 8s before connecting (STARTUP_DELAY)
2026/10/05 03:55:22 startup: connecting to postgres and redis
2026/10/05 03:55:24 readyz: all dependencies OK, pod can take traffic
2026/10/05 03:56:06 shutdown: SIGTERM received, draining in-flight requests (up to 20s)
2026/10/05 03:56:06 shutdown: done
```

The HTTP server was up at 03:55:14, but the pod only became Ready 10 seconds later, once it had warmed up and connected to Postgres and Redis. On the way out, traffic started moving to 1.1.3 around 03:56:01, and `SIGTERM` arrived at 03:56:06, after the 5-second `preStop` sleep. Look back at the rolling update screenshot: 1.1.1 was still getting requests for about a second after its shutdown began (72 of them in the 03:56:02 line). That's exactly the delay the sleep is there to cover. By the time `SIGTERM` arrived, nothing was being sent to it anymore, so the drain finished in the same second.

Watch out for this one in Node.js and Python: if the app runs as PID 1 in the container and doesn't register a `SIGTERM` handler, it ignores `SIGTERM` completely. It just sits there until the grace period runs out and gets killed, in-flight requests and all. Handle `SIGTERM` in code (`server.close()` in Express) or use a server that does it for you (gunicorn and uvicorn both shut down gracefully on `SIGTERM`), and start the app with `node server.js` rather than `npm start`, so the signal actually reaches your process.

# Deploying with Helm

Everything so far is plain Kubernetes, and you could deploy it all with `kubectl apply`. It works, but you don't get much else:

- **No real versioning.** A Deployment does keep a rollout history, but it only gets a new revision when the pod template changes, and `kubectl rollout undo` only rolls back that one Deployment. Your Service and ConfigMaps aren't part of it.
- **No record of what was deployed.** The rollout history's `CHANGE-CAUSE` column is empty unless someone remembers to add an annotation every time. Good luck finding out what was running yesterday at 3pm.

Helm fixes both. It treats everything a service needs (Deployment, Service, ConfigMaps) as one **release**, and every deploy becomes a numbered **revision** of that release. The revisions are stored as Secrets in the release's namespace, so the history lives in the cluster itself.

## The deploy script

This is roughly what our deploy script ran for each service:

```bash
#!/usr/bin/env bash
set -euo pipefail

SERVICE="$1"   # e.g. order-service
VERSION="$2"   # image tag, e.g. 1.4.0
NS="backend"

helm upgrade --install "$SERVICE" "./charts/$SERVICE" \
  --namespace "$NS" \
  --set image.tag="$VERSION" \
  --atomic \
  --wait \
  --timeout 10m \
  --history-max 20 \
  --description "deploy $VERSION"

helm history "$SERVICE" --namespace "$NS" --max 5
```

Going through the flags:

- **`--install`**: if the release doesn't exist yet, install it. Same command for the first deploy and every one after.
- **`--wait`**: don't mark the release as successful until the Deployment has enough Ready pods. And Ready means `/readyz` passed, so Helm only calls a release successful once the new pods can actually reach their DB, Redis, RabbitMQ and Kafka.
- **`--atomic`**: if the upgrade fails, for example because the new pods never became Ready within `--timeout`, Helm automatically rolls back to the last successful release. `--atomic` actually turns on `--wait` by itself, we just kept it in the script to be explicit.
- **`--timeout 10m`**: how long `--wait` waits before calling it a failure. The default is 5 minutes.
- **`--history-max 20`**: how many revisions Helm keeps per release. The default is 10.
- **`--description`**: shows up in `helm history`, so every revision says which version it deployed.

> If you're on Helm 4, `--atomic` has been renamed to `--rollback-on-failure` (`helm upgrade` still accepts `--atomic` for now, with a deprecation warning), and `--wait` now takes a strategy (`--wait` on its own uses `watcher`). The idea is the same. The demo in this post runs Helm 4, so that's the name you'll see in its output.

Here's the part I like the most. Remember the stuck rollout from before, where v1.1.3 never gets Ready and v1.1.2 keeps serving? With `--atomic`, that's no longer something someone has to notice and clean up at 11pm. Helm waits, times out, and rolls back on its own. And because of `maxUnavailable: 0`, the old pods were never removed in the first place, so users didn't see anything during the whole failed deploy.

In the demo I deployed 1.4.0 with a wrong database password. The new pod started, never passed `/readyz`, and just sat at `0/1`. Liveness only checks `/healthz`, so it was never restarted either, and the old pod kept serving the whole time:

![kubectl get pods during the broken deploy: the new order-service pod is 0/1 Running with 0 restarts after 100 seconds, while the old pod is 1/1 Running.](/images/zero-downtime-deployments-kubernetes/broken-release-stuck.png)

The demo uses a 2-minute timeout instead of 10, so two minutes in, Helm gave up, rolled back, and the broken pod went away:

![kubectl get pods right after the rollback: the new pod is Terminating at 2m1s old, and the old pod is still 1/1 Running.](/images/zero-downtime-deployments-kubernetes/broken-release-rolled-back.png)

In the pipeline, that shows up as a failed step:

```
Error: UPGRADE FAILED: release order-service failed, and has been rolled back due to rollback-on-failure being set: resource Deployment/backend/order-service not ready. status: InProgress, message: Pending termination: 1
context deadline exceeded
```

`Pending termination: 1` is Helm saying the Deployment still has one extra pod, the old one, waiting to be removed. That never happens, because the new pod never gets Ready.

`maxUnavailable: 0` also matters for `--wait` itself. In Helm 3, `--wait` only waits until the new ReplicaSet has at least (desired − `maxUnavailable`) Ready pods. The Helm docs actually call out that with 1 replica and `maxUnavailable` not set to 0, `--wait` returns as ready without really waiting for the new pod. With `maxUnavailable: 0`, Helm waits until every new pod passes `/readyz`. Helm 4's `watcher` strategy waits for every replica to be updated and Ready anyway, so there this caveat only applies if you pick `--wait=legacy`.

## helm history

Here's `helm history` from the demo after that failed deploy, plus one more deploy with the fix:

```
$ helm history order-service -n backend --max 5
REVISION  UPDATED                   STATUS      CHART                APP VERSION  DESCRIPTION
4         Mon Oct  5 11:55:13 2026  superseded  order-service-0.3.0  1.0.0        deploy 1.1.1
5         Mon Oct  5 11:55:50 2026  superseded  order-service-0.3.0  1.0.0        deploy 1.1.3
6         Mon Oct  5 11:56:55 2026  failed      order-service-0.3.0  1.0.0        Upgrade "order-service" failed: resource Deployment/backend/order-service not ready. status: InProgress, message: Pending termination...
7         Mon Oct  5 11:58:56 2026  superseded  order-service-0.3.0  1.0.0        Rollback to 5
8         Mon Oct  5 12:05:19 2026  deployed    order-service-0.3.0  1.0.0        deploy 1.4.1
```

(These times are my local time, UTC+8. The pod logs and load generator output earlier are in UTC.)

You can read most of the story straight from it: revision 6 never became Ready, Helm rolled back to revision 5 (that's revision 7), and the fixed release `1.4.1` went out in revision 8. One catch: when an upgrade fails, Helm replaces your `--description` with the error message, so revision 6 doesn't say which version it tried. `helm get values` does:

```
$ helm get values order-service -n backend --revision 6
USER-SUPPLIED VALUES:
image:
  tag: 1.4.0
postgres:
  password: wrong
```

The broken version, and the reason it was broken, in one command.

Also notice `APP VERSION` stays at `1.0.0`. It comes from the chart's `Chart.yaml`, not from the image tag we pass with `--set`, which is exactly why the deploy script sets `--description`.

A few commands we used all the time:

```bash
# What values were used in a specific revision?
helm get values order-service -n backend --revision 5

# Roll back by hand to a known good revision
helm rollback order-service 5 -n backend
```

A rollback is itself a new revision, so even rollbacks show up in the history. Nothing disappears.

## Helm Capabilities

Helm charts are templates, and Helm gives the templates a built-in `.Capabilities` object that describes the cluster you're deploying to: which API versions it supports (`.Capabilities.APIVersions.Has`) and which Kubernetes version it's running (`.Capabilities.KubeVersion`). That lets one chart render the right manifests for each cluster.

A good example is the `preStop` sleep from earlier. Kubernetes has a built-in `sleep` action for `preStop` hooks, turned on by default since 1.30 and stable since 1.34. It's nicer than `exec: sleep` because it doesn't need a `sleep` binary inside the image, which minimal images like distroless don't have. But an older cluster rejects it. If your clusters aren't all on the same version, the chart can pick the right one:

```yaml
          lifecycle:
            preStop:
              {{- if semverCompare ">=1.30-0" .Capabilities.KubeVersion.Version }}
              sleep:
                seconds: 5
              {{- else }}
              exec:
                command: ["sleep", "5"]
              {{- end }}
```

Same chart, same deploy script, and each cluster gets a `preStop` hook it actually understands.

# Making It Stick: Scripts, Pipeline and a Runbook

Getting one service right is the easy part. Getting every service, and everyone who deploys them, to do it the same way is the real work. That's where most of my effort went:

- **Deployment scripts.** The Helm script above became the standard way to deploy a service, and our CI/CD pipeline runs it. Every release goes through the same steps with the same flags, so `--atomic` and `--wait` protect every deploy, not just the ones where someone remembered them.
- **An operational runbook.** A written guide on how to deploy and, more importantly, what to do when something goes wrong. Nobody wants to figure out how to roll back for the first time while production is on fire.

Here's a condensed version of the "when things go wrong" part of the runbook:

| What you see | What it means | What to do |
|---|---|---|
| Pipeline failed, `helm history` shows `failed` and then `Rollback to N` | The new pods never got Ready, and `--atomic` rolled back. Users didn't notice anything. | Find out why the pods weren't Ready (see below), fix it, deploy again. |
| `Error: another operation (install/upgrade/rollback) is in progress` | Helm got stuck in the middle, usually because the pipeline job was cancelled mid-deploy. The release is left in `pending-upgrade`. | Helm refuses new upgrades while a release is pending, but a rollback still works. Roll back to the last good revision. If it's stuck in `pending-install` (the very first deploy was cancelled), there's nothing to roll back to, so `helm uninstall` it and deploy again. |
| Deploy succeeded, but the new version has a bug | Nothing wrong with the deploy itself, the code is the problem. | Roll back to the previous good revision, then fix forward. |

To find out why pods never got Ready, `/readyz` from earlier does most of the work:

```bash
kubectl -n backend get pods -l app=order-service
kubectl -n backend describe pod <pod-name>    # events: image pull errors, probe failures
kubectl -n backend logs <pod-name>
kubectl -n backend port-forward pod/<pod-name> 8080:8080 &   # works even if the image has no curl
curl -s localhost:8080/readyz                                # which dependency is failing?
```

And rolling back always follows the same three steps:

```bash
# 1. What state is the release in, and which revision was last good?
helm history order-service -n backend --max 5

# 2. Roll back to that revision, explicitly
helm rollback order-service <last-good-revision> -n backend --wait --timeout 10m

# 3. Confirm it's healthy
helm status order-service -n backend
kubectl -n backend rollout status deployment/order-service
```

Why pass the revision number explicitly? `helm rollback order-service` with no number rolls back to the current revision minus one. After an `--atomic` rollback, that "previous" revision is the failed one, and you'd be redeploying the broken release. In the history above, running it right after revision 7 would have taken you straight back to revision 6, the broken 1.4.0. So the runbook always says: check `helm history` first, and roll back to a revision you know was good.

# Things to Watch Out For

**The old and new versions serve traffic at the same time.** During a rolling update there's a window where both the old and the new version are running and taking requests. You can see it in the rolling update screenshot, where 1.1.1 and 1.1.3 both answered for about two seconds. So every release has to work next to the previous one: an API change can't break callers that still talk to the old version, and a new Kafka or RabbitMQ message format can't break consumers that are still on the old version. Major database migrations were a separate process from backend deployments for us, so they're not part of this flow at all.

**A shared dependency outage makes every pod unready.** If the database goes down, every pod fails `/readyz`, the Service has no endpoints, and callers get errors. Honestly, that's the correct outcome, since the pods couldn't have served those requests anyway. The upside is that nothing restarts, and traffic comes back on its own once the database is back. Still, only put hard dependencies in `/readyz`. If a service only uses Kafka to publish events in the background, it might be better to keep serving and retry the events than to pull the whole service out because Kafka is having a slow day.

**Keep `/readyz` cheap.** Kubernetes calls it on every pod, every `periodSeconds`, forever. A ping is fine. A real query against a big table is not.

**Don't set `replicas` when HPA manages the Deployment.** It's tempting to put `replicas: 1` in the manifest, since that's what we usually run. But then every `helm upgrade` sets the Deployment back to 1, even if HPA had scaled it to 4 because of load, and you lose capacity right in the middle of a deploy. Leave `replicas` out and let HPA's `minReplicas` handle the baseline. If the same chart is used with and without HPA, wrap it in `{{- if not .Values.autoscaling.enabled }}`, which is what `helm create` generates anyway.

**Slow-starting services need a `startupProbe`.** If a service takes a minute to start, a liveness probe with a short `initialDelaySeconds` restarts it before it ever finishes starting, over and over. A `startupProbe` holds off the liveness and readiness probes until the app has started successfully once.

# Results

Here's the before and after from the demo. Same app, same load, and only the deploy setup changed:

| | Before (no readiness probe, no graceful shutdown) | After |
|---|---|---|
| Failed requests | 760 of 902 in a 9-second window (84%) | 0 |
| Seconds where no request succeeded | 7 | 0 |

On top of that, the broken release never got a single request: the old version kept serving until Helm rolled it back. It's one laptop and about 150 requests per second, so treat the exact numbers as an illustration, not a benchmark. The shape is what matters: before, a deploy meant several seconds of errors. After, a deploy is a non-event, even a broken one.

Looking back, the big change wasn't really a Kubernetes feature. Kubernetes had probes and RollingUpdate all along. What made it work was the backend finally telling Kubernetes the truth: `/healthz` for "I'm alive", `/readyz` for "I can take traffic now", and a clean shutdown when it's asked to leave. Helm then used that same signal to decide whether a release succeeded, and rolled it back when it didn't. Once all of that was in place, deploys stopped being something we had to schedule around.
