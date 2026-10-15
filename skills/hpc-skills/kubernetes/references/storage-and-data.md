# Storage and Data on Kubernetes

Where data lives, and how to move it between your machine and the cluster.

---

## Storage Options

| Kind | Lifetime | Shared between pods | Typical use |
|---|---|---|---|
| Container filesystem | Pod | No | Nothing you need to keep |
| `emptyDir` | Pod | Containers of one pod | Scratch, cloned code, unpacked inputs |
| `emptyDir` with `medium: Memory` | Pod | Containers of one pod | `/dev/shm`, very fast small scratch (counts against memory) |
| PVC `ReadWriteOnce` (block) | Until the PVC is deleted | One node at a time | Environments, databases, many small files |
| PVC `ReadWriteMany` (CephFS/NFS) | Until the PVC is deleted | Many pods and nodes | Shared datasets, checkpoints |
| Object storage (S3) | Independent of the cluster | Anyone with credentials | Inputs and results, data exchange with your machine |

Storage class names are site-specific: `kubectl get storageclass`.

---

## PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-<username>
spec:
  storageClassName: <class>       # from: kubectl get storageclass
  accessModes: [ReadWriteMany]    # or ReadWriteOnce for block storage
  resources:
    requests:
      storage: 50Gi
```

Mount it in a pod:

```yaml
    spec:
      containers:
      - name: main
        volumeMounts:
        - {name: data, mountPath: /data}
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: data-<username>
```

- `kubectl get pvc` shows `Pending` until the volume is provisioned, then `Bound`.
- A `ReadWriteOnce` volume blocks a second pod on another node (`Multi-Attach error` in events). Delete the first pod or use `ReadWriteMany`.
- Deleting a PVC usually deletes its data. Deleting the pods does not.
- Many classes allow expansion: edit `spec.resources.requests.storage` upward.

---

## Shared Memory

```yaml
    spec:
      containers:
      - name: main
        volumeMounts:
        - {name: dshm, mountPath: /dev/shm}
      volumes:
      - name: dshm
        emptyDir:
          medium: Memory
          sizeLimit: 8Gi          # counts against the container memory limit
```

Needed for PyTorch `DataLoader(num_workers>0)`, NCCL and many MPI codes.

---

## Moving Data

**Small files, to or from a running pod** (needs `tar` in the image):

```bash
kubectl cp ./input.csv <ns>/<pod>:/data/input.csv
kubectl cp <ns>/<pod>:/data/results ./results
```

Without `tar`: `kubectl exec <pod> -- cat /data/out.json > out.json`.

**Results from a finished Job**: the pod's filesystem is gone once the pod is
deleted. Write results to a PVC or to object storage, then read them with a
small helper pod that mounts the same PVC:

```bash
kubectl run fetch-<username> --rm -it --restart=Never --image=alpine:3.20 \
  --overrides='{"spec":{"containers":[{"name":"fetch","image":"alpine:3.20","command":["sh"],"stdin":true,"tty":true,"volumeMounts":[{"name":"d","mountPath":"/data"}],"resources":{"requests":{"cpu":"100m","memory":"128Mi"},"limits":{"cpu":"100m","memory":"128Mi"}}}],"volumes":[{"name":"d","persistentVolumeClaim":{"claimName":"data-<username>"}}]}}'
```

**Large data**: use object storage (S3 with `aws s3`, `rclone` or `s5cmd`)
from both your machine and the pod. Put the S3 credentials in a Secret and
expose them as environment variables.

---

## Container Images

The image replaces `module load` and conda environments.

- Prefer a pinned tag (`pytorch/pytorch:2.4.1-cuda12.1-cudnn9-runtime`) over `:latest`.
- `pip install` at pod start is fine for a few small packages. For anything
  larger, build an image so every pod starts the same way and starts fast.
- Private registries need an `imagePullSecrets` entry referencing a
  `docker-registry` Secret.
- The image architecture must match the node (`amd64` vs `arm64`).
