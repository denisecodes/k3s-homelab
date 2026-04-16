# Helm Setup

This document covers how to install Helm on your k3s cluster nodes using the Ansible playbook in `helm/playbooks/helm-install.yml`.

Helm is the Kubernetes package manager used to install ArgoCD and other applications on the cluster. It must be installed on the target hosts before running any Helm-based playbooks.

## Prerequisites

- K3s cluster already running (see [README](../README.md))
- Linux baseline applied (see [baseline setup](../README.md))

## Run the playbook

From the repo root:

```bash
ansible-playbook -i linux/inventory/hosts.ini helm/playbooks/helm-install.yml \
  --ask-become-pass
```

- `--ask-become-pass` — required for `sudo` on the remote host

The playbook downloads and runs the official `get-helm-3` install script from [helm.sh](https://helm.sh). It is idempotent — if Helm is already installed at `/usr/local/bin/helm`, the install step is skipped.

## Verify

SSH into the master node and confirm Helm is installed:

```bash
helm version
```

You should see output like:

```
version.BuildInfo{Version:"v3.x.x", ...}
```

## Next steps

Once Helm is installed, proceed to [ArgoCD setup](argocd.md).
