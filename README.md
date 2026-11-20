# Ansible

Infrastruktur-Automatisierung mit Ansible: Playbooks, Rollen und Inventories für Staging und Production.

## Struktur

```
ansible.cfg                   # Projekt-Konfiguration (Standard-Inventory: staging)
requirements.yml              # Benötigte Collections
vault.yml.example             # Vorlage für Passwörter (ansible-vault)
inventories/
  staging/ | production/      # hosts.yml + group_vars (all, webservers, dbservers, kubernetes)
playbooks/
  site.yml                    # Alles installieren
  common.yml                  # Basis für alle Hosts (Pakete, User, SSH-Härtung)
  webservers.yml              # Apache2 + PHP   (auch einzeln: apache2.yml, php.yml)
  dbservers.yml / mysql.yml   # MySQL
  kubernetes_install.yml      # Kubernetes-Cluster mit kubeadm installieren
  kubernetes_update.yml       # Kubernetes Rolling Upgrade (Node für Node)
  update.yml                  # System-Updates (apt) für alle Nicht-K8s-Hosts, Reboot bei Bedarf
roles/
  common/                     # Pakete, Zeitzone, Admin-User + SSH-Keys, Auto-Updates
  ssh_hardening/              # sshd: kein Root-/Passwort-Login (mit Aussperr-Schutz)
  apache2/                    # Apache2, Module, VirtualHosts, Security-Header
  php/                        # PHP + Erweiterungen, php.ini-Einstellungen (mod_php)
  mysql/                      # MySQL (Ubuntu) bzw. MariaDB (Debian), DBs + User
  kubernetes/                 # containerd, kubeadm/kubelet/kubectl, Init, Join, Upgrade
  nginx/                      # optional, nicht in site.yml (Port-Konflikt mit Apache)
```

## Einrichtung

```bash
pip install ansible-core ansible-lint
ansible-galaxy collection install -r requirements.yml -p collections
```

## Benutzung

```bash
# Erreichbarkeit prüfen
ansible all -m ping

# Trockenlauf (Staging)
ansible-playbook playbooks/site.yml --check --diff

# Install
ansible-playbook playbooks/webservers.yml          # Apache2 + PHP
ansible-playbook playbooks/mysql.yml               # MySQL
ansible-playbook playbooks/kubernetes_install.yml  # Kubernetes-Cluster

# Update
ansible-playbook playbooks/update.yml              # apt-Updates (rollend, 50 % gleichzeitig)
ansible-playbook playbooks/update.yml -e update_serial=1 -e update_reboot=false

# Production
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml --ask-vault-pass

# Lint
ansible-lint
```

## Kubernetes

- Ein Control-Plane-Node (`k8s_control_plane`) plus beliebig viele Worker (`k8s_workers`), CNI: Flannel.
- Version steht in `inventories/<env>/group_vars/kubernetes.yml` (`kubernetes_version: "1.34"`).
- Die Pakete werden per `apt-mark hold` gesperrt, `update.yml` fasst Kubernetes-Hosts nicht an.
- **Upgrade:** `kubernetes_version` um genau eine Minor-Version erhöhen (z. B. 1.34 → 1.35), dann
  `ansible-playbook playbooks/kubernetes_update.yml`. Ablauf pro Node (seriell):
  Repo umstellen → kubeadm → `kubeadm upgrade apply/node` → drain → kubelet/kubectl → Neustart → uncordon.
  Bereits aktuelle Nodes werden übersprungen.

## Wichtig vor dem ersten Lauf

- Hosts/IPs in `inventories/*/hosts.yml` eintragen (aktuell Platzhalter).
- In `inventories/*/group_vars/all.yml` unter `common_admin_users` echte SSH-Public-Keys hinterlegen.
  Die Rolle `ssh_hardening` bricht ab, solange kein Key hinterlegt ist – damit man sich nicht aussperrt.
- MySQL-Passwörter: `vault_mysql_app_password` per `ansible-vault` anlegen (siehe `vault.yml.example`).
- Geheimnisse mit `ansible-vault` verschlüsseln, nicht im Klartext committen.
