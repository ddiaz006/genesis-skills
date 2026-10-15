# GPUs on NRP

Source: https://nrp.ai/documentation/userdocs/running/gpu-pods/

---

## Requesting a GPU

```yaml
        resources:
          requests: {cpu: "4", memory: 16Gi, nvidia.com/gpu: 1}
          limits:   {cpu: "4", memory: 16Gi, nvidia.com/gpu: 1}
```

- Up to **8** GPUs per pod in a Job (one node); up to **2** in a bare pod.
- Requests for 4 or 8 GPUs automatically get the toleration for the nodes
  reserved for multi-GPU work; you do not add it yourself.
- Without a product affinity the pod lands on any free NVIDIA GPU, from
  consumer cards (GTX 1080 Ti, RTX 3090) up to A40/L40. Pin a model when the
  code needs memory, precision or speed it cannot get everywhere.

## Choosing a Model

```bash
kubectl get nodes -L nvidia.com/gpu.product                       # model per node
kubectl get nodes -l nvidia.com/gpu.product -L nvidia.com/gpu.memory \
  -L nvidia.com/cuda.runtime.major -L nvidia.com/cuda.runtime.minor
```

Live availability: https://nrp.ai/viz/resources/

Most common `nvidia.com/gpu.product` values (October 2026, by node count):
`NVIDIA-GeForce-RTX-3090`, `NVIDIA-A10`, `NVIDIA-GeForce-RTX-2080-Ti`,
`NVIDIA-L40`, `NVIDIA-GeForce-GTX-1080-Ti`, `NVIDIA-RTX-A6000`, `NVIDIA-L4`,
`NVIDIA-L40S`, `NVIDIA-GeForce-RTX-4090`, `NVIDIA-RTX-A5000`, `NVIDIA-A40`.
A100, H100 and H200 nodes also carry product labels
(`NVIDIA-A100-SXM4-80GB`, `NVIDIA-H100-80GB-HBM3`, ...) but are requested
through their special resource keys below.

```yaml
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: nvidia.com/gpu.product
                operator: In
                values: ["NVIDIA-A10", "NVIDIA-L40", "NVIDIA-RTX-A6000"]
```

Use `preferredDuringSchedulingIgnoredDuringExecution` with a `weight` for a
soft preference that falls back to any GPU. Use the `NotIn` operator to avoid
models that are too small.

## CUDA and Driver Version

```yaml
            - matchExpressions:
              - key: nvidia.com/cuda.driver.major
                operator: Gt
                values: ["535"]
```

Labels: `nvidia.com/cuda.driver.major`/`minor`, `nvidia.com/cuda.runtime.major`/`minor`.

## Special Resources

These GPUs are scheduled by their own resource name, not `nvidia.com/gpu`:

| GPU | Key | Access |
|---|---|---|
| NVIDIA A100 | `nvidia.com/a100` | Quota 0 by default; request through the A100 access form |
| A100 MIG 1g.10gb | `nvidia.com/mig-small` | — |
| NVIDIA A40 | `nvidia.com/a40` | — |
| NVIDIA RTX A6000 | `nvidia.com/rtxa6000` | — |
| NVIDIA Quadro RTX 8000 | `nvidia.com/rtx8000` | — |
| NVIDIA H100 | `nvidia.com/h100` | Reserved; not user-requestable |
| NVIDIA H200 | `nvidia.com/h200` | Reserved; not user-requestable |
| NVIDIA GH200 | `nvidia.com/gh200` | Reserved; not user-requestable |
| RTX PRO 6000 Blackwell | `nvidia.com/rtx6000bw` | Exclusive use |
| Qualcomm Cloud AI 100 Ultra | `qualcomm.com/qaic` | One unit = one SoC (~25B-parameter LLMs) |

```yaml
        resources:
          requests: {cpu: "8", memory: 64Gi, nvidia.com/a100: 1}
          limits:   {cpu: "8", memory: 64Gi, nvidia.com/a100: 1}
```

## Opportunistic Priority

Any namespace can run on quota-gated GPUs without a reservation:

```yaml
    spec:
      priorityClassName: opportunistic     # or opportunistic2
```

Quotas do not apply, **but the pod can be preempted at any moment**. Only use
it for work that checkpoints to a PVC or S3 and restarts cleanly (with a Job
`backoffLimit` large enough to absorb preemptions).

## Grace Hopper (GH200, arm64)

Use an arm64 image (for example `nvidia/cuda:12.4.1-cudnn-devel-ubuntu22.04`,
which is multi-arch) and add:

```yaml
      tolerations:
      - key: nautilus.io/arm64
        operator: Exists
        effect: NoSchedule
```

## Shared Memory

The default `/dev/shm` is 64 MB. For PyTorch data loaders and NCCL:

```yaml
        volumeMounts:
        - {name: dshm, mountPath: /dev/shm}
      volumes:
      - name: dshm
        emptyDir: {medium: Memory}       # sizeLimit optional; counts against memory
```

## Utilization

- Keep GPU utilization above 40% (ideally near 100%): namespace GPU dashboard on
  https://grafana.nrp-nautilus.io.
- A GPU stays reserved as long as the pod exists, even when idle. Delete pods
  when done; prefer Jobs that exit over pods that sleep.
- Check inside a running pod: `kubectl exec <pod> -- nvidia-smi`.
