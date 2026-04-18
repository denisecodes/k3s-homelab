# Local DNS Setup

This document covers how to set up local DNS so homelab services are reachable by hostname (e.g. `gitea.home.lan`) instead of IP and port.

## How it works

[dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) runs on the homelab server and provides wildcard DNS resolution:

- Any `*.denise.home` query resolves to the **Traefik LoadBalancer IP** (`<TRAEFIK_LOADBALANCER_IP>`)
- Traefik routes traffic to the appropriate service based on the hostname
- All other DNS queries are forwarded to upstream resolvers (Cloudflare `1.1.1.1` and Google `8.8.8.8`)
- The router's DHCP hands out the server's IP as the DNS server for all LAN clients

This means new apps automatically get DNS — no per-app DNS entries needed. Simply create a Traefik IngressRoute with the desired hostname.

## Prerequisites

- Linux baseline applied (see [README](../README.md))
- Server has a static IP or DHCP reservation on your router

## 1. Install dnsmasq

```bash
ansible-playbook -i linux/inventory/hosts-local.ini linux/playbooks/dnsmasq-setup.yml \
  --ask-become-pass
```

The playbook will:
- Install dnsmasq
- Disable systemd-resolved (frees port 53)
- Configure wildcard DNS for `*.denise.home`
- Set upstream DNS forwarders (Cloudflare + Google)
- Open UFW ports 53/tcp and 53/udp
- Verify DNS resolution is working

## 2. Configure your router

Update your router's DHCP settings to hand out the server's IP as the DNS server for all LAN clients.

### ASUS RT-AX59U (via app)

1. Open the **ASUS Router** app on your phone
2. Go to **Settings > LAN > DHCP Server**
3. Under **DNS and WINS Server Setting**, set **DNS Server 1** to your server's LAN IP (e.g. `<YOUR_SERVER_IP>`)
4. Optionally set **DNS Server 2** to `1.1.1.1` as a fallback
5. Tap **Apply**

### ASUS RT-AX59U (via web panel)

1. Open `http://<YOUR_ROUTER_IP>` in your browser
2. Log in with your router admin credentials
3. Go to **LAN** in the left sidebar, then the **DHCP Server** tab
4. Under **DNS and WINS Server Setting**, set **DNS Server 1** to your server's LAN IP
5. Optionally set **DNS Server 2** to `1.1.1.1`
6. Click **Apply**

### Other routers

Look for **DHCP Server** settings in your router's admin panel. Set the **DNS Server** field to your homelab server's LAN IP.

## 3. Renew DHCP on your devices

After updating the router, devices need to renew their DHCP lease to pick up the new DNS server:

- **macOS:** Toggle Wi-Fi off/on, or run `sudo ipconfig set en0 DHCP`
- **Linux:** `sudo dhclient -r && sudo dhclient`
- **Windows:** `ipconfig /release && ipconfig /renew`
- **iOS:** Settings > Wi-Fi > tap your network > Renew Lease

## 4. Verify from a client device

From any device on your LAN:

```bash
nslookup whoami.denise.home
```

This should return the Traefik LoadBalancer IP (`<TRAEFIK_LOADBALANCER_IP>`). If it doesn't, check that:
- dnsmasq is running on the server: `sudo systemctl status dnsmasq`
- The router DHCP is handing out the correct DNS: check your device's DNS settings
- UFW is allowing port 53: `sudo ufw status`

## 5. Access services via hostname

Services with Traefik IngressRoutes can be accessed via their configured hostnames:

- **ArgoCD**: http://argocd.denise.home
- **Longhorn**: http://longhorn.denise.home
- **Traefik Dashboard**: http://traefik.denise.home

To add a new service, create a Traefik IngressRoute in the k3s-apps repository. See examples in `apps/traefik/ingressroutes/` or the [IngressRoutes README](https://github.com/denisecodes/k3s-apps/blob/main/apps/traefik/ingressroutes/README.md).

## Configuration

Key variables in `linux/playbooks/dnsmasq-setup.yml`:

| Variable | Default | Description |
|---|---|---|
| `homelab_domain` | `denise.home` | Wildcard domain for local services |
| `traefik_ip` | `<TRAEFIK_LOADBALANCER_IP>` | Traefik LoadBalancer IP (all `*.denise.home` traffic routes here) |
| `upstream_dns_1` | `1.1.1.1` | Primary upstream DNS (Cloudflare) |
| `upstream_dns_2` | `8.8.8.8` | Secondary upstream DNS (Google) |

## Troubleshooting

**dnsmasq won't start (port 53 in use):**
```bash
sudo ss -tlnp | grep :53
```
If systemd-resolved is still running, stop it: `sudo systemctl stop systemd-resolved`

**DNS works on the server but not from other devices:**
Check that UFW allows port 53 and that your router DHCP is pointing to the server IP.

**DNS queries are slow:**
Check the dnsmasq log: `sudo tail -f /var/log/dnsmasq.log`. Ensure upstream DNS servers are reachable: `dig google.com @1.1.1.1`
