# Local DNS Setup

This document covers how to set up local DNS so homelab services are reachable by hostname (e.g. `gitea.home.lan`) instead of IP and port.

## How it works

[dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) runs on the homelab server and provides wildcard DNS resolution:

- Any `*.<your-domain>.home` query resolves to the **Traefik LoadBalancer IP** (`<TRAEFIK_LOADBALANCER_IP>`)
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
- Configure wildcard DNS for `*.<your-domain>.home`
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
nslookup whoami.<your-domain>.home
```

This should return the Traefik LoadBalancer IP (`<TRAEFIK_LOADBALANCER_IP>`). If it doesn't, check that:
- dnsmasq is running on the server: `sudo systemctl status dnsmasq`
- The router DHCP is handing out the correct DNS: check your device's DNS settings
- UFW is allowing port 53: `sudo ufw status`

## 5. Access services via hostname

Services with Traefik IngressRoutes can be accessed via their configured hostnames. For example:

- `http://<service-name>.<your-domain>.home`

To add a new service, create a Traefik IngressRoute in the k3s-apps repository. See examples in `apps/traefik/ingressroutes/`.

## Configuration

Key variables in `linux/playbooks/dnsmasq-setup.yml`:

| Variable | Default | Description |
|---|---|---|
| `homelab_domain` | `<your-domain>.home` | Wildcard domain for local services |
| `traefik_ip` | `<TRAEFIK_LOADBALANCER_IP>` | Traefik LoadBalancer IP (all `*.<your-domain>.home` traffic routes here) |
| `upstream_dns_1` | `1.1.1.1` | Primary upstream DNS (Cloudflare) |
| `upstream_dns_2` | `8.8.8.8` | Secondary upstream DNS (Google) |

## Tailscale Integration

Once you've set up local DNS, you can extend it to work with Tailscale for remote access. This allows you to use the same `*.<your-domain>.home` URLs whether you're at home or accessing remotely via Tailscale.

### Overview

**Standard setup (local network only):**
- dnsmasq listens on localhost and LAN interface
- Returns LAN IP for `*.<your-domain>.home` queries
- Works only when connected to home WiFi

**Tailscale-integrated setup (works everywhere):**
- dnsmasq also listens on Tailscale interface
- Returns Tailscale IP for `*.<your-domain>.home` queries
- Tailscale MagicDNS forwards `*.<your-domain>.home` queries to your server via Split DNS
- Works at home AND remotely via Tailscale
- Same URLs everywhere

### How DNS Resolution Works

**When at home:**
1. Device queries dnsmasq on LAN IP
2. dnsmasq returns Tailscale IP
3. Traffic routes via Tailscale's direct LAN connection (no tunnel overhead)
4. Check with `tailscale status` — you'll see "direct <LAN_IP>:41641"

**When remote via Tailscale:**
1. Device queries Tailscale MagicDNS (`100.100.100.100`)
2. Tailscale recognizes `*.<your-domain>.home` matches Split DNS rule
3. Tailscale forwards query to your server's Tailscale IP
4. dnsmasq on server returns Tailscale IP
5. Traffic routes via Tailscale encrypted tunnel

### Setup

After completing the standard dnsmasq setup above, follow these steps:

#### 1. Run the Tailscale DNS Integration Playbook

```bash
ansible-playbook -i linux/inventory/hosts-local.ini \
  linux/playbooks/dnsmasq-tailscale-setup.yml \
  --ask-become-pass
```

This updates dnsmasq to:
- Listen on Tailscale interface in addition to localhost and LAN
- Return Tailscale IP for all `*.<your-domain>.home` queries
- Enable remote access while maintaining local network functionality

#### 2. Configure Tailscale MagicDNS

Configure Tailscale MagicDNS with Split DNS for `<your-domain>.home`. See [docs/tailscale.md - Remote Access via Tailscale DNS](tailscale.md#remote-access-via-tailscale-dns) for detailed step-by-step instructions.

**Critical configuration requirement:**
- ✅ Add Split DNS nameserver: `<YOUR_TAILSCALE_IP>` for domain `<your-domain>.home`
- ❌ Do NOT add `<your-domain>.home` to "Search Domains"

**Why this matters**: Adding `<your-domain>.home` as a search domain breaks browser DNS resolution. Command-line tools like `curl` and `dig` will work, but browsers will show `ERR_NAME_NOT_RESOLVED` errors because they try to query invalid domains like `<service-name>.<your-domain>.home.<your-domain>.home`.

### What Changes

The Tailscale integration updates your dnsmasq configuration:

**Before (local network only):**
```bash
listen-address=127.0.0.1,<LAN_IP>
address=/<your-domain>.home/<LAN_IP>
```

**After (works everywhere via Tailscale):**
```bash
listen-address=127.0.0.1,<LAN_IP>,<TAILSCALE_IP>
address=/<your-domain>.home/<TAILSCALE_IP>
```

Key changes:
- Adds Tailscale interface to listen addresses
- Changes DNS responses to return Tailscale IP instead of LAN IP
- Enables remote access via Tailscale MagicDNS

### Why Return Tailscale IP for Home Network Too?

You might wonder: "Why return the Tailscale IP even when I'm at home?"

**Answer**: Tailscale is smart enough to use direct LAN connections when both devices are on the same network. Check `tailscale status` on your devices — you'll see "direct <LAN_IP>:41641" when at home.

Benefits:
- Same URLs work everywhere (no mental context switching)
- No performance penalty (Tailscale uses LAN directly)
- Simpler configuration (one dnsmasq instance, not two)
- Easier troubleshooting (one DNS server to debug)

### Performance Impact

**Negligible**. Tailscale's WireGuard protocol adds microseconds of latency, and when you're at home, it uses direct LAN connections anyway (no tunnel overhead).

### Rollback

To revert to local-network-only DNS:

```bash
ansible-playbook -i linux/inventory/hosts-local.ini \
  linux/playbooks/dnsmasq-setup.yml \
  --ask-become-pass
```

This restores dnsmasq to the original configuration (listening only on localhost and LAN interface, returning LAN IP).

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

**DNS not resolving via Tailscale:**
```bash
# Verify dnsmasq is listening on Tailscale interface
sudo ss -tlnp | grep :53
# Should show your Tailscale IP (e.g., 100.x.x.x)

# Test direct query to Tailscale IP
dig <service-name>.<your-domain>.home @<YOUR_TAILSCALE_IP>
# Should return the Tailscale IP
```

**Browser shows `ERR_NAME_NOT_RESOLVED` but command-line tools work:**

This is the most common Tailscale DNS issue. See the detailed troubleshooting section in [docs/tailscale.md - Troubleshooting Remote Access](tailscale.md#troubleshooting-remote-access).

**Quick fix**: Check if `<your-domain>.home` was added to Tailscale's "Search Domains" - if so, remove it!

**MagicDNS not working:**
```bash
# Check MagicDNS status
tailscale status
# Should show: "MagicDNS: enabled"

# Verify Split DNS configuration in Tailscale Admin Console
# https://login.tailscale.com/admin/dns
# Should show: <your-domain>.home (Split DNS) -> <YOUR_TAILSCALE_IP>
```
