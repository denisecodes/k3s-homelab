# K3s Homelab - Baseline Server Setup

An Ansible playbook to bootstrap an Ubuntu server with essential packages, firewall rules, and security hardening for a K3s homelab.

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

### 3. Update the inventory

Edit `inventory/hosts.ini` and set the IP address, username, and SSH key path to match your setup:

```ini
[homelab]
192.168.1.100 ansible_user=your-user ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

### 4. Run the Ansible playbook

```bash
ansible-playbook -i inventory/hosts.ini baseline.yml --ask-become-pass
```

You will be prompted for the become (sudo) password of the remote user. The playbook will then:

- Update and upgrade system packages
- Install essential tools (curl, git, vim, ufw, fail2ban, unattended-upgrades)
- Enable UFW with a default-deny policy and allow SSH (22) and K3s API (6443)
- Disable root SSH login
- Set the timezone to Europe/London

### 5. Clone the K3s Ansible submodule

The K3s installation playbook is included as a Git submodule. After cloning this repo, initialise and fetch it:

```bash
git submodule update --init --recursive
```

### 6. Configure K3s

Edit `k3s-config.yml` and fill in your details:

```yaml
---
k3s_cluster:
  children:
    server:
      hosts:
        <YOUR_SERVER_IP>:
    agent:
      hosts: {}

  vars:
    ansible_user: <YOUR_UBUNTU_USERNAME>
    ansible_ssh_private_key_file: ~/.ssh/id_ed25519
    k3s_version: v1.31.0+k3s1
    token: "<GENERATE_A_SECURE_TOKEN_HERE>"
    api_endpoint: "{{ hostvars[groups['server'][0]]['ansible_host'] | default(groups['server'][0]) }}"
    extra_server_args: "--disable traefik"
```

Replace `<YOUR_SERVER_IP>` with the server IP from step 1, `<YOUR_UBUNTU_USERNAME>` with your username, and `<GENERATE_A_SECURE_TOKEN_HERE>` with a secure token (e.g. generated via `openssl rand -hex 32`).

### 7. Install K3s

```bash
ansible-playbook -i k3s-config.yml k3s-ansible/playbooks/site.yml --ask-become-pass
```

You will be prompted for the become (sudo) password. This will install and configure K3s on your server.

The playbook automatically fetches the kubeconfig, updates the server address to your API endpoint, and merges it into `~/.kube/config` under the context name `k3s-ansible`.

To rename the context to something more meaningful (e.g. `k3s-homelab`), run:

```bash
kubectl config rename-context k3s-ansible k3s-homelab
```

### 8. Verify the cluster

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
home-k8s     Ready    control-plane,master   10m   v1.31.0+k3s1
```

## Acknowledgements

This project uses [k3s-ansible](https://github.com/k3s-io/k3s-ansible) for automating the K3s installation. It is a community-maintained repository and not an official release from the K3s project, but it is actively maintained and widely used.
