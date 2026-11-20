# Ansible Role: kubernetes

Installs and updates Kubernetes components (`kubelet`, `kubeadm`, `kubectl`) and
pins them to a defined version. Supports Debian/Ubuntu (apt) and RHEL/EL (dnf)
using the official `pkgs.k8s.io` repositories.

## Version

Both the install and update targets default to **1.37.2**.

## Role Variables

See `defaults/main.yml`:

| Variable | Default | Description |
|----------|---------|-------------|
| `kubernetes_install_version` | `1.37.2` | Version used for a fresh install |
| `kubernetes_update_version`  | `1.37.2` | Version used when `kubernetes_action=update` |
| `kubernetes_repo_version`    | `v1.37`  | Package repo channel (`major.minor`) |
| `kubernetes_hold_packages`   | `true`   | Pin packages so they are not auto-upgraded |
| `kubernetes_packages`        | kubelet, kubeadm, kubectl | Managed packages |

> When bumping to a new minor release, update **both** the version and
> `kubernetes_repo_version` (e.g. `v1.38`), since each minor release has its own
> package repository.

## Usage

Fresh install (uses `kubernetes_install_version`):

```yaml
- hosts: k8s_nodes
  become: true
  roles:
    - kubernetes
```

Update an existing node (uses `kubernetes_update_version`):

```bash
ansible-playbook site.yml -e "kubernetes_action=update"
```
