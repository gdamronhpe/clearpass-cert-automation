# ClearPass Certificate Automation

Reusable Ansible automation for Aruba ClearPass certificate lifecycle tasks:

- preflight/API validation
- cluster UUID discovery
- current cert discovery (read-only)
- PKCS#12 replacement
- EST enrollment and install

This repo is designed for sharing across teams:

- code stays in repo
- environment data stays in `inventory/prod` (ignored by git)
- secrets stay in Ansible Vault

## Key Paths

- `inventory/example/`: sanitized templates you commit and share
- `inventory/prod/`: real environment data and secrets (gitignored)
- `playbooks/`: run entry points (`preflight`, discovery, replace, enroll)
- `roles/clearpass_cert/`: implementation logic
- `README.md`: setup and operations guide

## Quick Start

1. Clone and enter repo

```bash
git clone <repo-url> clearpass-cert-automation
cd clearpass-cert-automation
```

2. Create your local environment from templates

```bash
cp -R inventory/example inventory/prod
```

3. Encrypt secrets

```bash
ansible-vault encrypt inventory/prod/group_vars/vault.yml
ansible-vault encrypt inventory/prod/host_vars/cppm1.example.com.yml
ansible-vault encrypt inventory/prod/host_vars/cppm2.example.com.yml
```

4. Run preflight

```bash
ANSIBLE_ROLES_PATH="$(pwd)/roles" \
ansible-playbook \
-i inventory/prod/hosts.yml \
playbooks/preflight.yml \
--vault-password-file ~/.ansible/vault_pass.txt
```

5. Run first-time discovery

```bash
ANSIBLE_ROLES_PATH="$(pwd)/roles" \
ansible-playbook \
-i inventory/prod/hosts.yml \
playbooks/discover_cluster_uuids.yml \
--vault-password-file ~/.ansible/vault_pass.txt
```

## Minimal Config Example

`inventory/prod/group_vars/clearpass.yml`

```yaml
clearpass_api_path: /api
clearpass_token_path: /api/oauth
clearpass_client_id: Platform_Certificate_Manager
validate_certs: true
clearpass_cert_replace_method: PUT
```

`inventory/prod/group_vars/p12.yml`

```yaml
certificate_profiles:
  public_https_ecc:
    pkcs12_file_url: https://ha.example.com/certs/public_https_ecc.p12
    pkcs12_passphrase: "{{ vault_pkcs12_passphrase_public_https_ecc }}"
```

`inventory/prod/group_vars/publishers.yml`

```yaml
clearpass_service_cert_map:
  cppm1.example.com:
    - server_uuid: 11111111-1111-1111-1111-111111111111
      service_name: HTTPS(ECC)
      certificate_profile: public_https_ecc
```

## Inventory and Secret Model

Non-secret config:

- `group_vars/clearpass.yml`
- `group_vars/p12.yml`
- `group_vars/est.yml`
- `group_vars/publishers.yml`

Secrets (vault):

- `group_vars/vault.yml`
- `host_vars/<publisher>.yml` for per-publisher `clearpass_client_secret`

Recommended vault vars for PKCS#12:

- `pkcs12_passphrase` (global fallback)
- `vault_pkcs12_passphrase_<profile_name>` (preferred per-profile values)

## Playbooks

Run with:

```bash
ANSIBLE_ROLES_PATH="$(pwd)/roles" ansible-playbook -i inventory/prod/hosts.yml <playbook> --vault-password-file ~/.ansible/vault_pass.txt
```

`playbooks/preflight.yml`

- validates connectivity, OAuth, and core API access

`playbooks/discover_cluster_uuids.yml`

- calls `GET /cluster/server`
- writes `inventory/prod/group_vars/discovered_server_uuid_map.yml`

`playbooks/discover_current_certs.yml`

- calls `GET /server-cert/name/{server_uuid}/{service_name}`
- default target set is cluster-wide UUID union x service list
- writes `inventory/prod/group_vars/discovered_current_certs.yml`

`playbooks/replace_p12.yml`

- replaces certs using `clearpass_service_cert_map` and `certificate_profiles`

`playbooks/enroll_est.yml`

- creates CSR on ClearPass
- enrolls against EST
- installs signed cert

## Service Names

Supported service names in mappings:

- `RADIUS`
- `RadSec`
- `HTTPS(RSA)`
- `HTTPS(ECC)`

## Troubleshooting

Common replace API outcomes:

- `401` / `403`: OAuth credentials or API permissions are wrong
- `404`: UUID or service name mismatch
- `405`: wrong replace method for your ClearPass version
- `406`: request media type mismatch
- `422`: payload accepted but invalid (often PKCS#12 passphrase/key issues)

Common PKCS#12 issue:

- `Failed to extract Private Key from PKCS#12 file`
  - verify passphrase
  - verify private key exists in the `.p12`
  - for `HTTPS(ECC)`, use EC key material

## Notes for Shared/Container Mounts

On world-writable mounts, Ansible may ignore `ansible.cfg`. Use:

```bash
ANSIBLE_ROLES_PATH="$(pwd)/roles"
```

When one cluster node is offline, use:

```bash
--limit cppm1.example.com
```
