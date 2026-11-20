# Ansible Role: patchmon

Installiert den [PatchMon](https://patchmon.net)-Agent auf einem Host per
**Auto-Enrollment**. Der vom PatchMon-Server generierte Install-Skript wird
abgerufen, legt pro Host eigene API-Credentials an, installiert den Agent nach
`/usr/local/bin/patchmon-agent` und richtet den `patchmon-agent`-systemd-Dienst
ein (meldet standardmäßig stündlich an den Server).

## Variablen (`defaults/main.yml`)

| Variable | Default | Beschreibung |
|----------|---------|--------------|
| `patchmon_server_url` | `""` | Basis-URL des PatchMon-Servers, z. B. `https://patchmon.example.com` |
| `patchmon_enroll_token_key` | `""` | Auto-Enrollment-Token-Key (aus der PatchMon-UI) |
| `patchmon_enroll_token_secret` | `""` | Zugehöriges Token-Secret – **in die Vault** |
| `patchmon_enroll_type` | `direct-host` | `direct-host` oder `proxmox-lxc` |
| `patchmon_service_enabled` | `true` | Dienst aktivieren/starten |

## Idempotenz

Die Installation läuft nur, solange `/etc/patchmon/credentials.yml` noch nicht
existiert (`creates:`). Auf bereits registrierten Hosts wird der Schritt
übersprungen.

## Verwendung

Wird über `playbooks/common.yml` auf **alle** Hosts ausgerollt. Token in der
PatchMon-UI unter Auto-Enrollment erzeugen und in `group_vars/all` bzw. der
Vault hinterlegen.
