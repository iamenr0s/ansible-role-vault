[![Molecule](https://github.com/iamenr0s/ansible-role-vault/actions/workflows/molecule.yml/badge.svg)](https://github.com/iamenr0s/ansible-role-vault/actions/workflows/molecule.yml) ![Ansible Role](https://img.shields.io/ansible/role/d/iamenr0s/ansible_role_vault) [![CodeFactor](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-vault/badge)](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-vault)

Ansible Role: Vault
===================

This role integrates playbooks with an **external, pre-existing** HashiCorp Vault server. It authenticates (AppRole, Token, or Kubernetes auth), reads and writes KV secrets — rendering them into config files on target hosts — and signs SSH host/client certificates via Vault's SSH secrets engine. Zero static secrets or long-lived SSH keys in your repo.

This role never installs or configures the Vault server itself.

Features
--------
- Authenticates via AppRole, a pre-issued Token, or Kubernetes service account JWT — normalized into a single in-memory session token.
- Reads KV v1/v2 secrets and exposes them as the `vault_secret_data` fact for templating.
- Renders secrets into config files with strict permissions (`0600` by default) and backups.
- Writes/rotates KV v1/v2 secrets from Ansible.
- Signs SSH host certificates (eliminates SSH TOFU prompts) and short-lived client certificates.
- Every task touching credentials or secret data runs with `no_log: true`.

Requirements
------------
- ansible-core >= 2.16 and Python >= 3.10 on the controller.
- The `hashicorp.vault` collection (no stable Galaxy release yet — installed from GitHub):
  - `ansible-galaxy collection install -r requirements.yml`
- Python `requests` on the managed hosts (the collection's modules execute there).
- Network access to the Vault server from the controller and from target hosts.
- An existing Vault server with the relevant auth methods and secrets engines already configured.

Supported Platforms
-------------------
- AlmaLinux/Rocky/EL 8, 9, 10
- Fedora 42, 43, 44
- Debian 12 (bookworm), 13 (trixie)
- Ubuntu 22.04 (jammy), 24.04 (noble)

Role Variables
--------------
Defined in `defaults/main.yml` (validated by `meta/argument_specs.yml`):

- `vault_addr` (str, required): Vault server URL, e.g. `https://vault.example.com:8200`.
- `vault_validate_certs` (bool): Validate Vault's TLS certificate (default: `true`).
- `vault_ca_cert` (path): PEM CA certificate for TLS verification of KV module calls and the role's `uri` calls (token lookup, SSH signing). For AppRole/Kubernetes login (`auth_login`), trust the CA on the executing host instead — system store or `REQUESTS_CA_BUNDLE` (optional).
- `vault_namespace` (str): Vault namespace — `root` for OSS/self-hosted, `admin` for HCP Vault Dedicated (default: `root`).
- `vault_auth_method` (str): `approle`, `token`, or `kubernetes` (default: `approle`).
- `vault_approle_mount` (str): AppRole auth mount path (default: `approle`).
- `vault_approle_role_id` / `vault_approle_secret_id` (str): AppRole credentials — required for `approle`.
- `vault_token_value` (str): Pre-issued token — required for `token`.
- `vault_k8s_mount` (str): Kubernetes auth mount path (default: `kubernetes`).
- `vault_k8s_jwt_path` (path): Projected service account JWT path (default: `/var/run/secrets/kubernetes.io/serviceaccount/token`).
- `vault_k8s_role` (str): Vault role bound to the service account — required for `kubernetes`.
- `vault_operations` (dict): Operation toggles `read`, `write`, `ssh_host_sign`, `ssh_client_sign` (all default: `false`).
- `vault_kv_mount` (str): KV engine mount path (default: `secret`).
- `vault_kv_engine_version` (int): KV engine version, `1` or `2` (default: `2`).
- `vault_kv_read_path` (str): Secret path to read — required for `read`.
- `vault_kv_write_path` (str) / `vault_kv_write_data` (dict): Secret path and data to write — required for `write`.
- `vault_render_template` (path) / `vault_render_dest` (path): Optional Jinja2 template consuming `vault_secret_data`, and its destination on the target.
- `vault_render_owner` / `vault_render_group` / `vault_render_mode`: Ownership and mode of rendered files (defaults: `root` / `root` / `0600`).
- `vault_render_backup` (bool): Back up the previous rendered file (default: `false`). Backups are timestamped plaintext copies of old secret values — if enabled, purge `*~` files as part of secret rotation.
- `vault_ssh_mount` (str): SSH secrets engine mount path (default: `ssh`).
- `vault_ssh_host_role` (str): Vault SSH role for host certificates — required for `ssh_host_sign`.
- `vault_ssh_host_pubkey_path` / `vault_ssh_host_cert_path`: Host public key to sign and signed cert destination (defaults: `/etc/ssh/ssh_host_ed25519_key.pub` / `/etc/ssh/ssh_host_ed25519_key-cert.pub`).
- `vault_ssh_host_cert_ttl` / `vault_ssh_host_cert_mode`: Host cert TTL and file mode (defaults: `24h` / `0644`).
- `vault_ssh_client_role` / `vault_ssh_client_pubkey_path` / `vault_ssh_client_cert_path`: Client (user) certificate signing inputs — required for `ssh_client_sign`.
- `vault_ssh_client_cert_ttl` / `vault_ssh_client_cert_mode`: Client cert TTL and file mode (defaults: `8h` / `0644`).
- `vault_debug_unsafe` (bool): NEVER enable outside disposable environments (default: `false`).

Behavior Overview
-----------------
Dispatcher (`tasks/main.yml`):
- Asserts `vault_addr` and the required variables for the selected auth method.
- Includes `tasks/auth/<method>.yml`, which produces the in-memory `vault_token` fact.
- Runs only the operations enabled in `vault_operations`.

Auth (`tasks/auth/`):
- `approle.yml` / `kubernetes.yml` use `hashicorp.vault.auth_login`; `token.yml` verifies the supplied token via `auth/token/lookup-self`.
- Each method is wrapped in `block`/`rescue` with actionable error messages.

Secrets (`tasks/secrets/`):
- `read.yml` reads KV v1/v2 via `kv1_secret_info`/`kv2_secret_info` and exposes `vault_secret_data`; optionally renders `vault_render_template` to `vault_render_dest`.
- `write.yml` writes `vault_kv_write_data` via `kv1_secret`/`kv2_secret`.

SSH signing (`tasks/ssh/`):
- The `hashicorp.vault` collection has no SSH secrets engine module, so `sign_host.yml`/`sign_client.yml` call the Vault HTTP API (`POST /v1/<mount>/sign/<role>`) with `ansible.builtin.uri` and write the signed certificate to disk.

Idempotency & Rollback
-----------------------
- Idempotent: Partial — KV reads and file rendering only report changes when content changes; SSH certificate signing intentionally issues a fresh certificate (`changed`) on every run.
- Atomic: No — a failure mid-run can leave some operations applied; each operation is independent.
- Rollback: KV v2 retains prior secret versions in Vault itself; rendered files can keep on-disk backups via `vault_render_backup` (off by default — see the variable note).
- Check mode: Not supported — Vault API operations require live calls whose results feed later tasks.

Dependencies
------------
- Collections: `hashicorp.vault` (see `requirements.yml` — installed from GitHub until a stable Galaxy release exists).
- Role dependencies: none.

Example Usage
-------------
Read a secret and render it into an .env file (AppRole):

```yaml
- hosts: app_servers
  become: true
  roles:
    - role: iamenr0s.ansible_role_vault
      vars:
        vault_addr: "https://vault.example.com:8200"
        vault_auth_method: approle
        vault_approle_role_id: "{{ lookup('ansible.builtin.env', 'VAULT_ROLE_ID') }}"
        vault_approle_secret_id: "{{ lookup('ansible.builtin.env', 'VAULT_SECRET_ID') }}"
        vault_operations:
          read: true
        vault_kv_read_path: myapp/prod
        vault_render_template: templates/myapp.env.j2
        vault_render_dest: /etc/myapp/.env
```

Sign SSH host certificates (Token auth):

```yaml
- hosts: all
  become: true
  roles:
    - role: iamenr0s.ansible_role_vault
      vars:
        vault_addr: "https://vault.example.com:8200"
        vault_auth_method: token
        vault_token_value: "{{ lookup('ansible.builtin.env', 'VAULT_TOKEN') }}"
        vault_operations:
          ssh_host_sign: true
        vault_ssh_host_role: host-role
```

Kubernetes auth (play running inside a pod):

```yaml
- hosts: localhost
  roles:
    - role: iamenr0s.ansible_role_vault
      vars:
        vault_addr: "https://vault.example.com:8200"
        vault_auth_method: kubernetes
        vault_k8s_role: my-workload
        vault_operations:
          read: true
        vault_kv_read_path: myapp/prod
```

Notes
-----
- Credentials (role_id/secret_id, token, SA JWT) must come from the playbook layer — environment, AAP credentials, or ansible-vault encrypted vars — never from role defaults.
- The session token lives only as an in-memory fact for the duration of the play. Disable fact caching for plays using this role, or scope it away from this role's facts.
- `auth_login` has no TLS parameters: server trust for the login call comes from the executing host's CA store (or `REQUESTS_CA_BUNDLE`). `vault_validate_certs`/`vault_ca_cert` apply to the KV modules and the role's `uri` calls.
- Set `vault_namespace` to `admin` (HCP Vault Dedicated) or your Enterprise namespace when not using OSS Vault.

## Contributing

Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for the
local pipeline commands and pull request checklist. This project follows the
[Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).

## Security

See [SECURITY.md](SECURITY.md) — GitHub private vulnerability reporting, no
public issues for security bugs.

## License

This project is licensed under the [MIT License](LICENSE).

## Author Information

Author: iamenr0s

Galaxy: `iamenr0s.ansible_role_vault`
