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
  common.yml                  # Basis für alle Hosts (Pakete, Benutzer, SSH-Härtung, Firewall)
  users.yml                   # Nur Benutzer, Gruppen, sudo-Rechte
  firewall.yml                # Nur Firewall (UFW)
  webservers.yml              # Apache2 + PHP + Let's Encrypt (einzeln: apache2.yml, php.yml)
  letsencrypt.yml             # Nur Zertifikate ausstellen + HTTPS aktivieren
  dbservers.yml / mysql.yml   # MySQL
  kubernetes_install.yml      # Kubernetes-Cluster mit kubeadm installieren
  kubernetes_update.yml       # Kubernetes Rolling Upgrade (Node für Node)
  update.yml                  # System-Updates (apt) für alle Nicht-K8s-Hosts, Reboot bei Bedarf
roles/
  common/                     # Pakete, Zeitzone, Auto-Updates
  users/                      # Benutzer, Gruppen, SSH-Keys, sudo-Regeln (/etc/sudoers.d)
  ssh_hardening/              # sshd: kein Root-/Passwort-Login (mit Aussperr-Schutz)
  firewall/                   # UFW: default deny, SSH zuerst, Regeln pro Gruppe
  letsencrypt/                # certbot (webroot), Auto-Renewal, Reload-Hook
  apache2/                    # Apache2, Module, VirtualHosts, HTTPS/TLS, Security-Header
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

## Benutzer, Gruppen und sudo

Definiert in `group_vars`. Alle Variablen `users_groups_*` und `users_list_*` werden zusammengeführt,
d. h. `all.yml` legt Benutzer für alle Hosts an, `webservers.yml` zusätzliche nur für Webserver.

```yaml
users_groups_base:
  - { name: admins, sudo: true }                       # volle Rechte, mit Passwort
  - { name: automation, sudo: true, sudo_nopasswd: true }
users_groups_web:
  - name: webdevs                                      # nur bestimmte Befehle
    sudo_nopasswd: true
    sudo_commands: [/usr/bin/systemctl reload apache2]

users_list_base:
  - name: max
    comment: Max Mustermann
    groups: [admins]
    ssh_keys: ["ssh-ed25519 AAAA... max@laptop"]
    password: "{{ vault_max_password_hash }}"          # Hash: mkpasswd -m sha-512
  - name: ex_mitarbeiter
    state: absent                                      # Benutzer + sudo-Regel entfernen
    remove_home: true
```

- sudo-Regeln landen je Gruppe/Benutzer in `/etc/sudoers.d/group-<name>` bzw. `user-<name>`,
  werden vor dem Schreiben mit `visudo` geprüft und automatisch entfernt, wenn `sudo` wegfällt.
- Gruppe `admins`: sudo **mit** Passwort → Admins brauchen einen `password`-Hash.
- Der Ansible-Benutzer `deploy` ist in `automation` (sudo ohne Passwort, nötig für `become`)
  und kann nicht versehentlich gelöscht werden.
- `ssh_keys_exclusive: true` entfernt alle nicht gelisteten Keys des Benutzers.
- Nur Benutzer ändern: `ansible-playbook playbooks/users.yml`


- Läuft über `common.yml` auf **allen** Hosts. SSH wird immer zuerst erlaubt (mit Rate-Limit),
  danach gilt: eingehend alles blockiert, ausgehend alles erlaubt.
- SSH nur aus bestimmten Netzen: `firewall_ssh_sources: ["203.0.113.0/24"]`.
- Regeln werden aus allen Variablen `firewall_rules_*` eingesammelt – jede Gruppe kann eigene beisteuern:

  ```yaml
  firewall_rules_web:
    - { port: "80,443", proto: tcp, comment: "HTTP/HTTPS" }
    - { port: "3306", proto: tcp, from: ["10.0.0.5", "10.0.0.6"], comment: "MySQL" }
  ```

- Bereits vorkonfiguriert: Webserver (80/443), Kubernetes (API 6443, etcd, kubelet, Flannel,
  NodePorts – interne Ports nur von Cluster-Nodes), MySQL-Beispiel in `group_vars/dbservers.yml`.
- Regeln werden nur hinzugefügt, nie automatisch gelöscht (`ufw delete` bei Bedarf manuell).

## Let's Encrypt / HTTPS

- VirtualHost in `group_vars/webservers.yml` mit `letsencrypt: true` markieren, `letsencrypt_email` setzen.
- DNS des `server_name` (und `server_aliases`) muss auf den Server zeigen, Port 80 muss offen sein.
- Ablauf in `webservers.yml`: Apache (HTTP) → certbot holt Zertifikat per Webroot → VirtualHost wird
  auf HTTPS umgestellt, HTTP leitet per 301 weiter (außer `/.well-known/acme-challenge/`).
- Staging-Inventory nutzt die Let's-Encrypt-**Test-CA** (`letsencrypt_staging: true`): keine
  Rate-Limits, aber Browser-Warnung. In Production echte Zertifikate.
- Erneuerung automatisch über `certbot.timer`, Apache wird danach per Deploy-Hook neu geladen.
- TLS 1.2/1.3 (Mozilla „intermediate“). HSTS per `apache2_hsts: true`, sobald HTTPS stabil läuft.

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
- In `inventories/*/group_vars/all.yml` beim Benutzer `deploy` (und euren Admins) echte SSH-Public-Keys
  hinterlegen. Die Rolle `ssh_hardening` bricht ab, solange kein Key hinterlegt ist – damit man sich nicht aussperrt.
- MySQL-Passwörter: `vault_mysql_app_password` per `ansible-vault` anlegen (siehe `vault.yml.example`).
- Geheimnisse mit `ansible-vault` verschlüsseln, nicht im Klartext committen.
