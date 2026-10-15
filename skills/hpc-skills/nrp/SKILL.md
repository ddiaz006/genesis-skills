---
name: nrp
description: >-
  National Research Platform (NRP) Nautilus Kubernetes cluster. Use when
  running Jobs or pods on NRP with kubectl, setting up access (CILogon,
  kubelogin, nrp.ai/config, namespaces), following NRP policies (request/limit
  ratio, no sleep in Jobs, 6-hour pods, 2-week Deployments), requesting GPUs
  (gpu.product affinity, nvidia.com/a100, opportunistic), using Ceph storage
  or S3, or the managed LLM endpoint ellm.nrp-nautilus.io.
compatibility: Requires kubectl and the kubelogin (oidc-login) plugin on the local machine, an NRP account, and membership in an NRP namespace. No login node or SSH is involved.
metadata:
  version: "1.0"
  system: nautilus
  facility: nrp
  scheduler: kubernetes
allowed-tools: Bash(kubectl *) Read Write
---

# NRP Nautilus — National Research Platform

Nautilus is the National Research Platform's nationally distributed
Kubernetes cluster: 500+ nodes at 50+ institutions, 1,500+ GPUs (consumer
cards through A100/H100/H200/GH200), Qualcomm Cloud AI 100 accelerators,
and Ceph storage in several regions. There is **no login node and no batch
scheduler**: you submit Kubernetes objects with `kubectl` from your own
machine. There is no allocation or billing; access is through a
**namespace**, and resources are governed by usage policies.

For general Kubernetes (Job fields, Indexed Jobs, debugging) also load the
`kubernetes` skill.

## Critical Requirements

> **Every container must set CPU and memory `requests` and limits, with each
> limit no more than 20% above its request.** Gatekeeper rejects Jobs and
> Deployments without CPU/memory requests and a memory limit, and warns on
> Jobs whose limits exceed 1.2× the requests. The ratio is still policy.
> Use `requests == limits` unless you have a reason not to.

> **Never set `CUDA_VISIBLE_DEVICES` yourself.** Gatekeeper rejects any
> workload that sets it; request GPUs through `resources` and let the device
> plugin assign them.

> **Never run `sleep infinity` (or a script that ends in a sleep) in a Job.**
> This leads to a ban from the cluster. Jobs must run a command that ends by
> itself.

> **Deployments cannot request GPUs and are deleted after 2 weeks.** Bare
> pods (no controller) are deleted after 6 hours. Use Jobs for computation.

## Access

1. Log in at https://nrp.ai with your institution through CILogon. This creates the account.
2. Get a namespace. Students: ask a supervisor to add you. Faculty, staff, postdocs: request one at https://get-started.nrp-nautilus.io. Memberships: https://nrp.ai/namespaces.
3. Accept the Acceptable Use Policy and read the cluster policies.
4. Install `kubectl` and the `kubelogin` plugin, installed as `kubectl-oidc_login` on the `PATH`. The kubeconfig does not work without it.
5. Get the kubeconfig and set your namespace:

```bash
mkdir -p ~/.kube && curl -fsSL -o ~/.kube/config https://nrp.ai/config   # overwrites an existing config: merge if you have one
kubectl config use-context nautilus
kubectl get nodes                       # opens the CILogon login in a browser
kubectl config set contexts.nautilus.namespace <namespace>
kubectl get pods                        # "No resources found" means access works
```

The OIDC token lasts about 30 minutes and renews on its own. After you join a
new namespace, run `kubectl oidc-login clean` to refresh it. On a machine
without a browser, add `--grant-type=device-code` and `--skip-open-browser` to
the `oidc-login get-token` arguments in the kubeconfig.

Details: `references/access.md`.

## Usage Policies

| Rule | Value |
|---|---|
| Limit vs request | Each limit within 20% of its request; `limits == requests` when running >100 pods/jobs |
| Bare (interactive) pods | Deleted after **6 hours**; max **2 GPUs, 32 GB RAM, 16 CPU cores** per pod; `sleep` allowed |
| Jobs | Batch work; `sleep infinity` or equivalent **prohibited (ban)** |
| Deployments | No GPUs; minimal requests (Burstable QoS); deleted after **2 weeks** unless the namespace has an exception |
| Utilization targets | GPU > 40% of requested; CPU 20–200%; RAM 20–150% |
| Violations allowed | No more than **4 pods** outside those thresholds; pods requesting ≤1 CPU and ≤2 GB are exempt |
| GPUs per workload | Up to 8 per Job pod (one node); up to 2 per bare pod |
| Large requests | Plans using >50 GPUs must be presented to support first |
| Storage | Not archival; volumes not accessed for 6 months may be purged without notice |
| Fair share | **None**: thousands of Jobs block other users; use an Indexed Job with a `parallelism` cap or a work queue |

Namespaces with consistently underused requests can be banned. Check the
**Violations** page in the portal and the Grafana dashboards. Full list and
consequences: `references/policies.md`.

## GPUs

```yaml
        resources:
          requests: {cpu: "4", memory: 16Gi, nvidia.com/gpu: 1}
          limits:   {cpu: "4", memory: 16Gi, nvidia.com/gpu: 1}
```

Pick a model with node affinity on `nvidia.com/gpu.product`:

```yaml
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: nvidia.com/gpu.product
                operator: In
                values: ["NVIDIA-A10", "NVIDIA-L40"]
```

List the products available now: `kubectl get nodes -L nvidia.com/gpu.product`
(live view: https://nrp.ai/viz/resources/).

Some GPUs are **special resources** and are not reachable with `nvidia.com/gpu`:

| GPU | Resource key | Access |
|---|---|---|
| A100 | `nvidia.com/a100` | Namespace quota is 0; request access through the A100 form |
| A100 MIG 1g.10gb | `nvidia.com/mig-small` | — |
| A40 | `nvidia.com/a40` | — |
| RTX A6000 | `nvidia.com/rtxa6000` | — |
| Quadro RTX 8000 | `nvidia.com/rtx8000` | — |
| H100 / H200 / GH200 | `nvidia.com/h100`, `nvidia.com/h200`, `nvidia.com/gh200` | Reserved; not user-requestable |
| Qualcomm Cloud AI 100 | `qualcomm.com/qaic` | One unit per SoC |

`priorityClassName: opportunistic` lets any namespace use quota-gated GPUs
without a reservation, **but the pod can be preempted at any time**. Use it
only for work that checkpoints or can be rerun.

More (CUDA version selection, GH200 arm64 toleration, `/dev/shm`): `references/gpus.md`.

## Storage

| Need | Use |
|---|---|
| Environments, many small files, databases (one pod) | Block PVC, `ReadWriteOnce`: `rook-ceph-block` (default, US West), `rook-ceph-block-east`, `-central`, `-south-east`, `-pacific` |
| Large files, checkpoints, data shared by many pods | CephFS PVC, `ReadWriteMany`: `rook-cephfs` (US West), `rook-cephfs-east`, `-central`, `-south-east`, `-pacific` |
| Data exchange with your machine, very large datasets | Ceph S3: `https://s3-west.nrp-nautilus.io` (also `s3-central`, `s3-east`); keys from the portal (S3 Tokens) |

Put compute in the same region as its storage: node affinity on
`topology.kubernetes.io/region`. Details and in-cluster S3 endpoints:
`references/storage.md`.

## Managed LLM Endpoint

NRP hosts open-weights LLMs behind an OpenAI-compatible API, with no pod or
GPU of your own:

- Base URL: `https://ellm.nrp-nautilus.io/v1`
- Token: create one at https://nrp.ai/llmtoken. Keep it in an environment variable or a Kubernetes Secret, never in a manifest or command line.
- Models: `GET /v1/models`, or the docs page https://nrp.ai/documentation/userdocs/ai/llm-managed/models/.

## Templates

`assets/templates/` holds NRP-ready manifests (requests == limits, TTL, labels):

- `job-cpu.yaml` — CPU batch Job.
- `job-gpu.yaml` — single-GPU Job with product affinity and `/dev/shm`.
- `job-indexed.yaml` — parameter sweep with a `parallelism` cap.
- `job-git.yaml` — clone a repository in an init container, then run it.
- `pvc-cephfs.yaml` — shared CephFS volume.
- `pod-interactive.yaml` — debugging pod (6-hour limit).

## Rules

- Confirm the namespace (`kubectl config view --minify -o jsonpath='{..namespace}'`) before you create anything.
- Check a manifest with `kubectl apply --dry-run=server -f <file>` first. It runs the Gatekeeper policies without creating anything; fix any denial or `Warning:` it prints.
- Show the user every manifest and the resources it requests before you apply it. NRP is shared, and underused requests count against the namespace.
- Request only what the workload uses. Do not request a GPU for CPU work, or several GPUs unless one is near 100% utilized.
- Use Jobs with `ttlSecondsAfterFinished` and `activeDeadlineSeconds`. Delete interactive pods as soon as you are done.
- Do not add tolerations for reserved nodes (`nautilus.io/reservation`) unless the user's namespace is entitled to them.
- Never print or write credential values (S3 keys, LLM tokens, kubeconfig tokens). Use Secrets.

## Monitoring and Help

- Grafana: https://grafana.nrp-nautilus.io (namespace pod and GPU dashboards)
- Portal violations page: https://nrp.ai
- Docs: https://nrp.ai/documentation/
- Support (Matrix): https://nrp.ai/contact/

## Reference Files

- `references/access.md` — accounts, namespaces, kubeconfig, kubelogin troubleshooting.
- `references/policies.md` — the full usage policies and what happens when they are violated.
- `references/gpus.md` — GPU selection, special resources, opportunistic priority, CUDA labels.
- `references/storage.md` — storage classes by region, S3 endpoints and clients, data movement.
