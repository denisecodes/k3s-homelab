# K3s Upgrade Process

This document explains how upgrades are managed in this repo — both manually and via the automated GitHub Actions workflow.

## Upgrade strategy

It is recommended to stay **two minor versions behind** the latest K3s release for stability. For example, if the latest release is v1.35, the recommended version to run is v1.33.

K3s only supports upgrading **one minor version at a time**. You cannot skip versions. For example, to go from v1.31 to v1.33 you must run:

```
v1.31 → v1.32 → v1.33
```

Each step requires updating `k3s/k3s-config.yml` and running the upgrade playbook separately.

---

## Option 1: Automated upgrades via GitHub Actions

> **Downtime warning:** The automated workflow does **not** drain or uncordon nodes. Each node will experience a brief interruption when K3s restarts during the upgrade. If you need zero downtime, skip to [Option 3: Zero downtime upgrades (manual)](#zero-downtime-upgrades) instead.

Two workflows handle the automated upgrade process:

### How it works

1. **`check-k3s-upgrade.yml`** runs on the **1st of every month** at 9am UTC. It:
   - Fetches all stable K3s releases from the GitHub API
   - Determines the recommended version (2 minor versions behind the latest)
   - Compares it against the version currently set in `k3s/k3s-config.yml`
   - If an upgrade is available and no open upgrade issue already exists, it creates a GitHub issue:
     > **K3s upgrade available: v1.31.x+k3s1 → v1.33.x+k3s1**
   - The issue includes the recommended upgrade path and two action commands

2. **`upgrade-k3s.yml`** triggers when you comment on the issue. It handles two commands:

| Comment | Action |
|---|---|
| `/upgrade` | Runs the upgrade playbook, commits the updated `k3s/k3s-config.yml`, and closes the issue |
| `/reject` | Closes the issue without making any changes |

### Required GitHub secrets

Before the automated workflow can connect to your server, you need to add secrets to your repository under **Settings → Secrets and variables → Actions**:

| Secret | Required | Description |
|---|---|---|
| `SSH_PRIVATE_KEY` | Yes | The contents of your private SSH key (e.g. `~/.ssh/id_ed25519`) |
| `MASTER_NODE_IP` | Yes | The IP address of your control plane (master) node (e.g. `192.168.1.100`) |
| `ANSIBLE_USER` | Yes | The username Ansible will SSH in as (e.g. `your-user`) |
| `K3S_TOKEN` | Yes | The cluster token set during installation (from `k3s/k3s-config.yml`) |
| `WORKER_NODE_IPS` | Multi-node only | Comma-separated IPs of your worker nodes (e.g. `192.168.1.101,192.168.1.102`) |

If `WORKER_NODE_IPS` is not set, the workflow behaves as single-node and only upgrades the master node.

The workflow generates a temporary runtime config from these secrets at run time. Your real IP addresses, username, and token are never stored in the repository.

### Multi-node caveat

When `WORKER_NODE_IPS` is set, the workflow supports multi-node clusters. The `upgrade.yml` playbook will:

- Upgrade master nodes **one at a time** (to avoid etcd instability)
- Upgrade all agent nodes **in parallel**

However, the workflow **does not drain or uncordon nodes**. Each node will experience a brief interruption when K3s restarts. If you need zero downtime, use the manual steps described in the [Zero downtime upgrades](#zero-downtime-upgrades) section instead.

### Example flow

1. On the 1st of the month, the check workflow runs
2. It finds you are on v1.31.x and recommends v1.33.x
3. It opens a GitHub issue noting the upgrade path: v1.31 → v1.32 → v1.33
4. You read the issue and comment `/upgrade`
5. The upgrade workflow runs, upgrades the server to v1.32.x, commits the change, and closes the issue
6. Next month, the check workflow opens a new issue for v1.32 → v1.33
7. You comment `/upgrade` again to complete the upgrade

> **Note:** Because you can only upgrade one minor version at a time, the workflow will always target the **next** minor version in the path, not the final target. Each step requires a separate issue and `/upgrade` comment.

---

## Option 2: Manual upgrades

If you prefer to upgrade manually without GitHub Actions:

### 1. Edit `k3s/k3s-config.yml`

Update the `k3s_version` to the next minor version:

```yaml
k3s_version: v1.33.10+k3s1
```

You can find available versions on the [K3s releases page](https://github.com/k3s-io/k3s/releases).

### 2. Run the upgrade playbook

```bash
ansible-playbook -i k3s-config.yml k3s-ansible/playbooks/upgrade.yml --ask-become-pass
```

### 3. Verify the upgrade

```bash
kubectl get nodes
```

The `VERSION` column should reflect the new K3s version. Repeat steps 1–3 for each minor version until you reach your target.

---

## Option 3: Zero downtime upgrades (manual)

**This repo is designed for a single-node cluster.** With a single master node, true zero downtime is not possible — when K3s restarts during an upgrade, the control plane and any running workloads on that node are briefly interrupted. If you have no active workloads at the time of upgrade, this is unlikely to be noticeable.

### Multi-node clusters

If you have configured agent nodes in your `k3s/k3s-config.yml`, for example:

```yaml
k3s_cluster:
  children:
    server:
      hosts:
        192.16.35.11:
    agent:
      hosts:
        192.16.35.12:
        192.16.35.13:
```

#### How the upgrade playbook handles multiple nodes

The `upgrade.yml` playbook handles server and agent nodes differently:

- **Master nodes** are upgraded **one at a time** (`serial: 1`). This is intentional — etcd can only handle one node transitioning at a time, so upgrading master nodes in parallel would risk cluster instability.
- **Agent nodes** are upgraded **all in parallel**. Agents don't run etcd so there is no such constraint.

This means running the playbook as-is will handle the ordering correctly. However, the playbook does **not** drain or uncordon nodes — workloads on a node will be briefly interrupted when K3s restarts during the upgrade.

#### Zero downtime: manual drain/uncordon

If you need zero downtime, you must drain each node before upgrading it and uncordon it afterwards. The safe order is: **master node first, then each agent**.

**Master node:**

```bash
kubectl drain <master-node-name> --ignore-daemonsets --delete-emptydir-data
ansible-playbook -i k3s/k3s-config.yml k3s-ansible/playbooks/upgrade.yml --ask-become-pass --limit <master-ip>
kubectl uncordon <master-node-name>
kubectl get nodes
```

**Each agent node (one at a time):**

```bash
kubectl drain <agent-node-name> --ignore-daemonsets --delete-emptydir-data
ansible-playbook -i k3s/k3s-config.yml k3s-ansible/playbooks/upgrade.yml --ask-become-pass --limit <agent-ip>
kubectl uncordon <agent-node-name>
kubectl get nodes
```

Only move to the next agent once the previous one is back to `Ready`.

> **Note:** This repo's automated GitHub Actions upgrade workflow is configured for a single master node and does not handle the multi-node drain/uncordon sequence. For multi-node clusters, the manual steps above are recommended.

> **Planned improvement:** A dedicated Ansible playbook (`k3s/upgrade-zero-downtime.yml`) to automate the drain/uncordon sequence above is planned. See [docs/future-improvements.md](future-improvements.md) for details.
