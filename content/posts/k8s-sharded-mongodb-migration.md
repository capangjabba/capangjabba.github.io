+++
title = 'Custom Sharded MongoDB Migration in k8s with Minimal Downtime'
date = 2026-09-29T12:54:24+08:00
draft = false
categories = ["DevOps"]
tags = ["kubernetes", "mongodb", "migration"]
+++

<!-- TODO: 2-3 sentence summary: 7.0.20 → 7.0.30 to patch MongoBleed, rolling upgrade, how much downtime in the end. -->

# Background

Few months back in my previous company, I led the project to upgrade our Customer Shared MongoDB Cluster which deployed on kubernetes cluster. Our MongoDB infra at that time doesnt have any kubernetes operator and requires manual upgrade/migration process. In this blog I want to explain the Sharded MongoDB Architecture and how kubernetes features come into play during this migration.

<!-- TODO: why now: MongoBleed (CVE-2025-14847), fixed in 7.0.28. -->

# Understanding Sharded MongoDB Cluster in kubernetes

In a sharded MongoDB Cluster, It involves 3 Main Components
1. Mongos Router
2. Config Server/Metada
3. Shards

MongoDB official documentation provides very in-depth detail about this: https://www.mongodb.com/docs/manual/core/sharded-cluster-components/#sharded-cluster-components , however I like to also explain how I understands it

## Mongos Router

Mongos Router is a "stateless" component that routes MongoDB traffic/connection to the sharded cluster. It provides an interface for applications to communicate with the shards. Applications connection with the sharded cluster is never direct, it will go through Mongos Router.

## Config Server/Metadata

Config Server/Metadata is the component responsible in having the knowledge on "which data is stored at which shards", hence the name, metadata. Mongos Router frequently communicates with Config Server to route any operations to the correct shards. In simple words, Config Server is like the "Table of Contents" for the shards.

## Shards

Shards are the component that is storing our data.

<!-- TODO: each shard (and the config server) is itself a 3-member replica set: 1 primary, 2 secondaries. -->

# Our Setup

We ran the same topology in every environment: one config server replica set, two shards, and mongos in front.

```
                       applications
                             │
                             ▼
              ┌────────────────────────────┐
              │ mongos (router)            │
              │ Deployment, HPA 1-3 pods   │
              └──────────────┬─────────────┘
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│ rs0 (config)   │  │ shard1         │  │ shard2         │
│ StatefulSet    │  │ StatefulSet    │  │ StatefulSet    │
│ rs0-{0,1,2}    │  │ shard1-{0,1,2} │  │ shard2-{0,1,2} │
└────────────────┘  └────────────────┘  └────────────────┘
```

- **Config servers and shards are StatefulSets.** Each pod keeps a stable name (`shard1-0`, `shard1-1`, ...) and its own PersistentVolumeClaim, so a restarted pod comes back as the same replica set member with the same data.
- **mongos is a Deployment.** It stores nothing, so any pod can be replaced at any time, and a HorizontalPodAutoscaler scales it between 1 and 3 pods.
- **We build our own `mongod` and `mongos` images.** Upgrading means building new images and bumping the image tag in the manifests.

|  | Dev | Stage | Prod |
|---|---|---|---|
| Platform | Rancher / Harvester | AKS | AKS |
| Backup before upgrade | `mongodump` only | Disk snapshots + `mongodump` | Disk snapshots + `mongodump` |
| Risk | Low | Medium | High |

Dev has no volume snapshots, because at that time, longhorn doesnt support kubernetes VolumeSnapshoting feature, so a `mongodump` is the only backup there. Stage mirrors Prod exactly, so the procedure is proven on the same topology before it touches Prod.

# The Upgrade Plan

7.0.20 → 7.0.30 is a patch upgrade inside the same 7.0 release series, which keeps it low-risk:

- **The feature compatibility version (FCV) doesn't change.** It stays at `"7.0"` before, during and after.
- **Rolling back is just redeploying the old image.** Downgrading within the same release series is supported, so there's no restore needed unless data is actually damaged.

The order matters for a sharded cluster:

1. **Stop the balancer.** The balancer moves chunks of data between shards in the background, and you don't want a migration running while shard members restart underneath it.
2. **Upgrade the config servers (`rs0`).**
3. **Upgrade the shards, one at a time.** Only one shard is ever mid-restart.
4. **Upgrade mongos.**
5. **Start the balancer again.**

This is the order MongoDB documents for upgrading sharded clusters. Each step ends with a stop/go check: if a replica set doesn't come back healthy, stop there and decide whether to fix forward or roll back.

Inside each step, Kubernetes does the rolling for us. With the `RollingUpdate` strategy, a StatefulSet restarts its pods one at a time from the highest ordinal down (`-2`, `-1`, then `-0`), and waits for each pod to be Ready before moving on to the next.

We rehearsed the whole thing in Dev, then Stage, and only then Prod.

# Doing the Upgrade 

## 1. Pre-checks

Everything here must pass before starting.

```javascript
// On mongos
sh.status()             // both shards listed, none draining
sh.getBalancerState()   // note the current state
db.version()            // "7.0.20"
db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })  // { version: "7.0" }

// On each replica set (rs0, shard1, shard2)
rs.status()             // every member PRIMARY or SECONDARY, nothing RECOVERING or DOWN
```

Also check that disk usage is under 80% on every data volume (`kubectl exec shard1-0 -- df -h /db/data`), and that nobody else is deploying to the cluster.

## 2. Back up

Stop the balancer first so no chunks move while the dump runs. Then stream a compressed dump from mongos straight to your machine. The admin credentials are read from the pod's own environment, so they never appear in your local shell.

```bash
kubectl -n mongodb exec deploy/mongos -- sh -c \
  'mongosh --quiet -u "$MONGOS_ADMIN_USER" -p "$MONGOS_ADMIN_PASSWORD" --authenticationDatabase admin --eval "sh.stopBalancer()"'

kubectl -n mongodb exec deploy/mongos -- sh -c \
  'mongodump -u "$MONGOS_ADMIN_USER" -p "$MONGOS_ADMIN_PASSWORD" --authenticationDatabase admin --gzip --archive' \
  > "mongodump-$(date +%Y%m%d).archive.gz"
```

A backup you haven't restored isn't a backup yet. Restore it into a throwaway container to prove it works:

```bash
docker run -d --name restore-test mongo:7.0.20
docker exec -i restore-test mongorestore --gzip --archive < mongodump-YYYYMMDD.archive.gz
docker rm -f restore-test
```

## 3. Build the images and bump the tag

Build the `mongod` and `mongos` images on 7.0.30, and check the binary inside each one:

```bash
docker run --rm <registry>/mongod:7.0.30 mongod --version
docker run --rm <registry>/mongos:7.0.30 mongos --version
```

Push them, then update the image tag in all four manifests (config servers, `shard1`, `shard2` and mongos) in a single commit. That commit is also your rollback: reverting it puts every manifest back on 7.0.20.

## 4. Do the upgrade

### 4.1 Stop the balancer

```bash
echo "=== Disabling balancer ==="
mongos_eval "sh.stopBalancer()"
BALANCER_STOPPED=1
mongos_eval "sh.getBalancerState()"
```

The balancer keeps data evenly spread by moving chunks from one shard to another. Each move involves three parties: the shard giving the chunk, the shard receiving it, and the config servers, which record the new owner at the end. We're about to restart members of all three, one after another. A migration caught in the middle of that gets aborted and retried, which is wasted work and noise in the logs at exactly the moment you want a quiet cluster.

`sh.stopBalancer()` doesn't just flip a switch. It waits for any migration already in progress to finish, so by the time it returns, nothing is moving. `sh.getBalancerState()` should then print `false`.

Your applications don't notice any of this. Reads and writes carry on as normal. The only thing that pauses is rebalancing, which is fine for the length of an upgrade.

### 4.2 Upgrade config servers first

```bash
echo "=== Upgrading config servers (rs0) ==="
kubectl -n "$NS" apply -f "mongodb_config.yml"
kubectl -n "$NS" rollout status statefulset/rs0 --timeout=10m
```

`kubectl apply` changes the image in the StatefulSet's pod template. The StatefulSet controller then replaces the pods one at a time: `rs0-2` first, waits for it to be Ready, then `rs0-1`, then `rs0-0`. `kubectl rollout status` just watches this and returns once all three pods are running the new image. The 10-minute timeout makes the script fail if a pod gets stuck (for example on an image pull error or a crash loop) instead of waiting forever.

Why config servers first? Every other component depends on them. mongos asks them where data lives, and shards report chunk changes to them. MongoDB's upgrade procedure for sharded clusters always goes config servers, then shards, then mongos, so the components that everyone else talks to are upgraded before the ones that talk to them. For a patch upgrade within 7.0, mixed patch versions can run side by side during the rollout, so the order matters less here than in a major upgrade. We still keep it, so the same script works for the next major upgrade.

What happens to traffic while the config servers restart? mongos keeps a cached copy of the routing table, so ordinary reads and writes keep flowing even while the config replica set elects a new primary. What briefly pauses is anything that changes metadata, like creating or sharding a collection, and chunk migrations, which we already stopped.

### 4.3 Shards, one at a time

There are two kinds of "one at a time" here, and both matter.

**Pods within a shard restart one at a time**, and that comes from Kubernetes. A 3-member replica set needs a majority (2 members) up to elect a primary and to acknowledge majority writes. The StatefulSet only ever takes down one pod, so two members are always up and the shard never loses its majority. If two members went down together, the shard would have no primary and couldn't accept writes until they came back.


```bash
# Before starting: the primary must be ${SHARD}-0
kubectl -n "$NS" exec "${SHARD}-0" -- mongosh --quiet --eval 'db.hello().isWritablePrimary'   # true

# 1. Upgrade only -2 and -1
kubectl -n "$NS" patch statefulset "$SHARD" --type merge \
  -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":1}}}}'
kubectl -n "$NS" apply -f "mongodb_${SHARD}.yml"
kubectl -n "$NS" rollout status "statefulset/${SHARD}" --timeout=10m

# 2. Hand the primary role to an upgraded member
kubectl -n "$NS" exec "${SHARD}-0" -- mongosh --quiet --eval 'rs.stepDown()'

# 3. Upgrade -0, which is now a secondary
kubectl -n "$NS" patch statefulset "$SHARD" --type merge \
  -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
kubectl -n "$NS" rollout status "statefulset/${SHARD}" --timeout=10m
```

**Shards upgrade one at a time**, and that comes from the loop and the stop/go gate. Each shard owns a different slice of the data, so an election on `shard1` only affects writes to `shard1`'s data, and only for a few seconds. Upgrading both shards at once would mean elections on all your data at the same time. Worse, if the new image turned out to be broken, both shards would be broken together. Going one shard at a time means a bad image hurts one shard and gets caught at the gate.

The `SHARDS` array also means that adding `shard3` later is a one-word change.

#### Moving the primary last, on purpose

A StatefulSet doesn't know which member is the primary. It always rolls from the highest ordinal down: `-2`, `-1`, then `-0`. If the primary happens to be `-2`, it's the first pod killed, and the shard has to hold an election. The new primary might be `-1`, which gets restarted next, forcing another election. One careless rollout can put a shard through up to three elections.

So we control where the primary is, and when it moves:

1. **Make sure the primary is `-0` before starting.** The rollout starts from the highest ordinal, so the lowest one is restarted last. If the primary is somewhere else, move it to `-0` first, for example by giving `-0` a higher priority in the replica set config.
2. **Set `rollingUpdate.partition` to 1.** With a partition set, a StatefulSet only updates pods whose ordinal is greater than or equal to the partition. The rollout upgrades `-2` and `-1` and then stops, leaving `-0`, the primary, on the old version. The shard now has two upgraded secondaries and one old primary.
3. **Step down the primary.** `rs.stepDown()` waits (up to 10 seconds by default) for a secondary to catch up, then hands over the primary role. `-0` can't take it back for the next 60 seconds, so the new primary is one of the two members we just upgraded.
4. **Set the partition back to 0.** The StatefulSet now upgrades `-0`, which is just a secondary. Restarting a secondary doesn't cause an election.

The result is exactly one election per shard, and it's a planned handover between healthy, caught-up members, not a primary disappearing mid-write. That's how we keep downtime to a minimum. Writes to that shard only fail while the election runs, and MongoDB drivers retry a failed read or write once by default (retryable reads and writes), so the application usually doesn't see an error at all.

If you pinned `-0` as primary with a higher priority, it takes the primary role back once it has rejoined and caught up. That's a second election, but also a planned one, and by then every member is on 7.0.30.

### 4.4 mongos last

```bash
echo "=== Upgrading mongos ==="
kubectl -n "$NS" apply -f "mongos.yml"
kubectl -n "$NS" rollout status deployment/mongos --timeout=10m
mongos_eval "db.version()"
```

mongos goes last because it's a client of everything else. Once the config servers and shards are on 7.0.30, mongos never runs a newer version than the cluster behind it.

mongos is a Deployment, not a StatefulSet, and Kubernetes rolls it differently. With the default rolling update settings, a Deployment starts a new pod and waits for it to be Ready before removing an old one. Connections open on an old pod are closed when that pod stops, and the application drivers reconnect through the Service to another one.

That covers the rollout itself. But mongos is the only way into the cluster: if no mongos pod is running, every application loses the database, however healthy the shards are. And the rollout isn't the only thing removing pods during a maintenance window. A node drain, a node pool upgrade or the cluster autoscaler can evict a mongos pod at the same moment. Two settings make sure there's always one left:

- **The HPA's minimum is 2 replicas.** With a single replica, losing that one pod takes the whole cluster offline until a replacement is Ready. With two, one can go while the other keeps serving, and the drivers reconnect to it.
- **A PodDisruptionBudget (PDB) with `minAvailable: 1`.** Kubernetes checks the PDB before any voluntary eviction, such as a node drain. If evicting a mongos pod would leave none running, the eviction is refused and retried until another pod is Ready.

The two only work together. A PDB doesn't slow down the Deployment's own rolling update; the rollout settings above control that. What the PDB guards against is evictions happening at the same time as the rollout. And a PDB of `minAvailable: 1` on a single replica never allows an eviction at all, so a node drain gets stuck waiting forever. With at least 2 replicas, the PDB lets pods be evicted one at a time while one always keeps serving.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mongos
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mongos
  minReplicas: 2
  maxReplicas: 3
  # no metrics set: defaults to 80% average CPU utilization
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: mongos
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: mongos
```

`db.version()` runs on mongos itself, so it should now print `7.0.30`. This is also a good moment to check that a normal read and write from the application works through the new mongos.

### Balancer back on

```bash
echo "=== Re-enabling balancer ==="
mongos_eval "sh.startBalancer()"
BALANCER_STOPPED=0
mongos_eval "sh.getBalancerState()"

echo "Done."
```

With the whole cluster on 7.0.30, chunk migrations are safe again, so the balancer goes back on and `sh.getBalancerState()` should print `true`. Leaving it off for long means new data can pile up unevenly on one shard. Setting `BALANCER_STOPPED=0` also tells the exit trap there's nothing to warn about.

### The full script

<details>
<summary>upgrade.sh</summary>

```bash
#!/usr/bin/env bash
# Rolling upgrade of a sharded MongoDB cluster on Kubernetes.
# Order: balancer off -> config servers -> shards (one at a time) -> mongos -> balancer on.
# The manifests must already point at the new image tag.
set -euo pipefail

NS="${NS:-mongodb}"
SHARDS=(shard1 shard2)
ROOT="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"

# Run a mongosh command on mongos. Credentials come from the pod's own
# environment, so they never leave the pod.
mongos_eval() {
  kubectl -n "$NS" exec deploy/mongos -- sh -c \
    'mongosh --quiet -u "$MONGOS_ADMIN_USER" -p "$MONGOS_ADMIN_PASSWORD" --authenticationDatabase admin --eval "$1"' \
    _ "$1"
}

# Show each member of a StatefulSet with its readiness and image.
show_pods() {
  kubectl -n "$NS" get pods "$1-0" "$1-1" "$1-2" \
    -o custom-columns='POD:.metadata.name,READY:.status.containerStatuses[*].ready,IMAGE:.spec.containers[*].image'
}

confirm_proceed() {
  read -r -p "=== STOP/GO: $1. Continue? (yes/no): " CONFIRM
  if [ "$CONFIRM" != "yes" ]; then
    echo "STOPPED."
    exit 1
  fi
}

# If we stop partway, remind whoever is running this that the balancer is off.
BALANCER_STOPPED=0
on_exit() {
  if [ "$BALANCER_STOPPED" = 1 ]; then
    echo "WARNING: the balancer is still disabled. Run sh.startBalancer() on mongos once the cluster is settled."
  fi
}
trap on_exit EXIT

echo "=== Disabling balancer ==="
mongos_eval "sh.stopBalancer()"
BALANCER_STOPPED=1
mongos_eval "sh.getBalancerState()"

echo "=== Upgrading config servers (rs0) ==="
kubectl -n "$NS" apply -f "$ROOT/manifests/mongodb_config.yml"
kubectl -n "$NS" rollout status statefulset/rs0 --timeout=10m
show_pods rs0
confirm_proceed "Config servers upgraded, verify health"

for SHARD in "${SHARDS[@]}"; do
  echo "=== Upgrading ${SHARD} ==="
  kubectl -n "$NS" apply -f "$ROOT/manifests/mongodb_${SHARD}.yml"
  kubectl -n "$NS" rollout status "statefulset/${SHARD}" --timeout=10m
  show_pods "$SHARD"
  confirm_proceed "${SHARD} upgraded, verify health"
done

echo "=== Upgrading mongos ==="
kubectl -n "$NS" apply -f "$ROOT/manifests/mongos.yml"
kubectl -n "$NS" rollout status deployment/mongos --timeout=10m
mongos_eval "db.version()"
confirm_proceed "mongos upgraded, verify routing"

echo "=== Re-enabling balancer ==="
mongos_eval "sh.startBalancer()"
BALANCER_STOPPED=0
mongos_eval "sh.getBalancerState()"

echo "Done."
```
</details>

## 5. Post-checks

```javascript
// On mongos
db.version()            // "7.0.30"
sh.status()             // both shards listed, none draining
sh.getBalancerState()   // true

// On each replica set
rs.status()                        // every member PRIMARY or SECONDARY
rs.printSecondaryReplicationInfo() // secondaries caught up, lag of a few seconds at most
```

Then confirm the applications are healthy and can read and write, and keep an eye on latency, errors and replication lag for the next day.

# Rolling Back

Rollback is the upgrade in reverse: mongos first, then the shards in reverse order, then the config servers.

```bash
git revert <upgrade-commit>   # manifests back on 7.0.20

kubectl -n mongodb apply -f manifests/mongos.yml
kubectl -n mongodb rollout status deployment/mongos --timeout=10m

for SHARD in shard2 shard1; do
  kubectl -n mongodb apply -f "manifests/mongodb_${SHARD}.yml"
  kubectl -n mongodb rollout status "statefulset/${SHARD}" --timeout=10m
done

kubectl -n mongodb apply -f manifests/mongodb_config.yml
kubectl -n mongodb rollout status statefulset/rs0 --timeout=10m
```

Then start the balancer again, and run the pre-checks from step 1. Everything should report 7.0.20.

# Where the Downtime Actually Happens

<!-- TODO: the measured numbers. -->

# Lessons Learned

<!-- TODO -->

