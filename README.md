# Ansible

Infrastruktur-Automatisierung mit Ansible: Playbooks, Rollen und Inventories für Staging und Production.

## Struktur

```
ansible.cfg                 # Projekt-Konfiguration (Standard-Inventory: staging)
requirements.yml            # Benötigte Collections
inventories/
  staging/                  # hosts.yml + group_vars
  production/
playbooks/
  site.yml                  # Alles (importiert common.yml + webservers.yml)
  common.yml                # Basis für alle Hosts
  webservers.yml            # Nginx auf der Gruppe "webservers"
roles/
  common/                   # Pakete, Zeitzone, Admin-User + SSH-Keys, Auto-Updates
  ssh_hardening/            # sshd: kein Root-/Passwort-Login (mit Aussperr-Schutz)
  nginx/                    # Nginx mit Default-Site
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

# Ausführen in Production
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml

# Nur einzelne Rollen per Tag
ansible-playbook playbooks/site.yml --tags ssh
ansible-playbook playbooks/site.yml --limit webservers --tags nginx

# Lint
ansible-lint
```

## Wichtig vor dem ersten Lauf

- Hosts/IPs in `inventories/*/hosts.yml` eintragen (aktuell Platzhalter).
- In `inventories/*/group_vars/all.yml` unter `common_admin_users` echte SSH-Public-Keys hinterlegen.
  Die Rolle `ssh_hardening` bricht ab, solange kein Key hinterlegt ist – damit man sich nicht aussperrt.
- Geheimnisse mit `ansible-vault` verschlüsseln, nicht im Klartext committen.
