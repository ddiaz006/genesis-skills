# Troubleshooting Kubernetes Workloads

Work from the outside in: the Job, then its pods, then the containers.

---

## The Debugging Trio

```bash
kubectl describe pod <pod>                                          # state, scheduling, last events
kubectl get events --sort-by=.metadata.creationTimestamp | tail -20 # namespace timeline
kubectl logs <pod> [-c <container>] --previous                      # output of the crashed instance
```

For a Job:

```bash
kubectl describe job <job>                 # conditions: Complete, Failed, reason
kubectl get pods -l job-name=<job> -o wide # every attempt, node, restarts
```

---

## Pod Stuck in Pending

Read the `Events` section of `kubectl describe pod`.

| Event message | Meaning | Fix |
|---|---|---|
| `Insufficient cpu` / `Insufficient memory` | No node has that much free | Lower requests or wait |
| `Insufficient nvidia.com/gpu` | No free GPU of that kind | Wait, request fewer, or relax the GPU type |
| `didn't match Pod's node affinity/selector` | Affinity excludes every node with room | Check label values: `kubectl get nodes -L <key>` |
| `had untolerated taint` | Node is reserved | Add the toleration only if you are allowed to use those nodes |
| `exceeded quota` | Namespace ResourceQuota reached | `kubectl describe resourcequota`; delete other workloads or ask for quota |
| `persistentvolumeclaim … not found` / `unbound` | PVC missing or still provisioning | `kubectl get pvc` |
| `Multi-Attach error` | A `ReadWriteOnce` volume is in use on another node | Stop the other pod |

`Pending` with `ContainerCreating` means it is scheduled and still pulling the
image or mounting volumes; large images can take minutes.

---

## Pod Fails or Restarts

| State / reason | Meaning | Fix |
|---|---|---|
| `ErrImagePull`, `ImagePullBackOff` | Image name, tag or registry auth is wrong | Fix the image; add `imagePullSecrets` |
| `CrashLoopBackOff` | Container keeps exiting | `kubectl logs --previous` |
| `OOMKilled`, exit code 137 | Exceeded memory limit | Raise memory, reduce batch size |
| Exit code 1 / 2 | Application error | Logs |
| Exit code 126 / 127 | Command not executable / not found in the image | Check `command` and the image |
| Exit code 143 | Terminated (SIGTERM): deleted, preempted, or deadline | `kubectl describe`; check `activeDeadlineSeconds` |
| `Evicted` | Node ran out of memory or disk | Request realistic memory; use a volume instead of the container filesystem |
| `CreateContainerConfigError` | Missing ConfigMap/Secret key | `kubectl describe pod`; create the object |

---

## Apply Rejected

```
Error from server (Forbidden): admission webhook "..." denied the request: ...
```

A cluster policy (often Gatekeeper or Kyverno) rejected the manifest. The
message names the rule: missing requests or limits, limits too far above
requests, a forbidden image, a GPU in a Deployment. Fix the manifest; do not
retry unchanged.

```
Error from server (Forbidden): jobs.batch is forbidden: User "..." cannot create resource ...
```

RBAC: wrong namespace, or no role there. `kubectl auth can-i create jobs -n <ns>`.

---

## Job Never Finishes

- The command does not exit (a server, `sleep`, waiting on input). Fix the command.
- A retry loop: `backoffLimit` high and the pod keeps failing. Check `kubectl get pods -l job-name=<job>`.
- `parallelism` too low for `completions`. `kubectl describe job` shows active, succeeded, failed counts.

Set `activeDeadlineSeconds` so a stuck Job ends on its own.

---

## Authentication

- `Unauthorized` or a browser login prompt on every command: the OIDC token
  expired. Most kubeconfigs refresh it on the next command; follow the site's
  instructions to clear the token cache if it keeps failing.
- `kubectl config get-contexts` shows which cluster and namespace you are using.
