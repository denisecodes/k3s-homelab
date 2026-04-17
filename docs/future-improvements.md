# Future Improvements

This document tracks planned improvements that have been intentionally deferred to keep the project working and avoid breaking existing paths or workflows.

## Testing

### Code review workflow

Add a GitHub Actions workflow (`.github/workflows/code-review.yml`) that runs on every pull request targeting `main` and covers everything that ansible-lint does not. Enforce it as a required status check in branch protection alongside the ansible-lint and Molecule jobs.

- Trigger: `pull_request` targeting `main` (opened, synchronised, reopened)
- Scope: only scan files changed in the PR to keep the workflow fast

**YAML and Markdown formatting** (ansible-lint only covers playbook semantics, not formatting):
- `yamllint` — consistent indentation, quoting, and line length across all `.yml` files
- Markdown linting — consistent heading structure, no broken links in docs

**Secret scanning** (ansible-lint has no visibility of secret content):
- `gitleaks` or `trufflesecurity/trufflehog` — scan every changed file for accidentally committed secrets (tokens, passwords, private keys, IPs)
- Verify that `argocd/vault/secrets.yml` is ansible-vault ciphertext and not plaintext before merge

**Inline fix suggestions:**
- Use `reviewdog` as the reporter for `yamllint` and Markdown lint violations — this posts inline comments directly on the changed lines in the PR diff rather than just failing the job, so you can see exactly what needs fixing without reading raw logs
- Where the fix is mechanical (e.g. indentation, trailing whitespace), `reviewdog` can post a GitHub suggestion block that you can accept and commit with a single click from the PR review UI, without having to edit the file locally and push again

**Dependency and image vulnerability scanning:**
- `renovate` or `dependabot` — automatically open PRs when the ArgoCD Helm chart (`argocd_chart_version` in `argocd/playbooks/argocd-setup.yml`) or the `kubernetes.core` collection version in `requirements.yml` have newer versions available
- `trivy` — scan the ArgoCD container image referenced in the Helm chart for known CVEs; fail the workflow if high or critical vulnerabilities are found
- Flag when the pinned `k3s_version` in `k3s/k3s-config.yml` is more than 2 minor versions behind the latest stable release (reuse the same logic as the monthly upgrade check workflow)

### ansible-lint

Add a GitHub Actions workflow (`.github/workflows/lint.yml`) that runs `ansible-lint` on every pull request targeting `main`. This catches syntax errors, deprecated module usage, and best practice violations before the PR can be merged. Enforce it as a required status check in branch protection.

Playbooks to lint:
- `linux/playbooks/server-setup.yml`
- `linux/playbooks/dnsmasq-setup.yml`
- `traefik/playbooks/traefik-setup.yml`
- `k3s/k3s-config.yml` (inventory/vars file — skip if not a playbook)
- `argocd/playbooks/argocd-setup.yml`
- `k3s-ansible/playbooks/site.yml` and `k3s-ansible/playbooks/upgrade.yml` (submodule — may need to pin lint rules)

### Molecule tests

Add Molecule scenarios that run on every pull request targeting `main`, alongside ansible-lint, as required status checks before merging. Use [Molecule](https://ansible.readthedocs.io/projects/molecule/) with the Docker driver to test each playbook in isolation. Each scenario spins up a container, runs the playbook, and asserts expected state using `ansible.builtin.assert` or `testinfra`.

#### `linux/playbooks/server-setup.yml`

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

## Zero downtime upgrade playbook

Add a dedicated playbook (`k3s/upgrade-zero-downtime.yml`) to automate the manual drain/uncordon sequence described in [docs/upgrading.md](upgrading.md) Option 3. Only relevant for multi-node clusters.

The playbook should:
- Loop over nodes in the correct order: master first, then each agent one at a time
- `kubectl drain` the node (`--ignore-daemonsets --delete-emptydir-data`) before upgrading
- Run `k3s-ansible/playbooks/upgrade.yml` scoped to that node via `--limit`
- `kubectl uncordon` the node and wait for it to return to `Ready` before moving to the next
- Fail fast if a node does not return to `Ready` within a timeout, rather than silently continuing

Once the playbook exists, the `upgrade-k3s.yml` GitHub Actions workflow can be extended with an optional input (e.g. `zero_downtime: true`) to run this playbook instead of the standard upgrade.

## Safe package updates via GitHub Actions

Add a scheduled GitHub Actions workflow (e.g. monthly) to run `apt update && apt upgrade` on the homelab servers via Ansible. This keeps packages up to date without requiring manual SSH access.

- Trigger: `schedule` (e.g. first Sunday of month) + `workflow_dispatch` for manual runs
- Playbook: reuse `linux/playbooks/server-setup.yml` or extract a dedicated `linux/playbooks/update-packages.yml`
- The workflow should open a GitHub issue or send a notification on failure
- Consider a `--check` dry-run step first to show what would be upgraded before applying

## GitHub Actions workflows for cluster provisioning

Add dedicated workflows to provision each layer of the cluster from scratch via GitHub Actions, mirroring what is currently done manually via the terminal. Each workflow should be a separate file and triggered independently (e.g. `workflow_dispatch`) so they can be run in order or re-run individually if a step fails.

- **`provision-linux.yml`** — runs `linux/playbooks/server-setup.yml` against the inventory; requires secrets `SSH_PRIVATE_KEY`, `MASTER_NODE_IP`, `ANSIBLE_USER`
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
├── traefik/            # Traefik ingress controller setup
├── k3s-ansible/        # Submodule
├── .github/workflows/  # GitHub Actions workflows
└── docs/               # Documentation
```
