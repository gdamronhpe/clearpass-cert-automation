# ClearPass Certificate Automation

Reusable Ansible repository for Aruba ClearPass certificate lifecycle automation.

## Repository layout

```text
clearpass-cert-automation/
|-- ansible.cfg
|-- inventory/
|   |-- example/
|   |   |-- hosts.yml
|   |   |-- group_vars/
|   |   |   |-- clearpass.yml
|   |   |   |-- p12.yml
|   |   |   |-- est.yml
|   |   |   |-- publishers.yml
|   |   |   `-- vault.yml
|   |   `-- host_vars/
|   |       |-- cppm1.example.com.yml
|   |       `-- cppm2.example.com.yml
|-- playbooks/
|   |-- preflight.yml
|   |-- discover_cluster_uuids.yml
|   |-- discover_current_certs.yml
|   |-- replace_p12.yml
|   `-- enroll_est.yml
|-- roles/
|   `-- clearpass_cert/
|       |-- tasks/
|       |   |-- main.yml
|       |   |-- discover_cluster.yml
|       |   |-- discover_current_certs.yml
|       |   |-- replace_cert.yml
|       |   `-- est_enroll.yml
|       `-- defaults/
|           `-- main.yml
`-- README.md
```

## 1. Clone repository

```bash
git clone <repo-url> clearpass-cert-automation
cd clearpass-cert-automation
```

## 2. Create your environment inventory

```bash
cp -R inventory/example inventory/prod
```

Update these files for your environment:

- `inventory/prod/hosts.yml`
- `inventory/prod/group_vars/clearpass.yml`
- `inventory/prod/group_vars/p12.yml`
- `inventory/prod/group_vars/est.yml`
- `inventory/prod/group_vars/publishers.yml`
- `inventory/prod/host_vars/*.yml`

## 2a. Minimal working config example

`inventory/prod/group_vars/clearpass.yml`:

```yaml
clearpass_api_path: /api
clearpass_token_path: /api/oauth
clearpass_client_id: Platform_Certificate_Manager
validate_certs: true
clearpass_cert_replace_method: PUT
```

`inventory/prod/group_vars/p12.yml`:

```yaml
certificate_profiles:
  public_https_ecc:
    pkcs12_file_url: https://ha.example.com/certs/public_https_ecc.p12
    pkcs12_passphrase: "{{ vault_pkcs12_passphrase_public_https_ecc }}"
```

`inventory/prod/group_vars/publishers.yml`:

```yaml
clearpass_service_cert_map:
  cppm1.example.com:
    - server_uuid: 11111111-1111-1111-1111-111111111111
      service_name: HTTPS(ECC)
      certificate_profile: public_https_ecc
```

## 3. Add secrets with Ansible Vault

Shared secrets in `inventory/prod/group_vars/vault.yml`:

- `pkcs12_passphrase` (global fallback)
- `vault_pkcs12_passphrase_<profile_name>` (recommended per-profile PKCS#12 passphrases)
- `est_username`
- `est_password`
- `est_private_key_password`

Per-publisher OAuth secrets in host vars:

- `inventory/prod/host_vars/cppm1.example.com.yml` -> `clearpass_client_secret`
- `inventory/prod/host_vars/cppm2.example.com.yml` -> `clearpass_client_secret`

Encrypt the files:

```bash
ansible-vault encrypt inventory/prod/group_vars/vault.yml
ansible-vault encrypt inventory/prod/host_vars/cppm1.example.com.yml
ansible-vault encrypt inventory/prod/host_vars/cppm2.example.com.yml
```

Or edit in-place with vault:

```bash
ansible-vault edit inventory/prod/group_vars/vault.yml
ansible-vault edit inventory/prod/host_vars/cppm1.example.com.yml
ansible-vault edit inventory/prod/host_vars/cppm2.example.com.yml
```

## 4. Run preflight checks

Run this first to validate connectivity, OAuth authentication, and API endpoint
access before discovery or certificate changes.

```bash
ANSIBLE_ROLES_PATH="$(pwd)/roles" \
ansible-playbook \
-i inventory/prod/hosts.yml \
playbooks/preflight.yml \
--vault-password-file ~/.ansible/vault_pass.txt
```

If preflight fails, task output indicates whether the issue is:

- DNS/routing/TLS reachability
- OAuth credentials or token endpoint path
- API endpoint access (`GET /cluster/server`)

## 5. First run: discover cluster UUIDs

Use this as your first production run to auto-discover server UUIDs from ClearPass.
The playbook calls `GET /cluster/server` and writes a local template file.
In shared/container mounts that are world-writable, Ansible may ignore `ansible.cfg`;
set `ANSIBLE_ROLES_PATH` explicitly in the command.

```bash
ANSIBLE_ROLES_PATH="$(pwd)/roles" \
ansible-playbook \
-i inventory/prod/hosts.yml \
playbooks/discover_cluster_uuids.yml \
--vault-password-file ~/.ansible/vault_pass.txt
```

The playbook generates:

- `inventory/prod/group_vars/discovered_server_uuid_map.yml`
  - `discovered_server_uuid_map`: host key -> `[server_uuid]` mapping template

After the first run:

1. Open `inventory/prod/group_vars/discovered_server_uuid_map.yml`.
2. Copy values from `discovered_server_uuid_map`.
3. Populate `inventory/prod/group_vars/publishers.yml`:
   - `clearpass_service_cert_map`
   - `clearpass_est_service_map`

## 6. Configure PKCS#12 definitions

Edit `inventory/prod/group_vars/p12.yml`:

- `certificate_profiles`: named PKCS#12 sources used by mappings
- Recommended: reference vaulted passphrase vars per profile, for example
  `pkcs12_passphrase: "{{ vault_pkcs12_passphrase_public_https }}"`
- Fallback: omit per-profile passphrase and use global `pkcs12_passphrase` from vaulted vars

## 7. Configure EST definitions

Edit `inventory/prod/group_vars/est.yml`:

- `est_base_url`, `est_profile`: default EST endpoint settings
- `est_enrollment_profiles` (optional): multiple profile-specific EST + CSR defaults
- CSR fields:
  - Required: `subject_CN`, `private_key_type`, `digest_algorithm`
  - Required secret: `private_key_password` or `est_private_key_password`
  - Optional: `subject_O`, `subject_OU`, `subject_L`, `subject_ST`, `subject_C`, `subject_SAN`

`est_profile` is used in:

- `{{ est_base_url }}/.well-known/est/{{ est_profile }}/simpleenroll`

## 8. Configure service/server mappings

Edit `inventory/prod/group_vars/publishers.yml`:

- `clearpass_service_cert_map`: `server_uuid + service_name -> certificate_profile`
- `clearpass_est_service_map`: `server_uuid + service_name` plus optional `enrollment_profile`

Valid `service_name` values:

- `RADIUS`
- `RadSec`
- `HTTPS(RSA)`
- `HTTPS(ECC)`

## 9. Discover current certs (optional read-only audit)

After UUID discovery, you can query currently installed certs for each
`server_uuid + service_name` using `GET /server-cert/name/{server_uuid}/{service_name}`.

By default, this uses:

- all UUIDs from `discovered_server_uuid_map` (cluster-wide union)
- `clearpass_discovery_services` (`RADIUS`, `HTTPS(RSA)`, `HTTPS(ECC)`, `RadSec`)

To override targets per host, define `clearpass_cert_query_map` in
`inventory/prod/group_vars/publishers.yml`.

```bash
ANSIBLE_ROLES_PATH="$(pwd)/roles" \
ansible-playbook \
-i inventory/prod/hosts.yml \
playbooks/discover_current_certs.yml \
--vault-password-file ~/.ansible/vault_pass.txt
```

Output:

- `inventory/prod/group_vars/discovered_current_certs.yml`
  - `discovered_current_certs`: host -> queried services with HTTP status and cert payload

## 10. Run playbooks

Recommended order:

1. `playbooks/preflight.yml`
2. `playbooks/discover_cluster_uuids.yml` (first setup run)
3. `playbooks/discover_current_certs.yml` (optional audit)
4. `playbooks/replace_p12.yml` and/or `playbooks/enroll_est.yml`

Replace certificates from PKCS#12 source:

```bash
ANSIBLE_ROLES_PATH="$(pwd)/roles" \
ansible-playbook \
-i inventory/prod/hosts.yml \
playbooks/replace_p12.yml \
--vault-password-file ~/.ansible/vault_pass.txt
```

Enroll via EST and install signed certs:

```bash
ANSIBLE_ROLES_PATH="$(pwd)/roles" \
ansible-playbook \
-i inventory/prod/hosts.yml \
playbooks/enroll_est.yml \
--vault-password-file ~/.ansible/vault_pass.txt
```

## 11. Troubleshooting

Common API responses and what they usually mean:

- `401` or `403`: OAuth client credentials or API permissions are incorrect for the endpoint.
- `404`: `server_uuid` or `service_name` does not match what ClearPass knows.
- `405`: Wrong HTTP method for your ClearPass version. Set `clearpass_cert_replace_method` to `PUT` or `PATCH`.
- `406`: API did not accept request media types. Ensure JSON request headers are sent.
- `422`: Request parsed, but payload is invalid. For PKCS#12 this is commonly wrong passphrase, missing private key, or key type mismatch (for `HTTPS(ECC)`, use EC key material).

If running from a world-writable mount (`/repo/...`), Ansible may ignore `ansible.cfg`.
Use this command pattern:

```bash
ANSIBLE_ROLES_PATH="$(pwd)/roles" ansible-playbook -i inventory/prod/hosts.yml <playbook> --vault-password-file ~/.ansible/vault_pass.txt
```

Use `--limit` when a cluster node is offline:

```bash
--limit cppm1.example.com
```

## Variable model

- `group_vars/clearpass.yml`: non-secrets (`clearpass_api_path`, `clearpass_client_id`, `validate_certs`)
  - OAuth token endpoint path is `clearpass_token_path` (default `/api/oauth`)
  - PKCS#12 replace HTTP verb is `clearpass_cert_replace_method` (default `PUT`; use `POST` only if your ClearPass API expects it)
  - PKCS#12 replace success codes default to `200, 204, 304` (`clearpass_cert_replace_success_codes`)
- `group_vars/discovered_server_uuid_map.yml`: discovered UUID template (`discovered_server_uuid_map`)
- `group_vars/discovered_current_certs.yml`: optional cert discovery output (`discovered_current_certs`)
- `group_vars/p12.yml`: PKCS#12 certificate definitions (`certificate_profiles`)
- `group_vars/est.yml`: EST and CSR defaults (`est_enrollment_profiles` optional)
- `group_vars/publishers.yml`: environment mappings (`clearpass_service_cert_map`, `clearpass_est_service_map`, optional `clearpass_cert_query_map`)
- `group_vars/vault.yml`: shared secrets (`pkcs12_passphrase`, `vault_pkcs12_passphrase_<profile_name>`, `est_username`, `est_password`, `est_private_key_password`)
- `host_vars/<publisher>.yml`: per-publisher secret (`clearpass_client_secret`)

