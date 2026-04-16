# K3s Homelab

Ansible automation to provision a homelab from bare Ubuntu server to a running K3s cluster with ArgoCD. Covers Linux baseline hardening, K3s installation, and ArgoCD setup via Helm.

> **Multi-node support:** This repo supports both single-node and multi-node setups. The Linux baseline playbook configures all nodes, with port 6443 only opened on the master. The automated upgrade workflow is built around a single master node — for multi-node zero downtime upgrades see [docs/upgrading.md](docs/upgrading.md).

## Prerequisites

- A Linux server running a Debian-based distribution
- Another computer with Ansible installed and an SSH key pair

### Compatible distributions

The playbook uses `apt` and `ufw`, so it should work on other Debian-based distributions:

- Ubuntu 22.04 LTS / 24.04 LTS
- Debian 11 (Bullseye) / 12 (Bookworm)
- Linux Mint (Debian/Ubuntu-based editions)

It will **not** work out of the box on distributions that use a different package manager (e.g. Fedora, CentOS, Arch).

## Steps

### 1. Get the IP address of the Ubuntu server

On the Ubuntu server, run:

```bash
ip a
```

Look for the network interface that has an `inet` address on your local network (commonly `eth0`, `enp0s3`, or similar). The output will look something like this:

```
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:a1:b2:c3 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.100/24 brd 192.168.1.255 scope global dynamic enp0s3
       valid_lft 86400sec preferred_lft 86400sec
```

In this example the server's IP address is `192.168.1.100`. Note this down — you'll need it for the next steps.

### 2. Copy your SSH key to the server

From your local machine, copy your public key to the server so you can authenticate without a password.

**If you have an Ed25519 key:**

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@192.168.1.100
```

**If you have an RSA key:**

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub user@192.168.1.100
```

Replace `user` with your username on the server and `192.168.1.100` with the IP address you noted in step 1.

You'll be prompted for the server user's password. After this, you should be able to SSH in without a password:

```bash
ssh user@192.168.1.100
```

### 3. Clone this repo and initialise the K3s Ansible submodule

Clone this repo on your local machine:

```bash
git clone https://github.com/denisecodes/k3s-homelab
cd k3s-homelab
```

Then initialise the K3s Ansible submodule:

```bash
git submodule update --init --recursive
```

### 4. Update the inventory

Copy the inventory template and fill in your details:

```bash
cp linux/inventory/hosts.ini linux/inventory/hosts-local.ini
# edit hosts-local.ini with your real IPs, username, and timezone
```

The inventory is split into `[master]` and `[workers]` groups so the playbook can apply different rules to each (e.g. port 6443 is only opened on the master).

**Single node:**

```ini
[master]
192.168.1.100

[workers]

[homelab:children]
master
workers

[homelab:vars]
ansible_user=your-user
ansible_ssh_private_key_file=~/.ssh/id_ed25519
timezone=Europe/London
```

**Multi-node** — uncomment and add worker IPs under `[workers]`:

```ini
[master]
192.168.1.100

[workers]
192.168.1.101
192.168.1.102

[homelab:children]
master
workers

[homelab:vars]
ansible_user=your-user
ansible_ssh_private_key_file=~/.ssh/id_ed25519
timezone=Europe/London
```

Set `timezone` to your local timezone. You can find the correct string at [https://en.wikipedia.org/wiki/List_of_tz_database_time_zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones). If omitted, the playbook defaults to `UTC`.

### 5. Run the Ansible playbook

```bash
ansible-playbook -i linux/inventory/hosts-local.ini linux/baseline.yml --ask-become-pass
```

You will be prompted for the become (sudo) password of the remote user. The playbook will then:

- Update and upgrade system packages
- Install essential tools (curl, git, vim, ufw, fail2ban, unattended-upgrades)
- Enable UFW with a default-deny policy; allow SSH (22) on all nodes and K3s API (6443) on the master only
- Disable root SSH login
- Set the timezone to your configured timezone (defaults to `UTC` if not set)

### 6. Configure K3s

Copy the K3s config template and fill in your details:

```bash
cp k3s/k3s-config.yml k3s/k3s-config-local.yml
# edit k3s-config-local.yml with your real IPs, username, and token
```

**Single node (default):**

```yaml
---
k3s_cluster:
  children:
    server:
      hosts:
        <YOUR_MASTER_NODE_IP>:   # control plane (master) node
    agent:
      hosts: {}

  vars:
    ansible_user: <YOUR_UBUNTU_USERNAME>
    ansible_ssh_private_key_file: ~/.ssh/id_ed25519
    k3s_version: v1.33.10+k3s1
    token: "<GENERATE_A_SECURE_TOKEN_HERE>"
    api_endpoint: "{{ hostvars[groups['server'][0]]['ansible_host'] | default(groups['server'][0]) }}"
    extra_server_args: "--disable traefik"
```

**Multi-node (server + agents):**

If you have additional machines to use as agent (worker) nodes, add their IP addresses under `agent.hosts`:

```yaml
---
k3s_cluster:
  children:
    server:
      hosts:
        <YOUR_MASTER_NODE_IP>:   # control plane (master) node
    agent:
      hosts:
        <YOUR_WORKER_NODE_1_IP>:   # worker node 1
        <YOUR_WORKER_NODE_2_IP>:   # worker node 2

  vars:
    ansible_user: <YOUR_UBUNTU_USERNAME>
    ansible_ssh_private_key_file: ~/.ssh/id_ed25519
    k3s_version: v1.33.10+k3s1
    token: "<GENERATE_A_SECURE_TOKEN_HERE>"
    api_endpoint: "{{ hostvars[groups['server'][0]]['ansible_host'] | default(groups['server'][0]) }}"
    extra_server_args: "--disable traefik"
```

Each agent node must have your SSH key copied to it (repeat step 2 for each additional machine) and be accessible with the same `ansible_user`.

For both setups, replace the following placeholders:

- `<YOUR_MASTER_NODE_IP>` — the IP of your control plane (master) node from step 1
- `<YOUR_WORKER_NODE_1_IP>`, `<YOUR_WORKER_NODE_2_IP>`, etc. — the IPs of any additional worker nodes (multi-node only)
- `<YOUR_UBUNTU_USERNAME>` — your username on the server(s)
- `<GENERATE_A_SECURE_TOKEN_HERE>` — a secure random token (e.g. `openssl rand -hex 32`); this must be the same across all nodes

### 7. Install required Ansible collections

The K3s playbook requires the `ansible.posix` collection in addition to the ones listed in `requirements.yml`. Install all required collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

### 8. Install K3s

```bash
ansible-playbook -i k3s/k3s-config-local.yml k3s-ansible/playbooks/site.yml --ask-become-pass
```

You will be prompted for the become (sudo) password. This will install and configure K3s on your server.

The playbook automatically fetches the kubeconfig, updates the server address to your API endpoint, and merges it into `~/.kube/config` under the context name `k3s-ansible`.

To rename the context to something more meaningful (e.g. `k3s-homelab`), run:

```bash
kubectl config rename-context k3s-ansible k3s-homelab
```

### 9. Verify the cluster

Switch to the homelab context and check the nodes:

```bash
kubectl config use-context k3s-homelab
```

Or if you kept the default context name:

```bash
kubectl config use-context k3s-ansible
```

Then verify:

```bash
kubectl get nodes
```

You should see output similar to:

```
NAME         STATUS   ROLES                  AGE   VERSION
home-k8s     Ready    control-plane,master   10m   v1.33.10+k3s1
```

## Upgrading K3s

For full details on the upgrade process, including the automated GitHub Actions workflow and manual steps, see [docs/upgrading.md](docs/upgrading.md).

## Future improvements

- **Project structure:** Configuration files are now organised into subdirectories — `linux/` for the initial server setup, `k3s/` for K3s configuration, and `argocd/` for ArgoCD. See [docs/future-improvements.md](docs/future-improvements.md) for planned further improvements.

## Acknowledgements

This project uses [k3s-ansible](https://github.com/k3s-io/k3s-ansible) for automating the K3s installation. It is a community-maintained repository and not an official release from the K3s project, but it is actively maintained and widely used.
