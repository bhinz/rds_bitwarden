# Ansible Collection: rds.rds_bitwarden

Diese Collection enthält die Rolle `bitwarden_auth` zur sicheren Authentifizierung gegen Bitwarden und zur Verwaltung von Secrets in automatisierten Umgebungen (z. B. Ansible Semaphore oder GitLab CI).

## Voraussetzungen

1. Die **Bitwarden CLI (`bw`)** muss auf dem Host/Runner installiert sein.
2. Das offizielle Plugin `community.general.bitwarden` wird für reine Lesezugriffe benötigt (wird bei Installation dieser Collection automatisch als Abhängigkeit definiert).
3. Folgende Variablen müssen (z.B. über das Semaphore Environment) als *Secret* bereitgestellt werden:
   * `bw_client_id` (API Key Client ID)
   * `bw_client_secret` (API Key Secret)
   * `bw_master_password` (Das Master-Passwort zum Entsperren des Tresors)
   * `bw_server_url` *(Optional, nur bei Self-Hosted/Vaultwarden)*

## Nutzung in Playbooks

Der vollqualifizierte Name (FQCN) der Rolle lautet `rds.rds_bitwarden.bitwarden_auth`. 
Die Rolle nutzt eine `block`/`always`-Struktur im aufrufenden Playbook, um ein sicheres Logout zu garantieren.

### Komplettes Beispiel (Login, Auslesen, Rotieren & Logout)

Dieses Playbook zeigt den kompletten Workflow: Einloggen, ein bestehendes Passwort auslesen, ein Passwort für einen anderen Eintrag rotieren und am Ende sicher wieder ausloggen.

```yaml
---
- name: Bitwarden Workflow Example
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Führe sensible Bitwarden-Operationen sicher aus
      block:
        # ==========================================
        # 1. Login & Unlock
        # ==========================================
        - name: Bitwarden Vault entsperren (Session-ID generieren)
          ansible.builtin.include_role:
            name: rds.rds_bitwarden.bitwarden_auth

        # ==========================================
        # 2. Bestehendes Secret auslesen (Read-Only)
        # ==========================================
        - name: Hole ein Passwort aus dem Vault
          ansible.builtin.debug:
            msg: "Gefundenes Passwort: {{ lookup('community.general.bitwarden', 'Mein_Datenbank_Eintrag', field='password', bw_session=bw_session) }}"
          no_log: true 

        # ==========================================
        # 3. Passwort rotieren (Neu generieren & Speichern)
        # ==========================================
        - name: Rotiere Passwort für einen spezifischen Eintrag
          ansible.builtin.include_role:
            name: rds.rds_bitwarden.bitwarden_auth
            tasks_from: rotate_password.yml
          vars:
            # Die eindeutige UUID des Bitwarden-Eintrags
            bw_item_id: "76be324c-abcd-1234-efgh-9876543210ab"

        - name: Das neue Passwort direkt auf dem Zielsystem anwenden
          ansible.builtin.debug:
            msg: "Das frisch generierte Passwort lautet {{ bw_rotated_password }}. Nutze es jetzt für SQL/API..."
          no_log: true

      # ==========================================
      # 4. Garantiertes Logout (Lock & Session zerstören)
      # ==========================================
      always:
        - name: Bitwarden Vault wieder sperren und ausloggen
          ansible.builtin.include_role:
            name: rds.rds_bitwarden.bitwarden_auth
            tasks_from: logout