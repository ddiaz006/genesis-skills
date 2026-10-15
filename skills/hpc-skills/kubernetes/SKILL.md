---
name: kubernetes
description: >-
  Kubernetes for batch and research computing with kubectl. Use when writing
  Job, Indexed Job (job array), Pod, PVC, ConfigMap or Secret manifests,
  setting CPU/memory/GPU requests and limits, monitoring with get, describe,
  logs and events, debugging Pending, CrashLoopBackOff, ImagePullBackOff or
  OOMKilled pods, or translating a Slurm/PBS workflow to Kubernetes.
compatibility: Requires kubectl in PATH and a kubeconfig with access to at least one namespace. No login node is needed; kubectl runs from the local machine.
metadata:
  version: "1.0"
  scheduler: kubernetes
allowed-tools: Bash(kubectl *) Read Write
---

# Kubernetes for Batch Computing

Kubernetes runs **containers** on a cluster. You never log in to a node: you
write a YAML manifest that declares what should run, submit it with `kubectl`
from your own machine, and Kubernetes keeps the cluster in that state.

For site rules (required labels, request/limit policies, GPU names, storage
classes) load the site skill too, for example `nrp` for the National Research
Platform.

## HPC to Kubernetes

| HPC (Slurm / PBS) | Kubernetes |
|---|---|
| Login node | Your local machine running `kubectl` |
| Account / project (`-A`) | Namespace (`-n <namespace>`) |
| Batch script (`sbatch`, `qsub`) | `Job` manifest (`kubectl apply -f job.yaml`) |
| Job array (`--array`, `-J`) | Indexed Job (`completionMode: Indexed`) |
| Interactive job (`salloc`, `qsub -I`) | A bare `Pod` + `kubectl exec -it` |
| `module load` / conda env | Container image |
| `--cpus-per-task`, `--mem` | `resources.requests` / `resources.limits` |
| `--gres=gpu:N` | `resources.limits: {nvidia.com/gpu: N}` |
| `--constraint`, partition | `nodeSelector` / `nodeAffinity`, tolerations |
| `$HOME`, `$SCRATCH` | PersistentVolumeClaim (PVC), `emptyDir` |
| `squeue -u $USER` | `kubectl get jobs,pods` |
| `scancel <id>` | `kubectl delete job <name>` |
| `sacct` / job output file | `kubectl describe`, `kubectl logs` |
| Walltime (`-t`) | `activeDeadlineSeconds` |
| Requeue on failure | `backoffLimit` |

There is no billing allocation and usually no queue priority. Most clusters
also have **no fair-share queue**, so a flood of Jobs can starve other users.
Prefer one Indexed Job with a `parallelism` cap over thousands of Jobs.

## Quick Command Reference

| Command | Purpose |
|---|---|
| `kubectl config current-context` | Which cluster you are talking to |
| `kubectl config set-context --current --namespace=<ns>` | Set the default namespace |
| `kubectl auth can-i create jobs -n <ns>` | Check permissions |
| `kubectl apply -f job.yaml` | Create or update resources |
| `kubectl get jobs,pods -n <ns>` | List jobs and pods |
| `kubectl get pods -l job-name=<job> -w` | Watch the pods of one Job |
| `kubectl describe pod <pod>` | Scheduling decisions, state, recent events |
| `kubectl logs <pod> [-c <container>] [-f] [--previous]` | Container output |
| `kubectl logs -l job-name=<job> --tail=50` | Output of every pod of a Job |
| `kubectl get events --sort-by=.metadata.creationTimestamp` | Namespace timeline |
| `kubectl exec -it <pod> -- bash` | Shell in a running container |
| `kubectl cp <ns>/<pod>:/path ./local` | Copy files out (needs `tar` in the image) |
| `kubectl wait --for=condition=complete job/<job> --timeout=1h` | Block until done |
| `kubectl delete job <job>` | Cancel a Job and its pods |
| `kubectl get nodes -L <label>` | Show a node label as a column |

Pod phases: `Pending` (not scheduled, or pulling the image), `Running`,
`Succeeded`, `Failed`. Job status: `Complete` or `Failed` once finished.

## Minimal Batch Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-<username>          # must be unique in the namespace
spec:
  backoffLimit: 2                 # retries after a failed pod
  activeDeadlineSeconds: 3600     # walltime for the whole Job
  ttlSecondsAfterFinished: 86400  # delete the Job and its pods a day after it ends
  template:
    spec:
      restartPolicy: Never        # Jobs need Never or OnFailure
      containers:
      - name: main
        image: python:3.12-slim
        command: ["python", "-c", "import platform; print('hello from', platform.node())"]
        resources:
          requests: {cpu: "1", memory: 2Gi}
          limits:   {cpu: "1", memory: 2Gi}
```

```bash
kubectl apply -f job.yaml
kubectl wait --for=condition=complete job/hello-<username> --timeout=10m
kubectl logs -l job-name=hello-<username>
```

`command` must end on its own. A Job whose command never exits holds its
resources forever.

## Resources

- **requests** decide scheduling: the pod only lands on a node with that much free.
- **limits** are enforced: over the memory limit the container is OOMKilled; over the CPU limit it is throttled.
- `requests == limits` gives the pod `Guaranteed` QoS and is the safe default. Many shared clusters reject pods without both, or with limits far above requests.
- Units: CPU in cores or millicores (`500m`), memory in `Mi`/`Gi`.
- GPUs go in both requests and limits as an integer: `nvidia.com/gpu: 1`. GPUs are not shared between pods.
- Containers get a 64 MiB `/dev/shm` by default. PyTorch data loaders need more; mount an `emptyDir` with `medium: Memory` at `/dev/shm`.

## Parameter Sweeps (Job Arrays)

An Indexed Job runs `completions` pods, each with a distinct
`JOB_COMPLETION_INDEX` (0 … completions-1), at most `parallelism` at a time.

```yaml
spec:
  completions: 100
  parallelism: 10
  completionMode: Indexed
  backoffLimitPerIndex: 1         # retry each index independently
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: main
        command: ["sh", "-c", "python run.py --task $JOB_COMPLETION_INDEX"]
```

Full template: `assets/templates/indexed-job.yaml`.

## Data

- **Container filesystem** is lost when the pod ends.
- **`emptyDir`** is scratch shared by the containers of one pod, deleted with the pod.
- **PVC** is persistent storage that outlives pods. `ReadWriteOnce` volumes mount on one node at a time; `ReadWriteMany` (CephFS, NFS) can be mounted by many pods.
- **Object storage (S3)** is usually best for inputs and results moved to and from your machine.
- Code from git: clone it in an `initContainer` into an `emptyDir`, then run it in the main container.

See `references/storage-and-data.md`.

## Configuration and Secrets

Put settings in a ConfigMap and credentials in a Secret, then expose them as
environment variables (`envFrom`, `secretKeyRef`) or files. Create Secrets
from files or literals with `kubectl create secret generic …`; never write
credential values into a manifest that is committed to git. Anyone who can
read Secrets in the namespace can read their values.

## Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| `Pending` for minutes | No node has the requested CPU/memory/GPU, or affinity/tolerations exclude all nodes | `kubectl describe pod` → `Events` (`FailedScheduling`, `Insufficient nvidia.com/gpu`) |
| `ImagePullBackOff` / `ErrImagePull` | Wrong image name/tag or private registry without `imagePullSecrets` | `kubectl describe pod` |
| `CrashLoopBackOff` | Container exits non-zero and is restarted | `kubectl logs <pod> --previous` |
| `OOMKilled` (exit 137) | Memory limit too low | `kubectl describe pod` → `Last State`; raise `memory` |
| `admission webhook … denied` on apply | Cluster policy (missing limits, ratio, forbidden field) | Read the message; fix the manifest |
| `Forbidden` | No RBAC permission in that namespace | `kubectl auth can-i …`; check `-n` |
| Job `Failed`, `BackoffLimitExceeded` | All retries failed | Logs of the failed pods |
| Job `DeadlineExceeded` | `activeDeadlineSeconds` reached | Raise it or split the work |

More: `references/troubleshooting.md`.

## Rules

- Always set `requests` and `limits` for CPU and memory, and `restartPolicy` for Jobs.
- Always pass the namespace (`-n`) or confirm the context's default before you create anything.
- Use a Job, not a bare Pod, for batch work: Jobs retry and record completion.
- Clean up: `ttlSecondsAfterFinished` on Jobs, and `kubectl delete` pods you no longer need. Requested GPUs stay reserved while the pod exists.
- Before you apply to a shared cluster, show the user the manifest and the resources it requests.

## Reference Files

- `references/workloads.md` — Pod vs Job vs Deployment, Indexed Jobs, init containers, sidecars.
- `references/storage-and-data.md` — PVCs, `emptyDir`, `/dev/shm`, moving data in and out.
- `references/troubleshooting.md` — diagnosing failed and stuck workloads.
- `assets/templates/` — `batch-job.yaml`, `gpu-job.yaml`, `indexed-job.yaml`, `interactive-pod.yaml`.
