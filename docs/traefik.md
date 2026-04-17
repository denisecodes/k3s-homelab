# Traefik Setup

This document covers how to install Traefik as the ingress controller for the K3s cluster and validate that it routes traffic correctly.

## Background

K3s ships with Traefik by default, but this homelab disables it (`--disable traefik` in `k3s/k3s-config.yml`) to allow custom configuration via Helm. This playbook installs Traefik from the official Helm chart with sensible defaults for a homelab.

## Prerequisites

- K3s cluster running (see [README](../README.md))
- Helm installed (see [docs/helm.md](helm.md))
- `kubernetes.core` collection installed:
  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

## 1. Install Traefik

```bash
ansible-playbook -i linux/inventory/hosts-local.ini traefik/playbooks/traefik-setup.yml \
  --ask-become-pass
```

The playbook will:
- Add the Traefik Helm repository
- Install Traefik in the `traefik` namespace
- Configure HTTP (port 80) and HTTPS (port 443) entrypoints
- Use K3s ServiceLB to assign an external IP from the node
- Enable the Traefik dashboard (internal only)

## 2. Verify the installation

Check that Traefik is running and has an external IP:

```bash
kubectl get svc -n traefik
```

You should see the `traefik` service with `TYPE: LoadBalancer` and an `EXTERNAL-IP` matching your node IP.

## 3. Access the dashboard

The Traefik dashboard is not exposed externally. Use port-forwarding to access it:

```bash
kubectl port-forward -n traefik deploy/traefik 9000:9000
```

Then open [http://localhost:9000/dashboard/](http://localhost:9000/dashboard/) in your browser.

## 4. Validate with a test app

After setting up local DNS (see [docs/dns.md](dns.md)), run the validation playbook:

```bash
ansible-playbook -i linux/inventory/hosts-local.ini traefik/playbooks/traefik-test.yml \
  --ask-become-pass
```

This deploys a `whoami` test app with a Traefik IngressRoute at `whoami.home.lan`, validates that:

1. Traefik routes HTTP traffic to the app
2. DNS resolves `whoami.home.lan` to the server IP
3. End-to-end request via hostname works

The playbook automatically cleans up the test resources when done.

To run only the cleanup (if a previous run failed mid-way):

```bash
ansible-playbook -i linux/inventory/hosts-local.ini traefik/playbooks/traefik-test.yml \
  --ask-become-pass --tags cleanup
```

## Configuration

Key variables in `traefik/playbooks/traefik-setup.yml`:

| Variable | Default | Description |
|---|---|---|
| `traefik_chart_version` | `36.4.0` | Traefik Helm chart version |
| `traefik_namespace` | `traefik` | Kubernetes namespace |
| `homelab_domain` | `home.lan` | Local domain for services |

## Future plans

- Migrate Traefik to an ArgoCD-managed application in the `k3s-apps` repo
- Add TLS with cert-manager (see issue F-5)
- Add Traefik middleware for security headers
