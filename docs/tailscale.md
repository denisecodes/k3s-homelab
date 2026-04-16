# Tailscale Setup

This document covers how to install and configure Tailscale on your homelab nodes using the Ansible playbooks in `linux/`.

Tailscale creates a secure WireGuard-based mesh VPN between your devices. For this homelab it serves two purposes:

- **Remote access** — reach your homelab nodes by Tailscale IP from anywhere without port forwarding
- **GitHub Actions** — allows CI/CD workflows to SSH into homelab nodes from the public internet via `tailscale/github-action`

## Prerequisites

- Linux baseline applied (see [README](../README.md))
- Tailscale account — sign up free at [https://tailscale.com](https://tailscale.com)

## 1. Install Tailscale on the nodes

Run the install playbook to add the Tailscale apt repository, install the package, and start the `tailscaled` service:

```bash
ansible-playbook -i linux/inventory/hosts-local.ini linux/tailscale-install.yml \
  --ask-become-pass
```

The playbook will output a login URL for each node. Open each URL in your browser and log in with your Tailscale account to register the node.

## 2. Install Tailscale on a second device

> **Important:** Tailscale requires at least 2 devices registered in your account before you can generate an auth key. This is a known limitation — see [tailscale/tailscale#16217](https://github.com/tailscale/tailscale/issues/16217).

Install Tailscale on your laptop or desktop and log in with the same Tailscale account:

- **macOS:** `brew install tailscale` or download from [https://tailscale.com/download/mac](https://tailscale.com/download/mac)
- **Windows:** [https://tailscale.com/download/windows](https://tailscale.com/download/windows)
- **Linux:** [https://tailscale.com/download/linux](https://tailscale.com/download/linux)

Once both devices are registered, your Tailscale admin console will be accessible at [https://login.tailscale.com/admin](https://login.tailscale.com/admin).

## 3. Generate an auth key

Go to [https://login.tailscale.com/admin/settings/keys](https://login.tailscale.com/admin/settings/keys) and create a new auth key with the following settings:

- **Reusable** — so the same key works across multiple nodes and GitHub Actions runs
- **Ephemeral** — GitHub Actions runners are automatically removed from your tailnet when the job finishes
- **Expiry** — 90 days is a sensible default for a homelab

Keep the key safe — you will add it to the Ansible vault and as a GitHub Actions secret.

## 4. Authenticate the nodes

Add your auth key to the vault:

```bash
cp linux/vault/secrets.yml linux/vault/secrets-local.yml
# edit secrets-local.yml and replace the placeholder with your real auth key
ansible-vault encrypt linux/vault/secrets-local.yml
```

Then run the auth playbook:

```bash
ansible-playbook -i linux/inventory/hosts-local.ini linux/tailscale-auth.yml \
  --ask-become-pass --ask-vault-pass
```

The playbook will:
- Authenticate each node to your Tailscale account using the auth key
- Open UFW to allow Tailscale traffic (`41641/udp` and the `tailscale0` interface)
- Print the Tailscale IP for each node at the end of the run

The output will look like this:

```
ok: [192.168.50.x] => {
    "msg": "Tailscale IP for 192.168.50.x: 100.64.0.1"
}
```

> The IP on the left (`192.168.50.x`) is your node's local network IP — this is just how Ansible identifies the host from your inventory. The IP on the right (`100.64.0.1`) is the Tailscale IP. **This is the one you need** — use it to update `MASTER_NODE_IP` in your GitHub Actions secrets.

## 5. Configure GitHub Actions secrets

Update the following secrets in your repository at [https://github.com/denisecodes/k3s-homelab/settings/secrets/actions](https://github.com/denisecodes/k3s-homelab/settings/secrets/actions):

| Secret | Value |
|--------|-------|
| `TAILSCALE_AUTHKEY` | The auth key generated in step 3 |
| `MASTER_NODE_IP` | Your master node's Tailscale IP (e.g. `100.x.x.x`) |
| `WORKER_NODE_IPS` | Your worker node Tailscale IPs, comma-separated (if applicable) |

## Verify

Check that your nodes are visible and connected in the Tailscale admin console:

```
https://login.tailscale.com/admin/machines
```

Or from the node directly:

```bash
tailscale status
```

## Configuration reference

| Variable | File | Description |
|----------|------|-------------|
| `tailscale_authkey` | `linux/vault/secrets-local.yml` | Pre-auth key for authenticating nodes to your tailnet |
