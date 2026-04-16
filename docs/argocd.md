# ArgoCD Setup

This document covers how to install and configure ArgoCD on your k3s cluster using the Ansible playbook in `argocd/playbooks/argocd-setup.yml`.

ArgoCD is installed via Helm using the official `argo/argo-cd` chart. The playbook:

- Installs Helm on the master node
- Adds the Argo Helm repository
- Deploys ArgoCD into the `argocd` namespace
- Installs the `argocd` CLI, version-matched to the deployed chart
- Creates a dedicated `homelab-admin` user and disables the default `admin` account

## Prerequisites

- K3s cluster already running (see [README](../README.md))
- [ansible-vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html) installed locally
- The `kubernetes.core` Ansible collection installed (see below)

## 1. Install the required Ansible collection

```bash
ansible-galaxy collection install -r argocd/requirements.yml
```

## 2. Create and encrypt the vault secrets file

The playbook reads `argocd/vault/secrets.yml` to set the password for the `homelab-admin` user. This file must exist **before** encrypting it.

The file is already scaffolded in the repo with a placeholder value. Edit it first:

```bash
# Open the file and replace the placeholder with a real strong password
nano argocd/vault/secrets.yml
```

It should look like this (before encryption):

```yaml
argocd_user_password: "YourStrongPasswordHere"
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

ArgoCD is configured to run in **insecure mode** (no pod-level TLS). Access it by port-forwarding from your local machine:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

Then open [http://localhost:8080](http://localhost:8080) in your browser and log in with:

- **Username**: `homelab-admin`
- **Password**: the value you set in `argocd/vault/secrets.yml`

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
| `argocd_chart_version` | `7.8.26` | Helm chart version to install |
| `argocd_namespace` | `argocd` | Kubernetes namespace |
| `argocd_user` | `homelab-admin` | Dedicated ArgoCD user (default `admin` is disabled) |
| `argocd_user_password` | *(vault)* | Password for `homelab-admin`, read from `argocd/vault/secrets.yml` |

To pin a different chart version, update `argocd_chart_version` in `argocd/playbooks/argocd-setup.yml`. Check available versions at [ArtifactHub](https://artifacthub.io/packages/helm/argo/argo-cd).
