# Distributed n8n on Kubernetes

[![ci](https://github.com/BardiaYaghmaie/distributed-n8n-on-k8s/actions/workflows/ci.yml/badge.svg)](https://github.com/BardiaYaghmaie/distributed-n8n-on-k8s/actions/workflows/ci.yml)

```bash
make all
```

That is the quick start. It builds a local kind cluster running:

|  |  |
|---|---|
| **n8n** | queue mode: Main enqueues, Workers execute |
| **PostgreSQL** | CloudNativePG, 3 instances, automatic failover |
| **Redis** | redis-operator, 3 replicas, 3 Sentinels |
| **Routing** | Traefik behind Gateway API |
| **Autoscaling** | KEDA, on queue depth |
| **Isolation** | default-deny NetworkPolicies |

Every non-obvious line carries a comment saying why it is there. The rest of this page is how the
pieces fit together.

---

## The problem

You want to automate things, so you deploy n8n. One container, one database, and for a while it is
fine. Then the work grows: bigger files, longer workflows, more of them firing at once. You give
the pod more CPU and more memory and it barely helps, because n8n runs on a single Node.js
event loop, and one heavy workflow is one blocked loop. While it grinds through somebody's
50 MB CSV the editor freezes, webhooks time out, and the scheduler fires late, all at the same
time and all for the same reason. So you run a second copy, and that is worse: a webhook lands on
whichever replica the load balancer picked and that replica does all the work, while both of them
now believe they own the scheduled triggers, so your nightly job runs twice. Replicas duplicate
the *whole* process, including the part that must not be duplicated. The fix is to split it
instead: one thing that accepts work, another that does work, and a queue in between so the
second can be multiplied freely. Rails does it with Sidekiq, CI systems with runners, video sites
with encoding farms.

n8n calls it queue mode, and turning it on is one line:

```yaml
EXECUTIONS_MODE: queue
```

Everything else in this repository exists to make that line safe to depend on. The rest of this
page is how.

---

## The cast

```
                        http://n8n.localtest.me
                                  |
                     host :80 -> kind -> NodePort 30080
                                  |
                        +---------------------+
      namespace infra   |  Traefik / Gateway  |   the front door
                        +----------+----------+
                                   |  HTTPRoute
      - - - - - - - - - - - - - - -|- - - - - - - - - - - - - - - - - - - - -
                                   v
                        +---------------------+
      namespace n8n     |      n8n Main       |   accepts work
                        |  editor + API + hooks|
                        +----------+----------+
                                   |  push a job
                                   v
                        +---------------------+
                        |    Redis  master    |   hands work over
                        +----------+----------+   3 replicas + 3 Sentinels
                                   |  blocking pop
                +------------------+------------------+
                v                  v                  v
          +-----------+      +-----------+      +-----------+
          | n8n Worker|      | n8n Worker|      |    ...    |   does the work
          +-----+-----+      +-----+-----+      +-----+-----+   2-10, autoscaled
                |                  |                  |
                +------------------+------------------+
                                   v
                        +---------------------+
                        |     PostgreSQL      |   remembers everything
                        | primary + 2 replicas|
                        +---------------------+
```

**n8n Main** is the part you log into. Editor, REST API, webhook endpoint. In queue mode it stops
executing anything: a request arrives, it writes a row to PostgreSQL, pushes a job to Redis, and
holds the HTTP connection open waiting for someone else to finish. One insert and one push per
request, which is why a single replica keeps up with far more traffic than an executing instance
ever could.

**n8n Workers** are the same container image with a different command, `n8n worker`. No editor, no
API, nothing listening for traffic. A Worker connects to Redis, blocks until a job appears, runs
the workflow, writes the result, and blocks again. Nothing routes to a Worker; it goes and gets
its own work. That one property is what makes them multiply cleanly, because starting a new one
requires informing precisely nobody.

**Redis** is the hand-off, and it is load-bearing. Not a cache, not a speedup you could remove on a
bad day. If Redis is gone, Main has nowhere to put jobs and Workers have nothing to take, and the
platform stops accepting work entirely. That is why it runs replicated here instead of as the
single pod most tutorials give you.

**PostgreSQL** is the shared memory: workflow definitions, encrypted credentials, execution
history. Main and every Worker read and write the same database, and that is the trick that makes
the hand-off cheap. Main never ships a workflow to a Worker. It ships an id, and the Worker looks
the rest up.

**Traefik**, behind a Gateway API `Gateway`, is the front door. **KEDA** watches the depth of the
Redis queue and decides how many Workers there should be.

The two namespaces are deliberate. `infra` holds the operators and the proxy, the things a cluster
admin installs once. `n8n` holds the application and its data. You can delete the whole `n8n`
namespace and the platform underneath it is untouched.

---

## Watch one request go through

```bash
curl -X POST http://n8n.localtest.me/webhook/test \
  -H "Content-Type: application/json" \
  -d '{"message": "hello", "request_id": "12345"}'
```

**DNS.** `n8n.localtest.me` is a real public domain whose A record is `127.0.0.1`, so the request
goes to your own machine's port 80. It also has an AAAA record, `::1`, which matters: see
[Getting in](#getting-in).

**Two hops into the cluster.** kind runs nodes as Docker containers, and `kind/cluster.yaml` maps
host port 80 to port 30080 on the control-plane container. Traefik's Service claims `NodePort
30080`, a port opened on every node that forwards to the pod.

**Traefik finds a route.** The `HTTPRoute` in `kubernetes/60-gateway.yaml` says hostname
`n8n.localtest.me` goes to Service `n8n-main` on port 5678.

**The Service picks a pod.** `n8n-main` selects on `app=n8n` *and* `component=main`. That second
label earns its place: Workers also run an HTTP server, but only for health checks, and a webhook
must never land on one. The selector makes that impossible rather than merely unlikely.

**Main accepts and puts it down.** It writes an execution row to PostgreSQL through `n8n-pg-rw`,
pushes a job onto a Redis list through `n8n-redis-master`, then waits, holding your connection.

**A Worker wakes up.** Every Worker is already blocked on that list; Redis hands the job to exactly
one. It fetches the workflow from PostgreSQL (Main only sent an id), runs the two nodes, writes the
result back.

**The answer comes home.** Main is still holding your connection, so it replies. From outside, one
ordinary request that took 40 ms.

**Main and the Worker never learned of each other's existence.** No service discovery, no
registration, no leader election, no heartbeats. They share a list and a
database, and that is the entire coordination protocol. It is why you can kill a Worker mid-flight,
start five more, or move them to another machine, and nothing needs telling.

---

## The two-node workflow, and why its answer is proof

[`n8n/workflow.json`](n8n/workflow.json) is a Webhook node feeding a Set node. It does nothing
useful on purpose.

```json
{
  "message": "hello",
  "request_id": "12345",
  "processed": true,
  "executed_by": "n8n-worker-5756d8bf46-vg8rd",
  "execution_id": "1"
}
```

The first three fields are an echo. The last two are evidence. `executed_by` is `$env.HOSTNAME`
read from inside whichever process ran the workflow, which under Kubernetes is the pod name.
`execution_id` is the row id in PostgreSQL, so you can go and find the same execution in that
pod's logs.

The webhook uses `responseMode: lastNode`, so n8n holds the connection until the workflow finishes
and returns the last node's output. Since a Worker is what finishes it, getting a response *at
all* already proves the enqueue, dequeue and execute round trip worked.

`make verify` refuses to take the pod name at face value, and checks three separate things:

1. the response says what the workflow should say;
2. that pod name carries `component=worker`, asked of the Kubernetes API rather than
   pattern-matched on the string;
3. that Worker's own log contains this execution id.

Two and three are independent for a reason. A workflow could in principle report a hostname that
is not its own. A log is the process's own account of itself. Together there is no room left for
Main to have quietly run it.

One small design note. The pod name comes from a Set node reading `$env.HOSTNAME`, not a Code node
calling `os.hostname()`. n8n 2.x runs Code nodes in a separate task-runner process, so a Code node
would mean a sidecar container in every Worker pod, infrastructure existing purely to serve a
demo. The Set node gets the same answer in-process.

---

## Redis, up close

n8n uses [BullMQ](https://docs.bullmq.io), which is a job queue built on ordinary Redis lists.
Almost everything about its behaviour follows from thinking of it as *a list that two programs
agree about*.

Main pushes a job onto `bull:jobs:wait`. One list operation. Workers call a blocking pop and then
just wait. An idle Worker costs Redis nothing, no polling and no timers, and a job gets claimed
the instant a Worker is free. And because Redis is single-threaded,
"exactly one Worker gets this job" is not a distributed consensus problem. It is a side effect of
operations happening one at a time.

Two settings in `kubernetes/20-redis.yaml` both come down to choosing to fail loudly.

`maxmemory-policy noeviction`. Redis under memory pressure will, by default, start evicting keys.
For a cache that is correct behaviour. For a queue it means jobs that were accepted and then
silently never ran, which is about the worst failure a system can have, because nothing anywhere
reports an error. Setting `noeviction` makes Redis refuse new writes instead. Something breaks
visibly, which is what you want.

`appendonly yes`. A restart comes back with the queue it had rather than an empty one. This is not
strong durability; with `appendfsync everysec` you can still lose a second of accepted jobs. But
losing a second is a different category of problem from losing everything.

And now the limitation you should know about, because it shapes the whole Redis design here:
**n8n cannot talk to Redis Sentinel.** `QUEUE_BULL_REDIS_HOST` takes one hostname and that is all
it takes. So the three-node Redis does not work by n8n being clever. It works because
redis-operator moves the `n8n-redis-master` Service to whichever pod Sentinel elected, and n8n
reconnects to the same name having noticed nothing. Failover is resolved by DNS, behind n8n's
back. Whether that actually happens is exactly what `make drill` goes and checks.

---

## Why PostgreSQL gets an operator

Most tutorials give you a StatefulSet with one replica. Understanding why that is not enough is
more useful than the fix.

A StatefulSet gives you stable pod names, stable storage, and it restarts a pod that dies. For a
stateless service that is the entire job. For a replicated database it is not, because the thing
that needs to happen after a failure is not "start a pod". It is:

> work out which surviving replica has the most data, promote it to primary, point every client at
> it, and when the old primary comes back, rejoin it as a replica.

None of that can be expressed in a StatefulSet. It needs a program that watches the cluster and
acts on it, and a program like that is what an operator is.

[CloudNativePG](https://cloudnative-pg.io) is that program here. You declare an intention:

```yaml
kind: Cluster
spec:
  instances: 3
```

and it creates the pods, sets up replication, generates the database password into a Secret,
watches the primary's health, promotes a replica when the primary goes, and maintains three
Services:

| Service | Always points at |
|---|---|
| `n8n-pg-rw` | the current primary |
| `n8n-pg-ro` | the replicas, for read-only queries |
| `n8n-pg-r` | any instance |

n8n's `DB_POSTGRESDB_HOST` is `n8n-pg-rw`. After a failover the operator repoints that Service and
n8n reconnects, none the wiser. It is the same trick as Redis: **a stable name whose meaning
moves.**

That idea, declare what you want and let a controller drag reality toward it, is Kubernetes
itself in one sentence. Operators just extend it to things Kubernetes has never heard of, like
what a "primary" is.

---

## Who owns the front door

Routing here uses Gateway API rather than an `Ingress`, and the reason is about ownership more
than features.

An `Ingress` mashes two unrelated decisions into one object living in the application's namespace:
which ports the cluster's proxy exposes, which is a platform decision, and which hostname maps to
which Service, which is an application decision. Because they are the same object, "is this team
allowed to publish on this hostname" is not a question the cluster can answer.

Gateway API pulls them apart:

```
Gateway     (namespace infra)     the platform team's object
              which ports are open, which controller serves them,
              and which namespaces may attach routes

HTTPRoute   (namespace n8n)       the application team's object
              which hostname, which Service, which timeouts
```

The `Gateway` in `kubernetes/60-gateway.yaml` accepts routes only from namespaces labelled
`gateway-access: n8n`, and `kubernetes/00-namespace.yaml` is where that permission is written
down. An application team can publish a route. It cannot grant itself the right to publish one.
Try it from an unlabelled namespace and the route is rejected with
`Accepted: False, reason: NotAllowedByListeners`.

Two things that will bite you if you copy this:

The listener port is **8000**, not 80. Traefik matches a Gateway listener to one of its
entrypoints by port number, and its `web` entrypoint listens on 8000 inside the container. The
port humans connect to is a completely separate concern (NodePort 30080, forwarded from host 80).

Timeouts are API fields (`timeouts.request`), not controller-specific annotations. That
portability is the other thing Gateway API buys you.

There is also a reason not to reach for ingress-nginx, which most guides still recommend: it was
**retired in March 2026**. No more releases, no fixes for anything found after that date, and the
successor project was abandoned.

---

## Growing

```bash
make fanout N=1500
kubectl --context kind-n8n -n n8n get pods -w
```

The queue goes deep, KEDA adds Workers, and once the burst drains it holds them for the
five-minute scale-down window before removing them again. How many it adds depends on how fast
your machine drains the queue, so watch the `get pods -w` output rather than trusting a number
written here. Fire a second `make fanout N=600` while the fleet is still wide and the executions
spread evenly across it.

Use a big number. `N=200` scales nothing: the sample workflow is one Set node, so two Workers at
concurrency 5 clear 200 jobs in about three seconds, faster than the autoscaler's metric window
notices. Autoscaling reacts to a queue that *stays* deep,
not to a spike that has already drained. Real workflows, which sit around waiting on real APIs,
build a queue much more easily than this toy does.

There are two dials and they fix different problems. **Replicas** (KEDA, 2 to 10) is what you
raise when Workers are CPU-bound, or when losing one node would cost you too much capacity.
**`--concurrency`** in `kubernetes/50-n8n-worker.yaml` is what you raise when Workers are sitting
idle waiting on I/O, which is most real work. Capacity is the product of the two. A workflow that
holds a lot of data in memory wants *lower* concurrency and more pods, because everything running
inside one pod shares its memory limit.

What you scale *on* matters more than the numbers. A CPU-based HorizontalPodAutoscaler is the
obvious choice and the wrong one here. A Worker with ten HTTP calls in flight is fully occupied
and almost idle by CPU, so a CPU autoscaler would scale **down** exactly when the queue is growing.
Queue depth is the signal that actually means "work is arriving faster than it leaves".

```yaml
triggers:
  - type: redis
    metadata:
      listName: "bull:jobs:wait"
      listLength: "20"        # ~20 waiting jobs per Worker
```

The general lesson, which outlives n8n: **autoscale on the signal that leads the problem, not the
one that correlates with it.** For a queue that is depth. For a web tier it is usually latency or
in-flight requests. It is almost never CPU.

One Kubernetes detail that catches people out: `kubernetes/50-n8n-worker.yaml` has no `replicas`
field at all. KEDA owns that number. Set it in both places and two controllers spend the rest of
the day overwriting each other, which is a miserable thing to debug.

---

## Breaking it on purpose

```bash
make drill
```

It kills the PostgreSQL primary, kills the Redis master, and tries a connection the network
policies are supposed to refuse. Recovery is timed by when the webhook answers again, three times
in a row, because the first call after a failover can succeed on a connection that has not yet
noticed anything. "Time until the pod is Ready" is a prettier number that answers a different
question.

| Lose | What happens |
|---|---|
| A Worker | Its in-flight executions are retried elsewhere, nothing visible |
| The PostgreSQL primary | CNPG promotes a replica and repoints `n8n-pg-rw` |
| The Redis master | Sentinel elects one, the operator repoints `n8n-redis-master` |
| A whole node | All of the above at once, which is what three nodes are for |
| n8n Main | Webhooks refused while it restarts. Queued executions still finish |

How long each takes depends on your machine, so run it and read your own numbers. Both recoveries
land in tens of seconds. Redis is the slower of the two because the Sentinels have to agree the
master is gone before the operator can repoint anything.

That last row is the remaining single point of failure. Two Mains would need
`N8N_MULTI_MAIN_SETUP_ENABLED` so they elect a leader for schedule triggers, otherwise every cron
workflow fires twice, and that flag is an n8n **Enterprise** feature.

Run the Redis drill yourself. That failover rests entirely on the operator repointing a Service,
and the operator has open bugs about exactly that going stale
([#1711](https://github.com/OT-CONTAINER-KIT/redis-operator/issues/1711),
[#1779](https://github.com/OT-CONTAINER-KIT/redis-operator/issues/1779)). It behaved correctly on
every run here, on one cluster and one version. That is why the drill exists.

---

## Patterns to reuse

These apply well beyond n8n.

**Health checks that mean something.** In [`40-n8n-main.yaml`](kubernetes/40-n8n-main.yaml) both
probes hit `/healthz`, which answers as soon as the HTTP server is listening, and deliberately do
not check the database. If PostgreSQL is unreachable, restarting Main does not fix it, and a
liveness probe that fails on it turns one outage into a restart loop on top of an outage.

**Config split by who reads it.** [`30-config.yaml`](kubernetes/30-config.yaml) is three
ConfigMaps rather than one. `n8n-common` is what Main and Workers must agree on, `n8n-main` is
web-server settings, `n8n-worker` is queue-consumer settings. One big ConfigMap makes every
variable look like it applies to everything, so nobody reading it can tell whether
`N8N_SECURE_COOKIE` affects the Workers. Nothing is lost by splitting: everything that must match
still lives in one object both of them reference.

**Secrets generated by whoever owns them.** The database password is generated by CloudNativePG,
not by a script, because the thing that owns the database should own its credential.
[`30-secrets.sh`](scripts/30-secrets.sh) is idempotent and never rotates: a new
`N8N_ENCRYPTION_KEY` makes every credential already in the database permanently undecryptable.

**Spread across failure domains, with the right strictness.**
[`50-n8n-worker.yaml`](kubernetes/50-n8n-worker.yaml) uses `topologySpreadConstraints` with
`whenUnsatisfiable: ScheduleAnyway`, a preference that stops blocking once there are more Workers
than nodes. `10-postgres.yaml` uses `enablePodAntiAffinity`, which is stricter, because three
database instances on one machine defeats the entire purpose of three.

**A PodDisruptionBudget, and knowing what it isn't.** `minAvailable: 1` means a node drain cannot
leave the queue with nobody consuming it. It constrains voluntary disruption only. It will not
save you from a machine catching fire.

**A grace period that outlasts the work.** `terminationGracePeriodSeconds: 40`, longer than the 30
seconds n8n spends draining executions on SIGTERM. Shorter and Kubernetes SIGKILLs a Worker
mid-execution. Match the grace period to what the process actually does when asked to stop.

**Default-deny networking.** [`80-networkpolicy.yaml`](kubernetes/80-networkpolicy.yaml) denies
everything, then allows the six flows that exist. Only Workers get internet egress, because
workflows call external APIs and Main has no business doing so; a Main that can reach the internet
is mostly useful to an attacker. And the egress rule excludes RFC 1918 space, because
`0.0.0.0/0` on its own would also hand a Worker every pod in the cluster.

Enforced, not decorative: kind's default CNI has supported NetworkPolicy since v0.24, and
`make drill` proves it with a connection that has to fail. Check this on any cluster you deploy
to. A CNI without support accepts the objects and ignores them, with no error anywhere.

**Never trusting the ambient context.** [`scripts/common.sh`](scripts/common.sh) wraps `kubectl`
and `helm` as shell functions pinned to `--context kind-n8n`. `kubectl config use-context` is not
good enough, because it sets global state that anything on the machine can change between two
commands. These scripts install operators and delete pods. That does not belong in whatever
cluster you happen to have selected.

---

## Running it

**You need** `docker` 24+, `kind` 0.30+, `kubectl` 1.29+, `helm` 3.14+, and `curl`, `python3`,
`openssl`. Host port 80 has to be free. The cluster settles at about **3.5 GB** of RAM; give it 6
for headroom, or see [running smaller](#running-smaller).

`make all` runs six steps, each of which also runs on its own:

| | | |
|---|---|---|
| 1 | `make cluster` | kind cluster, 1 control-plane and 3 workers |
| 2 | `make operators` | Gateway API + Traefik, CloudNativePG, redis-operator, KEDA, into `infra` |
| 3 | `make secrets` | Generate the Redis password, encryption key and editor login |
| 4 | `make deploy` | Apply `kubernetes/` and wait for everything |
| 5 | `make seed` | Create the owner account, import and activate the workflow |
| 6 | `make verify` | Call the webhook and prove a Worker ran it |

Nearly all of the wall-clock time is pulling about 2 GB of images, so how long it takes is mostly
a question of your connection. Everything comes from the projects' own registries and Helm repos
at pinned versions. No mirrors, no preloading, no digest overrides, which is what lets
`kubernetes/` apply unchanged to a real cluster.

Then:

```bash
make credentials       # the generated editor login for http://n8n.localtest.me
make fanout N=1500     # a burst deep enough for KEDA to react to
make drill             # kill the primary, kill the Redis master, time the recovery
make logs              # follow every Worker at once
make scale REPLICAS=5  # override KEDA by hand
make down              # delete the cluster
```

### Getting in

The editor is at **http://n8n.localtest.me** and `make credentials` prints the login.

`localtest.me` is a real public domain whose A record is `127.0.0.1`, which is how this works with
no setup. Add one line to `/etc/hosts` anyway:

```
127.0.0.1  n8n.localtest.me
```

That single line fixes two things that otherwise look like a broken deployment when the platform
is running perfectly.

**Your browser may refuse to connect.** `n8n.localtest.me` also has an AAAA record, `::1`, and
browsers prefer the IPv6 answer. kind publishes port 80 on IPv4 only (Docker binds one address
family per mapping, and kind rejects a second mapping on the same port), so IPv6 gets an instant
refusal and Chrome says `ERR_CONNECTION_REFUSED`. `curl` falls back to IPv4 and reports 200, which
makes it confusing to diagnose. A hosts entry with only the IPv4 address wins over
DNS and removes the IPv6 answer entirely.

**It breaks when you go offline.** The name is resolved by a *public* resolver, so with no network
the lookup fails and nothing loads, even though every component is running on your own machine
and the internet is not otherwise involved. The hosts entry resolves it locally.

### Running smaller

To trade the failover away for a smaller footprint and keep everything else:

- `kubernetes/10-postgres.yaml` → `instances: 1`
- `kubernetes/20-redis.yaml` → `clusterSize: 1` on both resources
- `kind/cluster.yaml` → one worker node

### Layout

The numbers are apply order. Reading order is different: start at `30-config.yaml` for what
queue mode actually configures, `40-n8n-main.yaml` and `50-n8n-worker.yaml` for the split,
`20-redis.yaml` and `10-postgres.yaml` for the data layer, then `60-gateway.yaml`,
`70-autoscale.yaml` and `80-networkpolicy.yaml`.

```
Makefile                one target per step
kind/cluster.yaml       1 control-plane + 3 workers, host port 80
kubernetes/             the deployment, as plain commented manifests
  00-namespace   10-postgres   20-redis      30-config
  40-n8n-main    50-n8n-worker 60-gateway    70-autoscale   80-networkpolicy
scripts/                one script per Makefile target
  common.sh  10-cluster  20-operators  30-secrets  40-deploy
  50-seed    60-verify   70-fanout     80-drill    teardown
n8n/workflow.json       Webhook -> Set. The only workflow
docs/
  architecture.md       what runs, and why it is split this way
  decisions.md          the choices that could have gone the other way
  operations.md         scaling, failover drills, failure scenarios
  security.md           what is protected, and what is not
  production.md         what changes when this carries real work
```

---

## What is deliberately missing

Being specific about the gaps is more useful than pretending there are none. Each of these has an
answer in [docs/production.md](docs/production.md).

**TLS.** Plain HTTP, editor login included. This is also why `N8N_SECURE_COOKIE=false` is set, and
that flag is correct *only* because there is no TLS. Adding certificates means removing it in the
same change.

**Backups.** Replication survives a lost node. It does not survive a dropped table, because
replicas replicate mistakes with total fidelity. Availability and backups are different problems.

**A second Main.** Multi-main is an n8n Enterprise feature, so this deployment has one, and a
restart refuses webhooks for about twenty seconds.

**SSO and projects.** One shared owner account. Also Enterprise features.

**A monitoring stack.** `/metrics` is exposed on both roles and nothing scrapes it. Point an
existing Prometheus at it and you get data without touching these manifests.

**GitOps.** Deployed from a laptop rather than reconciled from git.

---

## Licence

MIT for the manifests, scripts and docs here. n8n itself is fair-code under the Sustainable Use
License, which is not the same as open source. See [LICENSE](LICENSE).
