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
ansible-playbook -i linux/inventory/hosts-local.ini linux/playbooks/tailscale-install.yml \
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
ansible-playbook -i linux/inventory/hosts-local.ini linux/playbooks/tailscale-auth.yml \
  --ask-become-pass --ask-vault-pass
```

The playbook will:
- Authenticate each node to your Tailscale account using the auth key
- Open UFW to allow Tailscale traffic (`41641/udp` and the `tailscale0` interface)
- Print the Tailscale IP for each node at the end of the run

The output will look like this:

```
ok: [<YOUR_NODE_LOCAL_IP>] => {
    "msg": "Tailscale IP for <YOUR_NODE_LOCAL_IP>: 100.64.0.1"
}
```

> The IP on the left (`<YOUR_NODE_LOCAL_IP>`) is your node's local network IP — this is just how Ansible identifies the host from your inventory. The IP on the right (`100.64.0.1`) is the Tailscale IP. **This is the one you need** — use it to update `MASTER_NODE_IP` in your GitHub Actions secrets.

## 5. Configure GitHub Actions secrets

Update the following secrets in your repository at `https://github.com/<YOUR_USERNAME>/<YOUR_REPO>/settings/secrets/actions`:

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

## Remote Access via Tailscale DNS

Once Tailscale is installed and authenticated, you can access your homelab services using the same `*.<your-domain>.home` URLs whether you're at home or connected remotely via Tailscale.

### Why DNS Integration?

Without DNS integration, remote access requires using IP:port combinations (e.g., `http://<TAILSCALE_IP>:30080`). With DNS integration, you can use the same friendly URLs everywhere:

- **At home**: `http://<service-name>.<your-domain>.home`
- **Remote via Tailscale**: `http://<service-name>.<your-domain>.home` (same URL!)

### Prerequisites

- Tailscale installed and authenticated on homelab nodes (steps 1-4 above)
- dnsmasq configured for local DNS (see [docs/dns.md](dns.md))
- Traefik accessible on Tailscale IP (verify with: `curl http://<TAILSCALE_IP>/`)

### 1. Update dnsmasq for Tailscale

Run the Tailscale DNS setup playbook to configure dnsmasq to listen on the Tailscale interface and return the Tailscale IP for all `*.<your-domain>.home` queries:

```bash
ansible-playbook -i linux/inventory/hosts-local.ini \
  linux/playbooks/dnsmasq-tailscale-setup.yml \
  --ask-become-pass
```

This playbook will:
- Detect your server's Tailscale IP automatically
- Update dnsmasq to listen on localhost, LAN interface, and Tailscale interface
- Configure wildcard DNS to return the Tailscale IP for `*.<your-domain>.home`
- Verify the configuration is working

**Expected output:**
```
DNS test: test.<your-domain>.home resolves to <TAILSCALE_IP>.
✅ Wildcard DNS is working correctly - resolving to Tailscale IP.
```

### 2. Configure Tailscale MagicDNS

Enable MagicDNS and configure Split DNS to forward `*.<your-domain>.home` queries to your homelab server:

1. **Go to Tailscale DNS Settings**
   - Navigate to [https://login.tailscale.com/admin/dns](https://login.tailscale.com/admin/dns)

2. **Enable MagicDNS**
   - If not already enabled, toggle "MagicDNS" to ON
   - This allows Tailscale to manage DNS for your tailnet

3. **Add Split DNS Nameserver**
   - In the "Nameservers" section, you should see your tailnet domain (e.g., `tail7b6c16.ts.net`)
   - Below that, click "Add nameserver"
   - A dialog will appear - select the **"Split DNS"** option
   - Enter:
     - **Domain**: `<your-domain>.home`
     - **Nameserver**: Your server's Tailscale IP (e.g., `100.x.x.x`)
       - To find this: Run `tailscale ip -4` on your server, or check the playbook output from step 1
   - Click "Add" or "Save"
   
   You should now see an entry in the Nameservers section like:
   ```
   <your-domain>.home     Split DNS
   100.x.x.x
   ```
   
   This configures Tailscale to forward all `*.<your-domain>.home` queries to your homelab server's dnsmasq.

4. **⚠️ CRITICAL: Verify Search Domains**
   - Scroll down to the "Search Domains" section
   - **Verify `<your-domain>.home` is NOT listed here**
   - You should only see your Tailscale domain (e.g., `tail7b6c16.ts.net`)
   
   **Why this matters**: If `<your-domain>.home` appears in "Search Domains", remove it immediately!
   - Search domains are for appending to single-word queries (e.g., `server` → `server.company.com`)
   - Having `<your-domain>.home` as a search domain breaks browser DNS resolution
   - Browsers will try invalid queries like `<service-name>.<your-domain>.home.<your-domain>.home`
   - This causes `ERR_NAME_NOT_RESOLVED` errors even though command-line tools work
   
   **To remove**: Click the X or remove button next to `<your-domain>.home` in the Search Domains section

5. **Wait for DNS Propagation**
   - DNS changes typically propagate within 10-20 seconds
   - Tailscale clients will automatically pick up the new configuration

### 3. Verify from Your Devices

**From your Mac** (while connected to Tailscale):

```bash
# Test DNS resolution
dig <service-name>.<your-domain>.home
# Should return your Tailscale IP (e.g., 100.x.x.x)

# Verify DNS configuration (optional)
scutil --dns | grep "search domain"
# Should NOT show <your-domain>.home in the search domains list
```

**From your iPhone** (connected to Tailscale, NOT on home WiFi):

Open Safari and navigate to your services using the `*.<your-domain>.home` URLs. For example:
- `http://<service-name>.<your-domain>.home` (replace with your actual service name)

All services with IngressRoutes configured for `*.<your-domain>.home` should load successfully!

### How It Works

**When at home:**
- Your device queries dnsmasq on the homelab server
- dnsmasq returns the Tailscale IP (e.g., `100.x.x.x`)
- Traffic routes via Tailscale's direct LAN connection (no tunnel overhead)
- Check with `tailscale status` — you'll see "direct <LAN_IP>:41641"

**When remote via Tailscale:**
- Your device queries dnsmasq via Tailscale MagicDNS
- dnsmasq returns the Tailscale IP (e.g., `100.x.x.x`)
- Traffic routes via Tailscale's encrypted WireGuard tunnel
- Same URLs, seamless experience

### Troubleshooting Remote Access

#### Browser Shows `ERR_NAME_NOT_RESOLVED` for `*.<your-domain>.home`

**Symptoms:**
- Command-line tools work: `curl http://<service-name>.<your-domain>.home` succeeds
- DNS resolution works: `dig <service-name>.<your-domain>.home` returns correct IP
- Browser fails: Shows "DNS address could not be found" or `ERR_NAME_NOT_RESOLVED`

**Root Cause:**
`<your-domain>.home` was incorrectly added to Tailscale's "Search Domains" section. This causes browsers to append the search domain to queries, creating invalid domains like `<service-name>.<your-domain>.home.<your-domain>.home`.

**Solution:**

1. **Check Tailscale DNS Configuration**
   - Go to https://login.tailscale.com/admin/dns
   - Scroll to "Search Domains" section
   - **If `<your-domain>.home` is listed**: Click the X or remove button to delete it
   - **Keep only**: Your Tailscale domain (e.g., `tail7b6c16.ts.net`) and any other legitimate search domains

2. **Verify Split DNS Configuration Still Exists**
   - In the "Nameservers" section, you should still see:
     ```
     <your-domain>.home     Split DNS
     <YOUR_TAILSCALE_IP>
     ```
   - This is correct and should remain

3. **Wait and Test**
   ```bash
   # Wait 10-20 seconds for DNS changes to propagate
   
   # Verify search domains (macOS)
   scutil --dns | grep "search domain"
   # Should NOT show <your-domain>.home
   
   # Test in browser
   open http://<service-name>.<your-domain>.home
   ```

4. **If Still Not Working**
   - Clear browser DNS cache:
     - **Chrome**: Visit `chrome://net-internals/#dns` → Click "Clear host cache"
     - **Safari**: Clear all website data or use Private Browsing
   - Restart Tailscale client on your device
   - Try opening `http://<service-name>.<your-domain>.home` in incognito/private mode

#### DNS Not Resolving on Tailscale

**Symptoms:**
- `dig <service-name>.<your-domain>.home` returns `NXDOMAIN` or no answer

**Solution:**
```bash
# Check MagicDNS is enabled
tailscale status
# Should show: "MagicDNS: enabled"

# Test direct query to your server
dig <service-name>.<your-domain>.home @<YOUR_TAILSCALE_IP>
# Should return the Tailscale IP

# If direct query works but MagicDNS doesn't:
# - Verify Split DNS is configured (see step 2 above)
# - Check that <your-domain>.home is NOT in search domains
```

#### DNS Resolves But Connection Times Out

**Symptoms:**
- DNS resolves correctly: `dig <service-name>.<your-domain>.home` returns `100.x.x.x`
- Browser shows "Took too long to respond" or connection timeout

**Possible Causes:**

1. **Tailscale Not Connected**
   ```bash
   # Check Tailscale status
   tailscale status
   # Should show "Online" and list your server
   ```

2. **Server Not Reachable**
   ```bash
   # Test connectivity to server
   ping <YOUR_TAILSCALE_IP>
   
   # Test SSH
   ssh <user>@<YOUR_TAILSCALE_IP>
   ```

3. **Traefik Not Accessible**
   ```bash
   # Verify Traefik is accessible on Tailscale IP
   curl -I http://<YOUR_TAILSCALE_IP>/
   # Should return HTTP 404 (Traefik responding)
   
   # Test with host header
   curl -I -H "Host: <service-name>.<your-domain>.home" http://<YOUR_TAILSCALE_IP>/
   # Should return HTTP 200 or redirect
   ```

4. **dnsmasq Not Running on Server**
   ```bash
   # SSH to server and check
   ssh <user>@<YOUR_TAILSCALE_IP>
   sudo systemctl status dnsmasq
   # Should show "active (running)"
   ```

#### Services Work at Home But Not Remotely

**Symptoms:**
- `http://<service-name>.<your-domain>.home` works on home WiFi
- Same URL fails when connected via Tailscale on different network

**Solution:**
This indicates Tailscale Split DNS is not configured correctly. Follow the "Configure Tailscale MagicDNS" section above, specifically:
- Ensure Split DNS nameserver is added for `<your-domain>.home` domain
- Verify `<your-domain>.home` is NOT in search domains
- Test with `dig <service-name>.<your-domain>.home` - should return Tailscale IP, not local IP

#### Verify DNS Configuration (macOS)

To check your DNS configuration is correct:

```bash
# View all DNS resolvers
scutil --dns | head -30

# Expected output for resolver #1:
#   search domain[0] : tail<random>.ts.net
#   search domain[1] : lan (or your network domain)
#   nameserver[0] : 100.100.100.100 (Tailscale MagicDNS)

# Key point: <your-domain>.home should NOT appear in search domains

# Test DNS resolution
dig <service-name>.<your-domain>.home
# Should return: ;; ANSWER SECTION: <service-name>.<your-domain>.home. 0 IN A <TAILSCALE_IP>
```

#### Home Network DNS Still Works

After updating dnsmasq, your home network devices should continue to work normally. The change from returning the LAN IP to returning the Tailscale IP is transparent because:
- Tailscale uses direct LAN connections when both devices are on the same network
- Traefik listens on all interfaces, including the Tailscale interface
- Performance impact is negligible (WireGuard adds microseconds)
- Check with `tailscale status` — you'll see "direct <LAN_IP>:41641" when at home

### Rollback

If you need to revert to the original dnsmasq configuration:

```bash
ansible-playbook -i linux/inventory/hosts-local.ini \
  linux/playbooks/dnsmasq-setup.yml \
  --ask-become-pass
```

This will restore dnsmasq to listen only on localhost and the LAN interface, returning the LAN IP for `*.<your-domain>.home` queries.

**Fallback access method**: You can always access services via NodePort even if DNS is not working:
- ArgoCD: `http://<TAILSCALE_IP>:30080`
- Longhorn: `http://<TAILSCALE_IP>:<LONGHORN_NODEPORT>`

## Configuration reference

| Variable | File | Description |
|----------|------|-------------|
| `tailscale_authkey` | `linux/vault/secrets-local.yml` | Pre-auth key for authenticating nodes to your tailnet |
