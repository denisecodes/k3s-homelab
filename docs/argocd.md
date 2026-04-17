# ArgoCD Setup

This document covers how to install and configure ArgoCD on your k3s cluster using the Ansible playbook in `argocd/playbooks/argocd-setup.yml`.

ArgoCD is installed via Helm using the official `argo/argo-cd` chart. The playbook:

- Adds the Argo Helm repository
- Deploys ArgoCD into the `argocd` namespace
- Installs the `argocd` CLI, version-matched to the deployed chart
- Creates a dedicated user (configured via `argocd_user`) and disables the default `admin` account
- Registers the k3s-apps repository via SSH deploy key
- Deploys the `argocd-apps` Application that watches the k3s-apps repository for application manifests

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

The playbook reads `argocd/vault/secrets-local.yml` to set the password for your dedicated ArgoCD user.

First, set your desired username by editing the `argocd_user` variable at the top of `argocd/playbooks/argocd-setup.yml`:

```yaml
vars:
  argocd_user: "your-username"   # change this to whatever you want
```

Copy the template and fill in your real password:

```bash
cp argocd/vault/secrets.yml argocd/vault/secrets-local.yml
# edit secrets-local.yml — replace <YOUR_ARGOCD_PASSWORD> with a real strong password
```

It should look like this before encryption:

```yaml
argocd_user_password: "a-real-strong-password"
```

Once you have set a real password, encrypt it with ansible-vault:

```bash
ansible-vault encrypt argocd/vault/secrets-local.yml
```

You will be prompted to set a vault password. Keep this safe — you will need it every time you run the playbook.

> **Never commit `secrets-local.yml` unencrypted.** It is gitignored — the template `secrets.yml` (with the placeholder) is what is committed.

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

### Option 1: Via hostname (recommended)

If you've configured local DNS (see [DNS setup](dns.md)), you can access ArgoCD at:

```
http://argocd.home.lan
```

This requires:
- dnsmasq configured to resolve `*.home.lan` to Traefik's LoadBalancer IP
- Traefik IngressRoute deployed (see `apps/argocd/ingressroute.yaml` in k3s-apps repo)

### Option 2: Via NodePort (fallback)

ArgoCD is also exposed via **NodePort** on port `30080`. Open your browser and navigate to:

```
http://<node-ip>:30080
```

For example, if your node IP is `192.168.1.100`:

```
http://192.168.1.100:30080
```

### Login credentials

Log in with:

- **Username**: the value you set for `argocd_user` in the playbook vars
- **Password**: the value you set in `argocd/vault/secrets-local.yml`

> This address is only reachable on your local network — it is not accessible from the internet unless you explicitly configure port forwarding on your router.

The NodePort (`30080`) is configurable via `argocd_nodeport` in the playbook vars.

## 5. Verify the CLI works

The playbook installs the `argocd` CLI on the master node. To confirm it is working, SSH into the master node and run:

```bash
argocd version
```

### 5.1 Logging in to ArgoCD via CLI

Before you can run ArgoCD CLI commands like `argocd repo list` or `argocd app list`, you need to log in. The CLI needs to know where the ArgoCD server is and authenticate with it.

There are two ways to do this:

#### Option 1: Port-forward (recommended for quick checks)

This method uses `kubectl` port-forwarding to access the ArgoCD API server without needing to specify a server address:

```bash
argocd repo list --port-forward --port-forward-namespace argocd
```

You can use `--port-forward` with any `argocd` CLI command. This is the simplest method for one-off commands.

#### Option 2: Persistent login session

For repeated CLI usage, log in once and the session will be saved:

```bash
# Log in via the NodePort (replace with your node IP)
argocd login <YOUR_NODE_IP>:30080 --username myuser --plaintext
```

- Replace `<YOUR_NODE_IP>` with your actual node IP
- Replace `myuser` with the username you set in `argocd_user`
- `--plaintext` is required because the NodePort is HTTP, not HTTPS

You'll be prompted for your password. Once logged in, the session is saved to `~/.config/argocd/config` and future commands will use it automatically.

To verify your login:

```bash
argocd account get-user-info
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
| `argocd_repo_url` | `git@github.com:<YOUR_USERNAME>/<YOUR_APPS_REPO>.git` | SSH URL of the k3s-apps repo registered in ArgoCD |
| `argocd_repo_ssh_key` | *(vault)* | SSH private key ArgoCD uses to pull from the k3s-apps repo |

To pin a different chart version, update `argocd_chart_version` in `argocd/playbooks/argocd-setup.yml`. Check available versions at [ArtifactHub](https://artifacthub.io/packages/helm/argo/argo-cd).

## 6. Register the k3s-apps repository in ArgoCD

ArgoCD needs read access to the `k3s-apps` repo to watch and sync application manifests. This is done via an SSH deploy key — it never expires and is scoped to a single repository (read-only).

### 6.1 Generate the SSH key pair

Run this locally (no passphrase):

```bash
ssh-keygen -t ed25519 -C "argocd@k3s-homelab" -f ~/.ssh/argocd_k3s_apps -N ""
```

This produces two files:
- `~/.ssh/argocd_k3s_apps.pub` — the public key (added to GitHub)
- `~/.ssh/argocd_k3s_apps` — the private key (added to the vault)

### 6.2 Add the public key as a deploy key on GitHub

1. Go to `https://github.com/<YOUR_USERNAME>/<YOUR_APPS_REPO>` → **Settings → Deploy keys → Add deploy key**
2. Title: `argocd-k3s-homelab`
3. Paste the contents of `~/.ssh/argocd_k3s_apps.pub`
4. Leave **Allow write access** unchecked — ArgoCD only needs read access
5. Click **Add key**

### 6.3 Add the private key to the vault

Decrypt your local vault file:

```bash
ansible-vault decrypt argocd/vault/secrets-local.yml
```

Open `argocd/vault/secrets-local.yml` and add the private key using a YAML multiline literal block (the indentation and line breaks must be preserved exactly):

```yaml
argocd_repo_ssh_key: |
  -----BEGIN OPENSSH PRIVATE KEY-----
  <paste the contents of ~/.ssh/argocd_k3s_apps here>
  -----END OPENSSH PRIVATE KEY-----
```

You can get the full key contents with:

```bash
cat ~/.ssh/argocd_k3s_apps
```

Re-encrypt the vault once done:

```bash
ansible-vault encrypt argocd/vault/secrets-local.yml
```

> **Never commit `secrets-local.yml` unencrypted** and never commit the private key file directly.

### 6.4 Re-run the playbook

The repo registration is handled automatically by the playbook. Re-running it will apply the ArgoCD repository `Secret` to the cluster:

```bash
ansible-playbook -i linux/inventory/hosts-local.ini argocd/playbooks/argocd-setup.yml \
  --ask-become-pass \
  --ask-vault-pass
```

### 6.5 Verify the repo is registered

After re-running the playbook, verify that ArgoCD successfully registered the `k3s-apps` repository.

#### CLI verification

SSH into the master node. If you haven't already logged in to ArgoCD via the CLI, you'll need to authenticate first (see section 5.1 for login instructions).

You can verify the repo using port-forwarding without a persistent login:

```bash
argocd repo list --port-forward --port-forward-namespace argocd
```

Or if you've already logged in:

```bash
argocd repo list
```

You should see `git@github.com:<YOUR_USERNAME>/<YOUR_APPS_REPO>.git` with a `Successful` connection status.

#### UI verification (alternative)

If you prefer to verify via the web interface:

1. Open the ArgoCD UI in your browser: `http://<node-ip>:30080`
2. Log in with your username and password
3. Navigate to **Settings** (gear icon in the left sidebar) → **Repositories**
4. You should see `git@github.com:<YOUR_USERNAME>/<YOUR_APPS_REPO>.git` listed with a green **Connected** status

The UI method is useful if you're already working in the browser or if you encounter CLI authentication issues.

## Next steps

### GitOps app deployment via ArgoCD

With the playbook complete, ArgoCD is now watching the `k3s-apps` repository. The `argocd-apps` Application is automatically deployed and monitors the `apps/` directory for new application manifests.

To deploy a new application:

1. **Create an `Application` manifest** in the `k3s-apps/apps/` directory:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: my-app
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://my-helm-repo.example.com
       targetRevision: 1.0.0
       chart: my-app
     destination:
       server: https://kubernetes.default.svc
       namespace: my-app
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
   ```

2. **Commit and push** to the k3s-apps repository — ArgoCD will automatically detect and sync the new application.
