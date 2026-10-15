# NRP Usage Policies

Source: https://nrp.ai/documentation/userdocs/start/policies/ — check it for
changes; the values below were current when this skill was written.

NRP has no billing. In exchange, resources you request are unavailable to
others until your pod ends, so the policies are about **using what you request
and releasing it**.

---

## Requests and Limits (enforced at apply time)

- Every container needs CPU and memory requests and limits.
- Each limit must be within **20%** of its request.
- When running more than about **100** pods or Jobs, set limits equal to requests.
- A container over its memory limit is OOMKilled.

What the Gatekeeper admission policies enforce at `kubectl apply` (observed
October 2026; `kubectl get constraints` shows the current set):

| Constraint | Applies to | Action |
|---|---|---|
| CPU + memory requests and a memory limit required | Job, Deployment | **deny** |
| Limit/request ratio ≤ 1.2 | Job | warn (still policy; monitored) |
| Requests and limits on bare pods | Pod | warn |
| Setting `CUDA_VISIBLE_DEVICES` | Pod, Job, CronJob, Deployment, StatefulSet, ... | **deny** |
| Tolerating `nautilus.io/system` or `node-role.kubernetes.io/control-plane` | Pod | **deny** |
| Linstor PVC size (512 Gi; 64 Gi for `linstor-ha`) | PVC | **deny** |
| DaemonSets, LoadBalancer / NodePort Services, `nodeName` | various | **deny** |

A denied apply prints `admission webhook "validation.gatekeeper.sh" denied
the request: [<constraint>] ...`; a warning prints `Warning: [<constraint>] ...`
and the object is still created. Treat warnings as errors to fix.

## Utilization Thresholds (monitored)

| Resource | Target, as a share of the request |
|---|---|
| GPU | above 40% (ideally near 100%) |
| CPU | 20–200% |
| Memory | 20–150% |

- No more than **4 pods** in a namespace may be outside these thresholds.
- Pods requesting 1 CPU core and 2 GB of memory or less are exempt.
- Admins may ban namespaces whose requests are consistently underused.
- See your pods' status on the **Violations** page of the portal (https://nrp.ai),
  and utilization on Grafana: https://grafana.nrp-nautilus.io.

## Workload Types

| Type | Rules |
|---|---|
| **Bare pod** (no controller: "interactive") | Destroyed after **6 hours** unless the namespace has an exception. Max **2 GPUs, 32 GB RAM, 16 CPU cores**. `sleep` is allowed. |
| **Job** | The way to run batch work. Guaranteed QoS optional. The command must end by itself: **`sleep infinity`, or a script that ends in a sleep, leads to a ban.** |
| **Deployment** | For long-running idle services. Minimal requests with proper limits (Burstable QoS). **Cannot request GPUs.** Deleted after **2 weeks** unless the namespace is on the exception list for a permanent service. |

- Permanent services need an exception from the admins through support,
  with a description and an estimated run period. Long idle pods cannot get an exception.
- You get 3 notifications before a Deployment is purged. Data on persistent volumes stays.

## GPUs

- Request a GPU only if the code uses it, and more than one only if one GPU is near 100%.
- Plans involving more than **50 GPUs** must be presented to support first.
- A100 quota is **0** by default for every namespace; request access through the A100 form.
- H100, H200 and GH200 are reserved for contributing groups and the LLM service.

## Scale and Fairness

- Nautilus has **no fair queue**. Submitting ~1000 Jobs at once can block all
  other users. Use an Indexed Job with a `parallelism` cap, or a work queue
  (Redis/RabbitMQ) with a fixed pool of workers.

## Storage

- Storage is not for archives. Delete what you do not need.
- A volume not accessed for **6 months** can be purged without notice.

## Agent Checklist Before `kubectl apply`

1. Right context and namespace.
2. Every container: CPU and memory requests and limits, limit ≤ 1.2 × request; no `CUDA_VISIBLE_DEVICES`.
   `kubectl apply --dry-run=server -f <file>` shows Gatekeeper denials and warnings without creating anything.
3. A Job, not a Deployment, for computation; no GPU in a Deployment.
4. The command ends on its own; no `sleep` at the end of a Job.
5. `activeDeadlineSeconds` and `ttlSecondsAfterFinished` set.
6. Requests match what the code uses (do not reserve 8 GPUs for a single-GPU script).
7. For more than a few dozen tasks, a `parallelism` cap.
8. The user has seen the manifest and agreed.
