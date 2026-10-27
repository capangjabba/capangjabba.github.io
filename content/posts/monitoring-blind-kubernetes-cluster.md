+++
title = 'Inheriting a Blind Kubernetes Cluster: Audit First, Then Exporters'
description = "How to go from metrics nobody looks at, with customer complaints as the only alert, to catching problems before a customer notices: audit first, reuse kube-state-metrics, add two exporters through pod annotations, and break things on purpose to prove the alerts fire."
date = 2026-10-26T12:00:00+08:00
draft = false
categories = ["DevOps"]
series = ["Observability"]
tags = ["kubernetes", "prometheus", "grafana", "redis", "bullmq", "mqtt", "alerting"]
+++

> **Disclaimer:** The story comes from a cluster I work with, but nothing from it is shown here. All the code, manifests, numbers and screenshots come from a small lab I rebuilt on my laptop with [kind](https://kind.sigs.k8s.io/), with made-up apps (an order queue and some sensors). Please adjust things before using them anywhere real. AI helped me write this blog post.

Kubernetes is useful: if a pod breaks, it replaces it. But on the other side, what if nothing breaks? What if a queue backs up, or a process takes too long to finish? Who finds out? A customer, when they complain. The cluster "looked" monitored. Prometheus was running, and Grafana had a few pretty dashboards. But when I actually checked, almost nothing could alert anyone, and almost nothing on those dashboards meant anything. CPU usage? Memory usage? Network bandwidth? These don't tell us what the underlying problem is. How about predicting or detecting? Hurmmmm.

This post is about what I did to get to alerts that fire before a customer notices, on a shared cluster, without owning the monitoring stack and without breaking much of the production code. The steps are:

1. **Audit first.** Find out what we can use, what is scraped, and what can actually be useful.
2. **Use what's already there.** `kube-state-metrics` and `cAdvisor` already know about crashloops, restarts and memory.
3. **Get scraped through Prometheus discovery and pod annotations**, so the platform team's Prometheus config stays untouched, with no risk of breaking it.
4. **Add exporters for what's missing.** A stock exporter where one exists, your own only when you have to.
5. **Drill every alert.** Break things on purpose and time how long it takes for the email to arrive.

Here's what the lab looks like at the end. Gray was already there, green is what this post adds:

![Architecture of the lab: in the monitoring namespace, Prometheus scrapes kube-state-metrics, Grafana queries Prometheus and sends alert emails. In the shop namespace, redis_exporter reads the BullMQ keys in Redis and our mqtt-exporter reads $SYS topics and sends a canary through Mosquitto. Prometheus scrapes both exporters because their pods are annotated prometheus.io/scrape.](/images/monitoring-blind-kubernetes-cluster/architecture.svg)

# Background

The real cluster runs a few dozen Node.js microservices. Some of them use Redis-backed job queues ([BullMQ](https://docs.bullmq.io/)), others use an MQTT broker ([Mosquitto](https://mosquitto.org/)). The platform team runs Prometheus, while Grafana is shared by everyone. I don't own their Prometheus config, and I shouldn't need to.

The lab copies that shape on a single-node kind cluster:

- **`monitoring` namespace**, the platform's: the stock [`prometheus-community/prometheus`](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus) Helm chart (with kube-state-metrics), Grafana, and [Mailpit](https://mailpit.axllent.org/) to catch alert emails.
- **`shop` namespace**, the apps: Redis with a BullMQ `orders` queue (`order-api` adds 20 jobs per second, `order-worker` processes them), and Mosquitto with five sensors publishing a reading every second.

On purpose, the apps look like the ones I inherited: no `prometheus.io` annotations, no probes, no resource limits.

# Step 1: Audit Before You Add Anything

Before adding a single exporter, it's better to ask these questions:

1. **What can generate alerts, and where do alerts go?** Rules mean nothing if they don't reach a person.
2. **What is scraped?** Which pods does Prometheus actually collect from?
3. **What is already collected** that nobody uses?

All three can be answered read-only, from the Prometheus and Grafana HTTP APIs plus `kubectl get`. This is the core of the audit script:

```bash
PROM=http://localhost:30090
GRAFANA=http://<user>:<pass>@localhost:30030

# 1. What can alert, and where would it go?
curl -s "$PROM/api/v1/rules"         | jq '[.data.groups[].rules[]] | length'
curl -s "$PROM/api/v1/alertmanagers" | jq '.data.activeAlertmanagers | length'
curl -s "$GRAFANA/api/v1/provisioning/alert-rules" | jq 'length'

# 2. What is scraped?
curl -s "$PROM/api/v1/targets" | jq -r '.data.activeTargets | group_by(.labels.job)[]
  | "\(.[0].labels.job)\t\(length) target(s)"'
```

And this is what it said about the lab, which reflects what I found at work:

```
== Prometheus: rules loaded
0
== Prometheus: Alertmanagers it sends to
0
== Grafana: alert rules
0
== Prometheus: scrape targets per job
kubernetes-api-servers        1 target(s)
kubernetes-nodes              1 target(s)
kubernetes-nodes-cadvisor     1 target(s)
kubernetes-service-endpoints  4 target(s)
prometheus                    1 target(s)
== Workload pods: scraped? probes? memory limit?
NAMESPACE   POD                 SCRAPED  READINESS  MEM_LIMIT
monitoring  grafana             -        yes        -
monitoring  mailpit             -        -          -
monitoring  kube-state-metrics  -        yes        -
monitoring  prometheus          -        yes        -
shop        mosquitto           -        -          -
shop        order-api           -        -          -
shop        order-worker        -        -          -
shop        redis               -        -          -
shop        sensors             -        -          -
== Already collected for the shop namespace (no exporter needed)
kube_pod_container_status_restarts_total      8 series
kube_pod_container_status_waiting_reason      0 series
kube_deployment_status_replicas_available     5 series
container_memory_working_set_bytes            23 series
```

Reading it top to bottom:

- **Zero rules and zero Alertmanagers.** Nothing in this cluster can tell a person that something is wrong. Everything else is secondary to this.
- **No app is scraped.** Prometheus is healthy, but it only collects Kubernetes' own metrics. Nobody knows how deep the queue is or whether the broker is delivering.
- **But a lot is already collected.** `kube-state-metrics` and `cAdvisor` already track restarts, waiting reasons, replica counts and memory for every pod in `shop`. That data was just sitting there.

So the gap wasn't "we have no metrics". It was "we have metrics, but we're not using them, and nobody gets told". That changes the order of work: alerting first, using what's already collected, then exporters for what's needed.

# Step 2: Free Wins from kube-state-metrics

Before adding anything new, here's what's already in Prometheus:

| I want to know when... | Metric (already collected) |
|---|---|
| a pod is crashlooping | `kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}` |
| a container keeps restarting | `kube_pod_container_status_restarts_total` |
| why it last died (OOMKilled, Error) | `kube_pod_container_status_last_terminated_reason` |
| a Deployment has fewer pods than it wants | `kube_deployment_spec_replicas - kube_deployment_status_replicas_available` |
| a pod's memory is creeping up | `container_memory_working_set_bytes` (cAdvisor) |

Alerting here uses Grafana's built-in alerting. Grafana can provision alert rules from a file, so the rules live in a ConfigMap next to everything else instead of being clicked together in the UI. Rules made in the UI live only in Grafana's database, while a file can be versioned, reviewed and recreated at any time. Here's the crashloop rule:

```yaml
- uid: pod-crashlooping
  title: pod crashlooping
  # No CrashLoopBackOff series means nothing is crashlooping: that's OK, not "no data".
  noDataState: OK
  condition: C
  data:
    - refId: A
      relativeTimeRange: { from: 300, to: 0 }
      datasourceUid: prometheus
      model:
        refId: A
        instant: true
        # Each CrashLoopBackOff wait only lasts 10s, 20s, 40s... between restarts,
        # so look back 5m instead of at this instant (kube-prometheus does the same).
        expr: max by (namespace, pod, container) (max_over_time(kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}[5m]))
    - refId: C
      datasourceUid: __expr__
      model:
        refId: C
        type: threshold
        expression: A
        conditions:
          - evaluator: { type: gt, params: [0] }
  for: 1m
  labels: { severity: warning }
  annotations:
    summary: '{{ $labels.namespace }}/{{ $labels.pod }} ({{ $labels.container }}) is in CrashLoopBackOff.'
```

Watch out for that `noDataState: OK` line. kube-state-metrics only exposes a `waiting_reason` series while a container is actually waiting. When everything is healthy, the query returns nothing at all. Grafana's default for an empty result is the `NoData` state, which notifies just like a firing alert. My first version of the rule went straight to this, with every pod healthy:

```
pod crashlooping Pending (NoData)
```

For this rule, an empty result means healthy, so `noDataState: OK` is correct. Think about it per rule, though. For the queue backlog rule further down, "no data" means the exporter is gone, which is a different problem with its own alert.

However, this rule didn't survive the drills, where I simulated what might happen and checked whether the alerts fired. The `max_over_time(...[5m])` was already my first fix, and in Step 7 it still fired almost 6 minutes late, so I replaced it with a rule that counts restarts. The `noDataState` lesson still holds for any rule built on a series that only exists while something is wrong.

# Step 3: Telling Prometheus to Scrape Me

The platform team owns the Prometheus config, so I couldn't add scrape jobs. But the `prometheus-community/prometheus` chart (and many hand-written configs that copied it) ships a `kubernetes-pods` job that scrapes **any pod that asks for it** through annotations. Trimmed down, it looks like this:

```yaml
- job_name: kubernetes-pods
  kubernetes_sd_configs:
    - role: pod
  relabel_configs:
    # only pods annotated prometheus.io/scrape: "true"
    - action: keep
      regex: true
      source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    # prometheus.io/path overrides /metrics
    - action: replace
      regex: (.+)
      source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      target_label: __metrics_path__
    # prometheus.io/port picks the port
    - action: replace
      regex: (\d+);((([0-9]+?)(\.|$)){4})
      replacement: $2:$1
      source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port, __meta_kubernetes_pod_ip]
      target_label: __address__
    # every pod label becomes a metric label, plus namespace and pod
    - action: labelmap
      regex: __meta_kubernetes_pod_label_(.+)
    - action: replace
      source_labels: [__meta_kubernetes_namespace]
      target_label: namespace
    - action: replace
      source_labels: [__meta_kubernetes_pod_name]
      target_label: pod
```

So all a new exporter needs is two annotations on its pod template:

```yaml
template:
  metadata:
    labels:
      app: redis-exporter
    annotations:
      prometheus.io/scrape: "true"
      prometheus.io/port: "9121"
```

The `labelmap` rule turns the pod's `app` label into a metric label, so alert rules can say `up{app="redis-exporter"}` instead of matching pod names.

> This works for my cluster, but if your cluster runs the `Prometheus Operator` (kube-prometheus-stack), annotations do nothing by default. There you need to add a `PodMonitor` or `ServiceMonitor` instead, which is also something you can do without touching the Prometheus config itself. Kubernetes is full of "adding another object".

Once the exporters from the next two steps were deployed, they showed up in the `kubernetes-pods` pool by themselves, with nothing changed on the Prometheus side:

![Prometheus target health page filtered to the kubernetes-pods pool: 2 of 2 up. redis-exporter on port 9121 and mqtt-exporter on port 9883, both with labels app, namespace=shop, pod and node.](/images/monitoring-blind-kubernetes-cluster/targets-after.png)

# Step 4: Redis and BullMQ with the Stock redis_exporter

We need to monitor the queue's state. A worker that stops consuming doesn't crash, doesn't restart, and doesn't show up anywhere in `kube-state-metrics`. The only sign is jobs piling up. When we don't monitor the queue, the customer comes to us, and we wouldn't want that :p

My first thought was to write a BullMQ exporter. Then I looked at what BullMQ actually stores in Redis. For a queue called `orders`:

| Key | Redis type | What it is |
|---|---|---|
| `bull:orders:wait` | list | jobs waiting for a worker |
| `bull:orders:active` | list | jobs being processed right now |
| `bull:orders:delayed`, `bull:orders:prioritized` | sorted set | delayed and prioritized jobs |
| `bull:orders:completed`, `bull:orders:failed` | sorted set | finished jobs (until cleaned up) |
| `bull:orders:id` | string | a counter, `INCR`ed on every job added |
| `bull:orders:meta` | hash | queue settings, one per queue |

Queue depth is just the length of a few keys. And the stock [redis_exporter](https://github.com/oliver006/redis_exporter) can already export the length (and value) of keys you name. No custom exporter needed, only config:

```yaml
env:
  - name: REDIS_ADDR
    value: redis://redis.shop.svc:6379
  - name: REDIS_USER
    value: exporter
  - name: REDIS_PASSWORD
    valueFrom:
      secretKeyRef: { name: redis-exporter, key: REDIS_PASSWORD }
  # Exact keys, looked up directly: no SCAN, so the cost doesn't
  # grow with the size of the keyspace.
  - name: REDIS_EXPORTER_CHECK_SINGLE_KEYS
    value: bull:orders:wait,bull:orders:active,bull:orders:delayed,bull:orders:prioritized,bull:orders:completed,bull:orders:failed,bull:orders:id
```

Which gives you this on `/metrics`, next to the usual Redis health metrics (`redis_up`, memory, clients):

```
redis_key_size{db="db0",key="bull:orders:active"} 0
redis_key_size{db="db0",key="bull:orders:completed"} 640
redis_key_size{db="db0",key="bull:orders:delayed"} 0
redis_key_size{db="db0",key="bull:orders:failed"} 0
redis_key_size{db="db0",key="bull:orders:id"} 3
redis_key_size{db="db0",key="bull:orders:prioritized"} 0
redis_key_size{db="db0",key="bull:orders:wait"} 0
redis_key_value{db="db0",key="bull:orders:id"} 640
```

Two things worth knowing here:

- **Keys that don't exist report as `0`.** BullMQ deletes `wait` when it's empty, and that's exactly what "nothing waiting" should look like.
- **The `:id` counter gives you the intake rate for free.** It goes up by one for every job added, so `rate(redis_key_value{key="bull:orders:id"}[1m])` is jobs added per second. Use `redis_key_value` for it, not `redis_key_size`: for a string, the "size" is its length, which is why `bull:orders:id` shows `3` above (the string `"640"`).

## The read-only Redis user that couldn't read

We don't want to forget about security: the exporter absolutely shouldn't use the apps' Redis password, so I gave it its own ACL user, copying the `ACL SETUSER` line from the redis_exporter README. Every key check failed, even for keys that clearly existed:

```
level=error msg="getKeyInfo() err: unknown type:  for key: bull:orders:failed"
level=error msg="getKeyInfo() err: unknown type:  for key: bull:orders:id"
```

Trying the same thing as that user with `redis-cli` made it obvious:

```
$ redis-cli --user exporter --pass ... TYPE bull:orders:completed
NOPERM No permissions to access a key
```

The README line allows the right **commands** (`+type`, `+llen`, `+get`...), but it gives the user no **keys** at all, and a Redis ACL user starts with none. `ACL LOG` even says so (`reason: key`). The fix is to add read-only access to just the queue keys at the end of the line:

```
user exporter on >*** -@all +@connection +memory ... +info +latency +scan -reset -auth -asking %R~bull:*
```

`%R~bull:*` means "read, and only keys starting with `bull:`". The exporter can see the queues and nothing else.

## Why not just discover the queues automatically?

redis_exporter can also take patterns: `REDIS_EXPORTER_CHECK_KEYS=bull:*:meta` would find every queue by itself, so new queues show up without a config change. Tempting. But a pattern means a `SCAN` through the **entire keyspace** on every scrape, and BullMQ creates one key per job. A backlog is exactly when the keyspace explodes, and exactly when you need monitoring to be cheap.

So I measured it. The lab's script `scan-cost.sh` stops the workers, piles up jobs, runs a second exporter with discovery on, and after one scrape reads what `SCAN` cost Redis from `INFO commandstats`:

```bash
redis config resetstat
kubectl -n shop exec deploy/redis-exporter-discovery -- wget -qO- localhost:9121/metrics >/dev/null
redis info commandstats | grep '^cmdstat_scan:'
# cmdstat_scan:calls=<SCAN calls>,usec=<total µs inside SCAN>,usec_per_call=...
```

| Keys in Redis | `SCAN` calls per scrape | Redis time in `SCAN` | Scrape time, discovery on | Scrape time, single keys |
|---|---|---|---|---|
| 10,017 | 11 | < 1 ms | 0.012 s | 0.011 s |
| 99,188 | 100 | 7 ms | 0.015 s | 0.012 s |
| 297,443 | 298 | 54 ms | 0.111 s | 0.013 s |
| 992,389 | 992 | 268 ms | 0.283 s | 0.009 s |

A quarter of a second at a million keys isn't a disaster on my laptop. The shape is what matters. The single-key exporter stays flat at about 10 ms however big the backlog gets. Discovery grows with the keyspace, and in my lab each key even got more expensive as it grew (0.06 µs per key at 10k keys, 0.27 µs at 1M). Redis runs commands on a single thread, so that's Redis time taken away from your workers on every scrape, and it peaks exactly when the queue is already in trouble. A busy production Redis will give you different numbers, so measure yours the same way before turning discovery on.

So I kept the explicit key list, and the price is a config change whenever someone adds a queue, which doesn't happen often. If you want both, there's a middle ground I haven't needed yet: a second redis_exporter that only does discovery, annotated `prometheus.io/scrape_slow: "true"`. The same Prometheus chart has a `kubernetes-pods-slow` job that scrapes those pods every 5 minutes instead of every 15 seconds.

# Step 5: When Stock Isn't Enough: A Small MQTT Exporter

For Mosquitto, I looked for an existing exporter first too. The first one that came up when I googled was [sapcc/mosquitto-exporter](https://github.com/sapcc/mosquitto-exporter), but its last release (v0.8.0) was in October 2021. More importantly, when I dug deeper, it didn't answer the questions I wanted answered. Mosquitto's `$SYS` topics tell you the broker is **alive** (clients, message rates, uptime), not that a message published by one client **gets delivered** to another.

So I wrote a new one: subscribe to a few `$SYS` topics, and every 15 seconds publish a canary message to a topic the exporter is also subscribed to, then time the round trip. If it doesn't come back within 5 seconds, `mqtt_canary_success` drops to 0.

```js
const SYS_TOPICS = {
  '$SYS/broker/clients/connected': ['mosquitto_clients_connected', 'Connected clients'],
  '$SYS/broker/messages/stored': ['mosquitto_messages_stored', 'Messages held by the broker (retained + queued)'],
  '$SYS/broker/load/messages/received/1min': ['mosquitto_messages_received_per_min', 'Messages received, 1 minute average'],
  '$SYS/broker/load/messages/sent/1min': ['mosquitto_messages_sent_per_min', 'Messages sent, 1 minute average'],
  '$SYS/broker/uptime': ['mosquitto_uptime_seconds', 'Broker uptime'],
};
const sys = Object.fromEntries(
  Object.entries(SYS_TOPICS).map(([topic, [name, help]]) => [topic, new client.Gauge({ name, help })]),
);
const up = new client.Gauge({ name: 'mosquitto_up', help: '1 if the exporter is connected to the broker' });
const canaryOk = new client.Gauge({ name: 'mqtt_canary_success', help: '1 if the last canary came back in time' });
const canaryRtt = new client.Gauge({ name: 'mqtt_canary_rtt_seconds', help: 'Round trip of the last canary' });

const mq = mqtt.connect(BROKER_URL, { clientId: `mqtt-exporter-${process.pid}`, reconnectPeriod: 2_000 });
mq.on('connect', () => {
  up.set(1);
  mq.subscribe(['$SYS/broker/#', CANARY_TOPIC]);
});
mq.on('close', () => up.set(0));

let pending = null; // { sentAt, timer } for the canary in flight
mq.on('message', (topic, payload) => {
  if (topic === CANARY_TOPIC) {
    if (!pending || payload.toString() !== String(pending.sentAt)) return;
    clearTimeout(pending.timer);
    canaryRtt.set((Date.now() - pending.sentAt) / 1000);
    canaryOk.set(1);
    pending = null;
    return;
  }
  // $SYS values are plain numbers, except uptime ("1234 seconds").
  sys[topic]?.set(parseFloat(payload.toString()));
});

setInterval(() => {
  if (pending) return; // the previous canary is still waiting for its timeout
  const sentAt = Date.now();
  const timer = setTimeout(() => {
    canaryOk.set(0);
    pending = null;
  }, CANARY_TIMEOUT_MS);
  pending = { sentAt, timer };
  mq.publish(CANARY_TOPIC, String(sentAt), { qos: 1 });
}, CANARY_EVERY_MS);
```

Plus a few lines of `node:http` to serve `/metrics` with [prom-client](https://github.com/siimp/prom-client). That's the whole exporter, about 60 lines. Its output:

```
mosquitto_clients_connected 2
mosquitto_messages_stored 51
mosquitto_messages_received_per_min 795.34
mosquitto_messages_sent_per_min 580.67
mosquitto_uptime_seconds 132
mosquitto_up 1
mqtt_canary_success 1
mqtt_canary_rtt_seconds 0.005
```

It gets the same two annotations as redis_exporter, and Prometheus picks it up by itself.

My rule of thumb from this: **a custom exporter has to earn its place.** Every one you write is code someone has to patch, upgrade and understand at 3am. redis_exporter was a config change. The MQTT exporter is the one thing that couldn't come from config, because the canary is behavior, not a metric someone already exposes.

# Step 6: Alert Rules, an Inbox, and a Dashboard

Everything Grafana needs for alerting lives in one provisioning file, mounted from a ConfigMap: where emails go, how they're grouped, and the rules themselves.

```yaml
apiVersion: 1

contactPoints:
  - orgId: 1
    name: oncall-email
    receivers:
      - uid: oncall-email
        type: email
        settings:
          addresses: oncall@shop.lab
          singleEmail: true

policies:
  - orgId: 1
    receiver: oncall-email
    group_by: [alertname]
    group_wait: 10s
    group_interval: 30s
    repeat_interval: 4h

groups:
  - orgId: 1
    name: shop
    folder: Shop
    interval: 10s
    rules: [...]
```

These are the five rules, by where their data comes from:

| Rule | Data from | Expression | Fires when |
|---|---|---|---|
| redis-exporter down | Prometheus itself | `sum(up{app="redis-exporter"}) or vector(0)` | < 1 for 1m |
| orders queue backlog | redis_exporter | `max by (key) (redis_key_size{key="bull:orders:wait"})` | > 500 for 1m |
| mqtt canary failing | our MQTT exporter | `min(mqtt_canary_success) or vector(0)` | < 1 for 1m |
| container restarting | kube-state-metrics | `max by (namespace, pod, container) (increase(kube_pod_container_status_restarts_total[5m]))` | > 2 for 1m |
| deployment replicas missing | kube-state-metrics | `kube_deployment_spec_replicas - kube_deployment_status_replicas_available` | > 0 for 5m |

The thresholds and `for:` durations are short so the drills finish in a few minutes. On a real cluster you'd set them from what normal looks like for your queues.

A few of these expressions look odd on purpose.

**`or vector(0)` on the "is it there" rules.** When you scale the exporter to 0 or its pod gets deleted, Prometheus doesn't record `up == 0`. The target simply disappears, so there's no `up` series left to compare. A rule like `up{app="redis-exporter"} == 0` would sit quietly through the whole outage. Counting the healthy targets and falling back to `0` turns "gone" into a number the threshold can catch. The canary rule does the same thing: if the MQTT exporter itself is gone, there's no canary either, and that should count as failing.

**`max by (key)` on the backlog rule.** Every label on the series ends up in the email subject. My first version used the raw metric, and the emails looked like this:

```
[FIRING:1] orders queue backlog (redis-exporter db0 Shop 10.244.0.26:9121 kubernetes-pods bull:orders:wait shop monitoring-lab-control-plane redis-exporter-5d6fd944dd-jbpj5 5d6fd944dd warning)
```

That's the pod IP, the node and the ReplicaSet hash of the exporter, none of which helps the person reading it. With `max by (key)`:

```
[FIRING:1] orders queue backlog (Shop bull:orders:wait warning)
```

**`interval: 10s`, not 15s.** I first set the rule group interval to 15s to match the scrape interval. Grafana refused to start at all:

```
Error: ✗ invalid service state: Failed, expected: Terminated, failure: starting module provisioning: invalid service state: Failed, expected: Running, failure: alert rules: invalid alert rule: interval (15s) should be non-zero and divided exactly by scheduler interval: 10
```

Grafana's alert scheduler ticks every 10 seconds, so a rule group's interval has to be a multiple of 10s. And one bad rule file takes the whole Grafana down, not just that rule. The only reason nobody lost their dashboards is that Grafana has a readiness probe: the broken new pod never became Ready, so the old one kept serving. That's the same idea as my [zero-downtime deployments post](/posts/zero-downtime-deployments-kubernetes/), showing up somewhere I didn't expect.

The dashboard goes in a second ConfigMap. It's mainly for the person who just got the email and wants to see what's going on:

![The Grafana dashboard over the half hour of drills. The orders queue panel shows three backlogs climbing and draining. The MQTT row shows the canary dropping to failing once and messages per minute falling to zero around 10:00. The pods row shows the order-worker restarts climbing during the OOM drill.](/images/monitoring-blind-kubernetes-cluster/dashboard-drills.png)

This is the half hour of drills from Step 7: three backlogs in the queue panel (drill 2, then drill 3 twice), the broker going away around 10:00 in the MQTT row, and the OOMKilled worker's restarts in the pods row.

Every query on it is wrapped in `max(...)`. When the exporter pod restarts, the new pod has a new `pod` label, so for a while Prometheus has two series for the same queue, and a stat panel happily shows both.

# Step 7: Drills: Prove the Alerts Fire

A rule that has never fired is a guess. Maybe the expression is wrong, maybe the label doesn't match, maybe the email never leaves Grafana. The only way to know is to break the thing on purpose and wait for the email.

The lab's `drills.sh` does that one failure at a time. It breaks something, polls Grafana's alert API and Mailpit's API every 2 seconds, and logs every state change and email with a timestamp. Then it fixes the thing and waits for the `RESOLVED` email too:

```bash
drill() { # drill <title> <alert> <break cmd> <fix cmd>
  local title=$1 alert=$2 brk=$3 fix=$4 t0 t1 t2 t3
  echo "== $title"
  t0=$(date +%s); log break "$brk"; eval "$brk" >/dev/null
  t1=$(wait_for "$alert" FIRING)     # logs Pending, Alerting, then the email
  t2=$(date +%s); log fix "$fix"; eval "$fix" >/dev/null
  t3=$(wait_for "$alert" RESOLVED)
  echo "   break -> firing email: $(secs $((t1 - t0)))   fix -> resolved email: $(secs $((t3 - t2)))"
}
```

Four drills, one per kind of failure:

| # | What I broke | Alert | Break → email | Fix → resolved email |
|---|---|---|---|---|
| 1 | Scaled `redis-exporter` to 0 | redis-exporter down | 1m48s | 41s |
| 2 | Pointed `order-worker` at a Redis host that doesn't exist | orders queue backlog | 2m15s | 42s |
| 3 | Gave `order-worker` a 24Mi memory limit | container restarting | 2m34s | 2m41s |
| 4 | Scaled `mosquitto` to 0 | mqtt canary failing | 1m26s | 43s |

Drill 3 takes longer to resolve because the rule looks at restarts over the last 5 minutes, so it stays firing until the restarts age out of that window.

All four alerts reached the inbox, and each one resolved by itself after the fix. The fifth rule, deployment replicas missing, doesn't have a drill yet. This is the inbox after the first full run (the crashloop rule in it was later replaced, more on that in drill 3):

![Mailpit inbox with nine emails from Grafana to oncall@shop.lab: FIRING and RESOLVED pairs for redis-exporter down, orders queue backlog, pod crashlooping, and a FIRING mqtt canary failing at the top.](/images/monitoring-blind-kubernetes-cluster/mailpit-inbox.png)

Drills 1 and 4 are the boring ones, and that's good: they prove the `or vector(0)` rules catch something that disappears entirely. Drills 2 and 3 taught me more.

## Drill 2: Running, but not working

I pointed the worker at a Redis host that doesn't exist. I expected it to crash. It didn't:

```
09:44:10  break  kubectl -n shop set env deploy/order-worker REDIS_HOST=redis-typo
09:44:10  state  orders queue backlog: Normal
09:45:14  state  orders queue backlog: Pending
09:46:14  state  orders queue backlog: Alerting
09:46:25  email  [FIRING:1] orders queue backlog (Shop bull:orders:wait warning)
--- kubectl -n shop get pods -l app=order-worker
NAME                           READY   STATUS    RESTARTS   AGE
order-worker-6756d5b98-2n75f   1/1     Running   0          2m15s
---
09:46:43  fix    kubectl -n shop set env deploy/order-worker REDIS_HOST-
09:46:43  state  orders queue backlog: Alerting
09:47:04  state  orders queue backlog: Normal
09:47:25  email  [RESOLVED] orders queue backlog (Shop bull:orders:wait warning)
```

`1/1 Running`, `0` restarts, and not a single job processed. BullMQ's Redis client just keeps retrying quietly. Every Kubernetes-level signal says this pod is fine. kube-state-metrics has nothing to report, and neither does the crashloop or restart rule. The only thing that noticed was the queue backing up behind it:

![Timeline of drill 2. The worker is broken from 0s to 2m33s while showing STATUS Running with 0 restarts. The alert stays Normal while the queue fills up until 1m04s, is Pending until 2m04s, then Alerting. The FIRING email arrives at 2m15s, the fix goes in at 2m33s, the alert returns to Normal at 2m54s and the RESOLVED email arrives at 3m15s.](/images/monitoring-blind-kubernetes-cluster/drill-timeline.svg)

This is the most important lesson of the whole exercise: **alert on the symptom, not only on the cause.** Pod alerts catch pods that crash. They can't catch pods that are alive and useless. The question users care about is "is my order being processed?", and the queue depth answers that directly.

## Drill 3: OOMKilled, and an alert that came late

I gave the worker a 24Mi memory limit, far too small for a Node.js process, expecting a textbook crashloop. Two things happened that I didn't plan for.

**The rollout took the healthy pod away first.** The worker has no readiness probe (it's one of the inherited apps), so the new pod counted as Ready the moment its container started. Kubernetes created the new pod and removed the old, healthy one in the same second:

```
10:02:06  ScalingReplicaSet  Scaled up replica set order-worker-98896b4b4 from 0 to 1
10:02:06  ScalingReplicaSet  Scaled down replica set order-worker-5d4fbc8d76 from 1 to 0
```

A moment later the new pod was OOMKilled, and there was no worker left. With a readiness probe and `maxUnavailable: 0`, the old pod would have kept working while the new one failed. That's the whole point of the [zero-downtime post](/posts/zero-downtime-deployments-kubernetes/). In one of my runs it got even weirder: after 3 restarts, the worker stayed `Running` at 22Mi of its 24Mi limit and processed nothing for 8 minutes, while the queue grew from 520 to over 10,000 jobs. That's drill 2 all over again.

**My crashloop alert was late, and then later.** I went through three versions of the Kubernetes-level rule on this same drill:

| Rule | Break → firing email |
|---|---|
| `kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}`, instant, `for: 1m` | never (gave up after 8 minutes) |
| same, wrapped in `max_over_time(...[5m])` | 5m51s |
| `increase(kube_pod_container_status_restarts_total[5m]) > 2` | 2m34s |
| *the queue backlog alert, same drill, for comparison* | *2m20s* |

The first version looked only at the current instant, and between restarts the container just wasn't in `CrashLoopBackOff` for a full minute. Looking back 5 minutes with `max_over_time` (the same thing kube-prometheus does for its `KubePodCrashLooping` alert) made it fire, but almost 6 minutes after the break. So I went back to Prometheus and looked at what kube-state-metrics reported for that container at every scrape:

| Scrapes (every 15s) | Container state reported | Restarts |
|---|---|---|
| 09:47:30 – 09:50:00 | terminated, reason `OOMKilled` | 1 → 4 |
| 09:50:15 | running | 5 |
| 09:50:30 – 09:51:45 | terminated, reason `OOMKilled` | 5 |
| 09:52:00 – 09:53:00 | waiting, reason `CrashLoopBackOff` | 5 |

For the first four and a half minutes, the container was almost always reported as **terminated with `OOMKilled`**, not as waiting in `CrashLoopBackOff`. That reason only showed up from 09:52:00, once the backoff between restarts had grown long. Meanwhile the restart counter had been going up since the very first scrape. So the rule now counts restarts instead of looking at the state.

![Grafana alert rules filtered to firing: orders queue backlog and container restarting are both firing during the OOM drill.](/images/monitoring-blind-kubernetes-cluster/alerts-firing-drill3.png)

And this is the email for it:

![The container restarting alert email in Mailpit: subject FIRING container restarting for pod order-worker-98896b4b4-2lct8, summary "restarted 2.006 times in 5 minutes", with labels for namespace, pod, container and severity.](/images/monitoring-blind-kubernetes-cluster/email-container-restarting.png)

At that moment `kubectl` showed exactly 2 restarts:

```
NAME                           READY   STATUS    RESTARTS      AGE
order-worker-98896b4b4-2lct8   1/1     Running   2 (86s ago)   2m34s
last terminated: order-worker-98896b4b4-2lct8 OOMKilled
```

`increase()` extrapolates to the edges of its window, so 2 real restarts came out as `2.006` and crossed the `> 2` threshold. For an alert that's fine. Just don't read the value as an exact count.

Even the best version of the pod rule came in after the queue alert, though. The symptom alert won again.

# What can still be improved?

This doesn't give "full" observability, but it gets us some, and it shows where to look. What's still missing:

- **The database layer.** No exporter yet for the database or the proxy in front of it.
- **The log pipeline.** If logs stop flowing, nothing notices until someone goes looking for a log line that isn't there.
- **Things outside the cluster** the apps depend on: shared storage, the SMTP server, external APIs.
- **TLS certificates.** An expired certificate is a classic outage that a simple blackbox probe can warn about weeks ahead.

# Results

| | Before | After |
|---|---|---|
| Alert rules | 0 | 5 |
| Where alerts go | nowhere | an inbox |
| App-level targets scraped | 0 | 2 exporters |
| Custom code | none | one ~60-line exporter |
| Changes to the Prometheus config | n/a | none |
| Failure modes proven with a drill | 0 | 4 |

The work wasn't a big monitoring platform. It was mostly reading: what's already collected, what Prometheus already discovers, what BullMQ already keeps in Redis, what redis_exporter can already do with config. The code I actually wrote was one small exporter, for the one thing no config could give me, a message that really goes through the broker.

What I'd tell myself at the start:

1. **Audit first.** "We have Prometheus" and "someone gets told when it breaks" are different things.
2. **Use what's already collected** before adding anything.
3. **Known exporter first, your own only when it has to be behavior.**
4. **Alert on symptoms** (the queue is growing) as well as causes (the pod is restarting).
5. **Drill every alert.** My crashloop rule looked right, passed review in my head, and was wrong twice. Only breaking things showed me.

