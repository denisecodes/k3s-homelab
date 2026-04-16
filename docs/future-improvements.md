# Future Improvements

This document tracks planned improvements that have been intentionally deferred to keep the project working and avoid breaking existing paths or workflows.

## Testing

### ansible-lint

Add a GitHub Actions workflow (e.g. `.github/workflows/lint.yml`) that runs `ansible-lint` against all playbooks on every push and pull request. This catches syntax errors, deprecated module usage, and best practice violations before they reach the cluster.

Playbooks to lint:
- `linux/baseline.yml`
- `k3s/k3s-config.yml` (inventory/vars file — skip if not a playbook)
- `argocd/playbooks/argocd-setup.yml`
- `k3s-ansible/playbooks/site.yml` and `k3s-ansible/playbooks/upgrade.yml` (submodule — may need to pin lint rules)

### Molecule tests

Use [Molecule](https://ansible.readthedocs.io/projects/molecule/) with the Docker driver to test each playbook in isolation. Each scenario spins up a container, runs the playbook, and asserts expected state using `ansible.builtin.assert` or `testinfra`.

#### `linux/baseline.yml`

Scenario: `molecule/linux/`

Assert:
- `ufw` is enabled and default incoming policy is deny
- Port 22 and 6443 are allowed in UFW
- `fail2ban` is installed and running
- Root SSH login is disabled (`PermitRootLogin no` in sshd_config)
- Timezone is set correctly (e.g. `Europe/London`)
- `apt` packages are up to date

#### K3s install (`k3s-ansible/playbooks/site.yml`)

Scenario: `molecule/k3s-install/`

> Note: k3s-ansible already has its own CI — this scenario would test the integration with our inventory and configuration on top of it.

Assert:
- `k3s` service is running on the master node
- `kubectl get nodes` returns the expected master and worker nodes in `Ready` state
- kubeconfig is accessible at `/etc/rancher/k3s/k3s.yaml`

#### K3s upgrade (`k3s-ansible/playbooks/upgrade.yml`)

Scenario: `molecule/k3s-upgrade/`

Assert:
- K3s is running a pinned version pre-upgrade
- After running the upgrade playbook with a newer pinned version, `k3s --version` reports the new version
- All nodes return to `Ready` state after the upgrade

#### ArgoCD setup (`argocd/playbooks/argocd-setup.yml`)

Scenario: `molecule/argocd/`

> Note: requires a running K3s cluster or a `kind`/`k3d` cluster in the CI runner.

Assert:
- ArgoCD Helm release is present in the `argocd` namespace
- `argocd-server` deployment has at least 1 ready replica
- `argocd` CLI is installed at `/usr/local/bin/argocd`
- `homelab-admin` account exists (`argocd account list`)
- Default `admin` account is disabled
- `homelab-admin` can log in with the vault-provided password

## Safe package updates via GitHub Actions

Add a scheduled GitHub Actions workflow (e.g. monthly) to run `apt update && apt upgrade` on the homelab servers via Ansible. This keeps packages up to date without requiring manual SSH access.

- Trigger: `schedule` (e.g. first Sunday of month) + `workflow_dispatch` for manual runs
- Playbook: reuse `linux/baseline.yml` or extract a dedicated `linux/update-packages.yml`
- The workflow should open a GitHub issue or send a notification on failure
- Consider a `--check` dry-run step first to show what would be upgraded before applying

## GitHub Actions workflows for cluster provisioning

Add dedicated workflows to provision each layer of the cluster from scratch via GitHub Actions, mirroring what is currently done manually via the terminal. Each workflow should be a separate file and triggered independently (e.g. `workflow_dispatch`) so they can be run in order or re-run individually if a step fails.

- **`provision-linux.yml`** — runs `linux/baseline.yml` against the inventory; requires secrets `SSH_PRIVATE_KEY`, `MASTER_NODE_IP`, `ANSIBLE_USER`
- **`provision-k3s.yml`** — generates `runtime-k3s-config.yml` from secrets and runs `k3s-ansible/playbooks/site.yml` to install the cluster; requires `SSH_PRIVATE_KEY`, `MASTER_NODE_IP`, `WORKER_NODE_IPS`, `ANSIBLE_USER`, `K3S_TOKEN`
- **`provision-argocd.yml`** — installs the `kubernetes.core` collection, decrypts `argocd/vault/secrets.yml` using an `ANSIBLE_VAULT_PASSWORD` secret, and runs `argocd/playbooks/argocd-setup.yml`

> Note: the ArgoCD workflow will need the vault password stored as a GitHub Actions secret (`ANSIBLE_VAULT_PASSWORD`) and passed to `--vault-password-file` via a temp file, rather than `--ask-vault-pass`.

## ~~Project structure reorganisation~~ — Completed

Files are now organised into:

```
k3s-homelab/
├── linux/              # Initial Ubuntu server setup (baseline.yml, inventory/)
├── k3s/                # K3s configuration (k3s-config.yml)
├── argocd/             # ArgoCD setup
├── k3s-ansible/        # Submodule
├── .github/workflows/  # GitHub Actions workflows
└── docs/               # Documentation
```
