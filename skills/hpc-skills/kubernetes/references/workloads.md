# Kubernetes Workloads for Research Computing

Which object to use, and the Job patterns that replace common HPC idioms.

---

## Choosing a Workload

| Object | Use for | Ends? | Restarts on failure |
|---|---|---|---|
| `Job` | Batch work: simulations, training, preprocessing | Yes, when the command exits | Up to `backoffLimit` |
| `Pod` (no controller) | Short interactive debugging (`kubectl exec`) | When deleted | No |
| `Deployment` | Long-running services: APIs, dashboards, work-queue brokers | No | Yes, forever |
| `CronJob` | A Job on a schedule | Each run ends | Per Job |

Rule of thumb: if the work should finish, use a Job. Shared clusters often
delete bare Pods after a few hours and Deployments after a few weeks.

---

## Job Fields

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sim-<username>
  labels: {app: sim, owner: <username>}
spec:
  backoffLimit: 3                 # pod retries before the Job is Failed
  activeDeadlineSeconds: 14400    # wall-clock limit for the whole Job
  ttlSecondsAfterFinished: 86400  # auto-delete the Job and pods after it ends
  template:
    metadata:
      labels: {app: sim}
    spec:
      restartPolicy: Never        # Never: a new pod per retry (logs kept); OnFailure: restart in place
      containers:
      - name: main
        image: ghcr.io/org/sim:1.4.2   # pin a tag; avoid :latest for reproducibility
        workingDir: /work
        command: ["sh", "-c"]
        args:
        - |
          set -euo pipefail
          python simulate.py --steps 10000 --out /results/run.h5
        resources:
          requests: {cpu: "4", memory: 8Gi}
          limits:   {cpu: "4", memory: 8Gi}
```

- `sh -c` with `set -euo pipefail` makes a multi-step command fail on the first error.
- With `restartPolicy: Never` each retry is a new pod; `kubectl get pods -l job-name=<job>` shows them all.
- Labels let you select and delete groups: `kubectl delete jobs -l app=sim`.

---

## Indexed Jobs (Job Arrays)

```yaml
spec:
  completions: 50                 # number of array tasks
  parallelism: 5                  # max running at once
  completionMode: Indexed
  backoffLimitPerIndex: 2         # retries per index (Kubernetes >= 1.29)
  maxFailedIndexes: 5             # give up on the whole Job after this many indexes fail
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: main
        image: python:3.12-slim
        env:
        - name: N_TASKS
          value: "50"
        command: ["sh", "-c", "python sweep.py --index $JOB_COMPLETION_INDEX --n $N_TASKS"]
```

- `JOB_COMPLETION_INDEX` is set in every pod. Map it to a parameter set (a
  line of a file in a ConfigMap, an offset into a list).
- Pod hostnames are `<job>-<index>`, useful when one index must reach another
  through a headless Service.
- Slurm `%a` in output names becomes `$JOB_COMPLETION_INDEX` in your own output paths.

For dynamic or very large task lists, a work queue (Redis or RabbitMQ in a
Deployment, workers in a Job with `parallelism: N`) avoids creating thousands
of Job objects.

---

## Dependencies Between Steps

Kubernetes has no `--dependency=afterok`. Options:

1. **Sequential in the client**: `kubectl apply -f step1.yaml && kubectl wait --for=condition=complete job/step1 --timeout=2h && kubectl apply -f step2.yaml`.
2. **Init containers**: run setup steps before the main container in the same pod; each must exit 0.
3. **A workflow engine** (Argo Workflows, Kubeflow Pipelines, Parsl with a Kubernetes provider) when the cluster provides one.

---

## Init Containers: Fetch Code Before Running

```yaml
spec:
  template:
    spec:
      restartPolicy: Never
      initContainers:
      - name: clone
        image: alpine/git:2.45.2
        args: ["clone", "--depth=1", "https://github.com/org/repo.git", "/work/repo"]
        volumeMounts: [{name: work, mountPath: /work}]
        resources:
          requests: {cpu: 500m, memory: 512Mi}
          limits:   {cpu: 500m, memory: 512Mi}
      containers:
      - name: main
        image: python:3.12-slim
        workingDir: /work/repo
        command: ["sh", "-c", "pip install -q -r requirements.txt && python main.py"]
        volumeMounts: [{name: work, mountPath: /work}]
        resources:
          requests: {cpu: "2", memory: 4Gi}
          limits:   {cpu: "2", memory: 4Gi}
      volumes:
      - name: work
        emptyDir: {}
```

Logs of the init container: `kubectl logs <pod> -c clone`.

---

## Sidecars

A pod can run several containers that share `localhost` and volumes (for
example a training loop plus a checkpoint uploader). Use one pod only when
the containers are tightly coupled; otherwise use separate pods. On a
multi-container pod, `kubectl logs` needs `-c <container>`.

---

## Interactive Work

```bash
kubectl apply -f interactive-pod.yaml           # a Pod with a bounded sleep
kubectl wait --for=condition=Ready pod/dev-<username> --timeout=5m
kubectl exec -it dev-<username> -- bash
kubectl delete pod dev-<username>               # release the resources
```

Bound the sleep (`sleep 21600`) rather than `sleep infinity`, and delete the
pod when you are done. Many clusters forbid sleeping inside Jobs; check the
site skill.

---

## Services and Port-Forwarding

To reach a web UI or API running in a pod from your machine without exposing it:

```bash
kubectl port-forward pod/<pod> 8888:8888        # or deployment/<name>, svc/<name>
```

Public exposure needs a Service plus an Ingress; the ingress class and allowed
hostnames are site-specific.
