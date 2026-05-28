# Ansible Collection: rds.rds_bitwarden

Diese Collection enthält die Rolle `bitwarden_auth` zur sicheren Authentifizierung gegen Bitwarden und zur Verwaltung von Secrets in automatisierten Umgebungen (z. B. Ansible Semaphore oder GitLab CI).

## Voraussetzungen

1. Die **Bitwarden CLI (`bw`)** muss auf dem Host/Runner installiert sein.
2. Das offizielle Plugin `community.general.bitwarden` wird für reine Lesezugriffe benötigt (wird bei Installation dieser Collection automatisch als Abhängigkeit definiert).
3. Folgende Variablen müssen (z.B. über das Semaphore Environment) als *Secret* bereitgestellt werden:
   * `bitwarden_auth_client_id` (API Key Client ID)
   * `bitwarden_auth_client_secret` (API Key Secret)
   * `bitwarden_auth_master_password` (Das Master-Passwort zum Entsperren des Tresors)
   * `bitwarden_auth_server_url` *(Optional, nur bei Self-Hosted/Vaultwarden)*

## Nutzung in Playbooks

Der vollqualifizierte Name (FQCN) der Rolle lautet `rds.rds_bitwarden.bitwarden_auth`.
Die Datei `tasks/rotate_password.yml` kann jetzt direkt verwendet werden:

* Mit bestehender Session (`bitwarden_auth_session`) nutzt sie diese Session.
* Ohne Session führt sie Login/Unlock/Synchronisierung automatisch aus.
* Wenn sie die Session selbst erzeugt hat, wird danach automatisch ausgeloggt.

### Einfaches Beispiel (nur rotate_password)

Dieses Beispiel reicht für die meisten Fälle aus.

```yaml
---
- name: Passwort in Bitwarden rotieren
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Passwort für einen Eintrag rotieren
      ansible.builtin.include_role:
        name: rds.rds_bitwarden.bitwarden_auth
        tasks_from: rotate_password.yml
      vars:
        bitwarden_auth_item_id: "00000000-0000-0000-0000-000000000000"

    - name: Rotiertes Passwort weiterverwenden
      ansible.builtin.debug:
        msg: "Neues Passwort: {{ bitwarden_auth_rotated_password }}"
      no_log: true
```

### Erweiterte Nutzung (zentrale Session für viele Hosts)

Für sehr viele Hosts kann weiterhin ein zentraler Login mit geteilter Session sinnvoll sein, um API-Aufrufe zu reduzieren.
Dann wird `bitwarden_auth_session` vorab gesetzt und an `rotate_password.yml` übergeben.

### Komplexes Beispiel (zentrale Session-ID + throttle-Block)

Dieses Muster eignet sich für viele Hosts mit parallelen Host-Operationen, aber serieller Passwortrotation gegen Bitwarden.

```yaml
---
- name: Passwörter mit zentraler Bitwarden-Session rotieren
  hosts: firewalls
  gather_facts: false
  strategy: linear
  vars:
    # Einheitlicher Cache-Pfad für den zentralen Login
    bitwarden_auth_appdata_dir: /tmp/bw_ansible_shared_rotate_pw

  tasks:
    - name: Gesamter Ablauf mit zentralem Login/Logout
      block:
        - name: Zentraler Bitwarden Login
          run_once: true
          delegate_to: localhost
          ansible.builtin.include_role:
            name: rds.rds_bitwarden.bitwarden_auth

        - name: Session-ID an alle Hosts verteilen
          ansible.builtin.set_fact:
            global_bw_session: "{{ hostvars[ansible_play_hosts[0]]['bitwarden_auth_session'] }}"

        - name: Passwortrotation pro Host (throttle)
          when: bw_item_id is defined and bw_item_id != ""
          throttle: 1
          block:
            - name: Passwort in Bitwarden rotieren
              ansible.builtin.include_role:
                name: rds.rds_bitwarden.bitwarden_auth
                tasks_from: rotate_password.yml
                apply:
                  delegate_to: localhost
              vars:
                bitwarden_auth_item_id: "{{ bw_item_id }}"
                bitwarden_auth_session: "{{ global_bw_session }}"

        - name: Rotiertes Passwort auf dem Host weiterverwenden
          when: bw_item_id is defined and bw_item_id != ""
          ansible.builtin.debug:
            msg: "Rotiertes Passwort für {{ inventory_hostname }} liegt vor."
          no_log: true

      always:
        - name: Zentraler Bitwarden Logout
          run_once: true
          delegate_to: localhost
          ansible.builtin.include_role:
            name: rds.rds_bitwarden.bitwarden_auth
            tasks_from: logout.yml
```

### Wichtige Variablen

* Pflicht für `rotate_password.yml`: `bitwarden_auth_item_id`
* Rückgabewert: `bitwarden_auth_rotated_password`
* Im komplexen Beispiel pro Host: `bw_item_id`
