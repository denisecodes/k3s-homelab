# ArgoCD Setup

This document covers how to install and configure ArgoCD on your k3s cluster using the Ansible playbook in `argocd/playbooks/argocd-setup.yml`.

ArgoCD is installed via Helm using the official `argo/argo-cd` chart. The playbook:

- Adds the Argo Helm repository
- Deploys ArgoCD into the `argocd` namespace
- Installs the `argocd` CLI, version-matched to the deployed chart
- Creates a dedicated user (configured via `argocd_user`) and disables the default `admin` account

## Prerequisites

- K3s cluster already running (see [README](../README.md))
- Helm installed on the cluster nodes — **complete [Helm setup](helm.md) first**
- [ansible-vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html) installed locally
- The `kubernetes.core` Ansible collection installed (see below)

## 1. Install the required Ansible collection

```bash
ansible-galaxy collection install -r argocd/requirements.yml
```

## 2. Create and encrypt the vault secrets file

The playbook reads `argocd/vault/secrets.yml` to set the password for your dedicated ArgoCD user. This file must exist **before** encrypting it.

First, set your desired username by editing the `argocd_user` variable at the top of `argocd/playbooks/argocd-setup.yml`:

```yaml
vars:
  argocd_user: "your-username"   # change this to whatever you want
```

The file is already scaffolded in the repo with a placeholder value. Edit it first:

```bash
# Open the file and replace the placeholder with a real strong password
vim argocd/vault/secrets.yml
```

It should look like this (before encryption):

```yaml
argocd_user_password: "YourStrongPasswordHere"  # replace with a real strong password
```

Once you have set a real password, encrypt the file with ansible-vault:

```bash
ansible-vault encrypt argocd/vault/secrets.yml
```

You will be prompted to set a vault password. Keep this safe — you will need it every time you run the playbook.

> **Never commit the unencrypted file.** The file is encrypted in-place; the ciphertext is safe to commit.

## 3. Run the playbook

From the repo root:

```bash
ansible-playbook -i linux/inventory/hosts.ini argocd/playbooks/argocd-setup.yml \
  --ask-become-pass \
  --ask-vault-pass
```

- `--ask-become-pass` — required for `sudo` on the remote host
- `--ask-vault-pass` — prompts for the vault password to decrypt `vault/secrets.yml`

## 4. Access the ArgoCD UI

ArgoCD is exposed via **NodePort** on port `30080`. Open your browser and navigate to:

```
http://<node-ip>:30080
```

For example, if your node IP is `192.168.50.113`:

```
http://192.168.50.113:30080
```

Log in with:

- **Username**: the value you set for `argocd_user` in the playbook vars
- **Password**: the value you set in `argocd/vault/secrets-local.yml`

> This address is only reachable on your local network. The `192.168.50.x` range is a private address — it is not accessible from the internet unless you explicitly configure port forwarding on your router.

The NodePort (`30080`) is configurable via `argocd_nodeport` in the playbook vars.

## 5. Verify the CLI works

The playbook installs the `argocd` CLI on the master node. To confirm it is working, SSH into the master node and run:

```bash
argocd version
```

## Uninstalling ArgoCD

Because ArgoCD is managed by Helm, it can be cleanly removed with:

```bash
helm uninstall argocd -n argocd
kubectl delete namespace argocd
```

## Configuration reference

| Variable | Default | Description |
|---|---|---|
| `argocd_chart_version` | `7.8.28` | Helm chart version to install |
| `argocd_namespace` | `argocd` | Kubernetes namespace |
| `argocd_nodeport` | `30080` | NodePort for the ArgoCD UI — access at `http://<node-ip>:<nodeport>` |
| `argocd_user` | *(your choice)* | Dedicated ArgoCD user — set this at the top of the playbook (default `admin` is disabled) |
| `argocd_user_password` | *(vault)* | Password for your dedicated user, read from `argocd/vault/secrets.yml` |

To pin a different chart version, update `argocd_chart_version` in `argocd/playbooks/argocd-setup.yml`. Check available versions at [ArtifactHub](https://artifacthub.io/packages/helm/argo/argo-cd).

## Next steps

### GitOps app deployment via ArgoCD

The natural next step is to have ArgoCD watch a Git repository and automatically deploy applications via Helm. The workflow looks like this:

1. **Create an app-of-apps repo** — a separate Git repository (e.g. `k3s-apps`) that contains Helm charts or ArgoCD `Application` manifests for each service you want to deploy.

2. **Register the repo in ArgoCD** — point ArgoCD at the repo so it can pull from it:
   ```bash
   argocd repo add https://github.com/your-username/k3s-apps --username your-user --password your-token
   ```

3. **Create an ArgoCD Application** — define what to deploy and where:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: my-app
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://github.com/your-username/k3s-apps
       targetRevision: HEAD
       path: charts/my-app
     destination:
       server: https://kubernetes.default.svc
       namespace: my-app
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
   ```

4. **Automate with Ansible** — this can be added to `argocd-setup.yml` using `kubernetes.core.k8s` to apply `Application` manifests, so the full cluster state is declared in code and reproduced by running the playbook.
