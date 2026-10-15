# NRP Access: Accounts, Namespaces, kubeconfig

Source: https://nrp.ai/documentation/userdocs/start/getting-started/

---

## Account

- Log in at https://nrp.ai through CILogon with your institutional identity.
  If your institution is not listed, Microsoft, Google or GitHub identities work.
  ORCID users must make their email visible.
- The first login creates the account. Before running anything, accept the
  Acceptable Use Policy and read the cluster policies.

## Namespaces

A namespace is the unit of access, quota and accountability (the HPC analogue
of a project allocation).

| Role | How |
|---|---|
| Student / member | A namespace admin adds you (they can invite you by email before your first login) |
| Faculty, staff, postdoc | Request a namespace at https://get-started.nrp-nautilus.io; you become its admin |

- Names: lowercase letters, digits and dashes. Existing names: https://nrp.ai/viz/namespaces/
- Your memberships: https://nrp.ai/namespaces (tab **Mine**). Admins manage members and subgroups there.
- Everyone in a namespace can read its Secrets and delete its pods. Use a
  separate namespace for credentials you do not want to share.

## kubectl and kubelogin

NRP's kubeconfig authenticates through OIDC with the `kubelogin` plugin. The
binary must be on the `PATH` as `kubectl-oidc_login`.

**macOS** (Homebrew):

```bash
brew install kubectl int128/kubelogin/kubelogin
```

**Linux** (release binary; set `ARCH` to `amd64` or `arm64`):

```bash
ARCH=amd64
VER="$(curl -fsSL https://api.github.com/repos/int128/kubelogin/releases/latest | jq -r .tag_name)"
curl -fsSL -o kubelogin.zip "https://github.com/int128/kubelogin/releases/download/${VER}/kubelogin_linux_${ARCH}.zip"
unzip -o kubelogin.zip kubelogin && rm kubelogin.zip
install -m 0755 kubelogin ~/.local/bin/kubectl-oidc_login && rm kubelogin
```

Check: `kubectl oidc-login --help`.

## kubeconfig

```bash
mkdir -p ~/.kube
curl -fsSL -o ~/.kube/nautilus.config https://nrp.ai/config
```

If `~/.kube/config` does not exist yet, save it there directly. Otherwise
merge it so other clusters are kept:

```bash
KUBECONFIG=~/.kube/config:~/.kube/nautilus.config kubectl config view --flatten > ~/.kube/config.merged \
  && mv ~/.kube/config.merged ~/.kube/config && chmod 600 ~/.kube/config
```

Then:

```bash
kubectl config use-context nautilus
kubectl get nodes                                         # first call opens CILogon in the browser
kubectl config set contexts.nautilus.namespace <namespace>
kubectl auth can-i create jobs                            # yes
```

## kubelogin Options

Add these to the `args` of the `oidc-login get-token` exec entry in the kubeconfig:

| Situation | Argument |
|---|---|
| No local browser (SSH session, container) | `--grant-type=device-code` and `--skip-open-browser` |
| Browser opens in the wrong place (WSL) | `--browser-command="<path to browser>"` |
| Callback port in use | `--listen-address=0.0.0.0:18000` |
| Token persistence | `--token-cache-storage=disk` or `--token-cache-storage=keyring` |

Tokens last about 30 minutes and renew automatically. After joining a
namespace (or if access looks stale), run `kubectl oidc-login clean`.

## Other Entry Points

- **JupyterHub**: https://jupyterhub-west.nrp-nautilus.io — notebooks without kubectl (5 GB home, stops ~1 hour after the browser disconnects).
- **Coder**: browser VS Code environments on the cluster.
- **GitLab**: https://gitlab.nrp-nautilus.io — repositories, CI, and a container registry for your images.
