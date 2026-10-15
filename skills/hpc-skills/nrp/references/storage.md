# Storage on NRP

Sources: https://nrp.ai/documentation/userdocs/storage/intro/ ·
https://nrp.ai/documentation/userdocs/storage/ceph/ ·
https://nrp.ai/documentation/userdocs/storage/ceph-s3/

NRP storage is **not archival**: a volume not accessed for 6 months can be
purged without notice. Keep the only copy of important results elsewhere.

---

## Which Filesystem

| Filesystem | Good for | Avoid for |
|---|---|---|
| Ceph block (RBD), `ReadWriteOnce` | conda/pip environments, compiling, databases, many small files | Sharing between pods |
| CephFS, `ReadWriteMany` | Large files, checkpoints, datasets read by many pods in parallel | conda/pip installs, many small files |
| Ceph S3 | Moving data to and from your machine; very large datasets; per-file limit 5 TiB | Tools that need a POSIX filesystem |
| Linstor (block) | Lowest latency, HA databases; < 10 TB | Large volumes |
| CVMFS | Read-only software and datasets from OSDF origins | Writing |

CephFS has a 16 TB per-file limit and performs best with parallel streams and
aggregated (not tiny) files.

## Storage Classes by Region

| Region | Block (`ReadWriteOnce`) | CephFS (`ReadWriteMany`) |
|---|---|---|
| US West | `rook-ceph-block` (cluster default) | `rook-cephfs` |
| US Central | `rook-ceph-block-central` | `rook-cephfs-central` |
| US East | `rook-ceph-block-east` | `rook-cephfs-east` |
| US South East | `rook-ceph-block-south-east` | `rook-cephfs-south-east` |
| Hawaii + Guam | `rook-ceph-block-pacific` | `rook-cephfs-pacific` |

Other classes belong to specific institutions' local pools (for example
`rook-cephfs-ucsd`, NVMe, for data that is deleted as soon as the running
computation ends). `kubectl get storageclass` shows the current list.

Run pods near their storage for better I/O:

```yaml
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: topology.kubernetes.io/region
                operator: In
                values: ["us-west"]
```

Region values: `kubectl get nodes -L topology.kubernetes.io/region,topology.kubernetes.io/zone`.

## PVC Example

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-<username>
spec:
  storageClassName: rook-cephfs
  accessModes: [ReadWriteMany]
  resources:
    requests:
      storage: 100Gi
```

Volumes created since December 2020 can be expanded by raising
`spec.resources.requests.storage`; older ones need an admin.

## Ceph S3

| Pool | From your machine (HTTPS) | From inside the cluster (HTTP) |
|---|---|---|
| West (default) | `https://s3-west.nrp-nautilus.io` | `http://rook-ceph-rgw-nautiluss3.rook` |
| Central | `https://s3-central.nrp-nautilus.io` | `http://rook-ceph-rgw-centrals3.rook-central` |
| East | `https://s3-east.nrp-nautilus.io` | `http://rook-ceph-rgw-easts3.rook-east` |

Credentials: portal → **User → S3 Tokens** (https://nrp.ai/s3token/).
Store them in `~/.aws/credentials` (a named profile) on your machine, and in a
Secret inside the cluster:

```bash
kubectl create secret generic s3-credentials \
  --from-file=AWS_ACCESS_KEY_ID=./key_id --from-file=AWS_SECRET_ACCESS_KEY=./secret_key
```

then expose it with `envFrom: [{secretRef: {name: s3-credentials}}]`.

Clients:

```bash
aws s3 ls s3://<bucket>/ --endpoint-url https://s3-west.nrp-nautilus.io --profile nrp
rclone copy ./results nrp-s3:<bucket>/results -P   # rclone remote of type s3, provider Ceph
```

The AWS CLI can fail on multipart uploads of files above ~80 MB against this
endpoint; use `rclone` or `s3cmd` for large files. Inside the cluster use the
HTTP endpoint for the same pool, which avoids the public network.

## Moving Results Off a Finished Job

The pod's own filesystem is gone after the pod is deleted. Have the Job write
to a PVC or S3. To read from a PVC, start a small helper pod that mounts it
and `kubectl cp` from there, then delete the helper.
