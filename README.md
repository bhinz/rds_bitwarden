# Ansible Collection: bhinz.rds_bitwarden

This collection contains the `bitwarden_auth` role for secure authentication against Bitwarden and secret management in automated environments (for example Ansible Semaphore or GitLab CI).

## Requirements

1. The **Bitwarden CLI (`bw`)** must be installed on the host/runner.
2. The following variables must be provided:
   * `bitwarden_auth_client_id` (API Key Client ID)
   * `bitwarden_auth_client_secret` (API Key Secret)
   * `bitwarden_auth_master_password` (The master password used to unlock the vault)
   * `bitwarden_auth_server_url` *(Optional, only for self-hosted/Vaultwarden)*

## Usage in Playbooks

The fully qualified role name (FQCN) is `bhinz.rds_bitwarden.bitwarden_auth`.
The `tasks/rotate_password.yml` file can be used directly:

* With an existing session (`bitwarden_auth_session`), it uses that session.
* Without a session, it automatically performs login/unlock/synchronization.
* If it created the session itself, it will automatically log out afterwards.

### Simple example (rotate_password only)

This example is sufficient for most use cases.

```yaml
---
- name: Rotate password in Bitwarden
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Rotate password for an item
      ansible.builtin.include_role:
        name: bhinz.rds_bitwarden.bitwarden_auth
        tasks_from: rotate_password.yml
      vars:
        bitwarden_auth_item_id: "00000000-0000-0000-0000-000000000000"

    - name: Reuse rotated password
      ansible.builtin.debug:
        msg: "New password: {{ bitwarden_auth_rotated_password }}"
      no_log: true
```

### Advanced usage (central session for many hosts)

For a large number of hosts, a central login with a shared session can still be useful to reduce API calls.
In that case, set `bitwarden_auth_session` in advance and pass it to `rotate_password.yml`.

### Complex example (central session ID + throttle block)

This pattern is useful for many hosts with parallel host operations, but serialized password rotation against Bitwarden.

```yaml
---
- name: Rotate passwords with a central Bitwarden session
  hosts: firewalls
  gather_facts: false
  strategy: linear
  vars:
    # Shared cache path for central login
    bitwarden_auth_appdata_dir: /tmp/bw_ansible_shared_rotate_pw

  tasks:
    - name: Complete workflow with central login/logout
      block:
        - name: Central Bitwarden login
          run_once: true
          delegate_to: localhost
          ansible.builtin.include_role:
            name: bhinz.rds_bitwarden.bitwarden_auth

        - name: Distribute session ID to all hosts
          ansible.builtin.set_fact:
            global_bw_session: "{{ hostvars[ansible_play_hosts[0]]['bitwarden_auth_session'] }}"

        - name: Per-host password rotation (throttle)
          when: bw_item_id is defined and bw_item_id != ""
          throttle: 1
          block:
            - name: Rotate password in Bitwarden
              ansible.builtin.include_role:
                name: bhinz.rds_bitwarden.bitwarden_auth
                tasks_from: rotate_password.yml
                apply:
                  delegate_to: localhost
              vars:
                bitwarden_auth_item_id: "{{ bw_item_id }}"
                bitwarden_auth_session: "{{ global_bw_session }}"

        - name: Reuse rotated password on the host
          when: bw_item_id is defined and bw_item_id != ""
          ansible.builtin.debug:
            msg: "Rotated password for {{ inventory_hostname }} is available."
          no_log: true

      always:
        - name: Central Bitwarden logout
          run_once: true
          delegate_to: localhost
          ansible.builtin.include_role:
            name: bhinz.rds_bitwarden.bitwarden_auth
            tasks_from: logout.yml
```

### Important variables

* Required for `rotate_password.yml`: `bitwarden_auth_item_id`
* Return value: `bitwarden_auth_rotated_password`
* Per-host in the complex example: `bw_item_id`
