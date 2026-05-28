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
Die Rolle nutzt eine `block`/`always`-Struktur im aufrufenden Playbook, um ein sicheres Logout zu garantieren.

### Komplettes Beispiel (Login, Auslesen, Rotieren & Logout)

Dieses Playbook zeigt den kompletten Workflow: Einloggen, ein bestehendes Passwort auslesen, ein Passwort für einen anderen Eintrag rotieren und am Ende sicher wieder ausloggen.

```yaml
---
- name: Passwörter rotieren und mit Bitwarden synchronisieren
  hosts: localhost
  gather_facts: false
  strategy: linear
  vars:
    # Einheitlicher Cache-Pfad für den zentralen Login
    bitwarden_auth_appdata_dir: /tmp/bw_ansible_shared_rotate_pw

  tasks:
    - name: Gesamter Ablauf mit sicherem zentralen Logout
      block:
        # ==========================================
        # 1. Zentraler Login & Unlock (Einmalig)
        # ==========================================
        - name: Zentraler Bitwarden Login
          run_once: true
          delegate_to: localhost
          block:
            - name: Bitwarden Vault entsperren (Session-ID generieren)
              ansible.builtin.include_role:
                name: rds.rds_bitwarden.bitwarden_auth

        - name: Session-ID an alle Hosts verteilen
          ansible.builtin.set_fact:
            global_bw_session: "{{ hostvars[ansible_play_hosts[0]]['bitwarden_auth_session'] }}"

        # ==========================================
        # 2. Passwort in Bitwarden rotieren (Throttled)
        # ==========================================
        - name: "Isolierter Block für Bitwarden Passwort-Rotation"
          # Verhindert Dateisperren (File Locks) auf dem gemeinsamen bw_ansible_shared Cache
          throttle: 1
          block:
            - name: Rotiere Passwort in Bitwarden für diesen Host
              ansible.builtin.include_role:
                name: rds.rds_bitwarden.bitwarden_auth
                tasks_from: rotate_password.yml
                apply:
                  delegate_to: localhost
              vars:
                bitwarden_auth_item_id: "{{ bw_item_id }}"
                bitwarden_auth_session: "{{ global_bw_session }}"

        - name: Das neue Passwort direkt auf dem Zielsystem anwenden
            ansible.builtin.debug:
              msg: "Das frisch generierte Passwort lautet {{ bitwarden_authrotated_password }}."
            no_log: true

    # ==========================================
    # 3. Garantiertes Logout (Einmalig)
    # ==========================================
    always:
      - name: Zentraler Bitwarden Logout
        run_once: true
        delegate_to: localhost
        block:
          - name: Bitwarden Vault wieder sperren und ausloggen
            ansible.builtin.include_role:
              name: rds.rds_bitwarden.bitwarden_auth
              tasks_from: logout
