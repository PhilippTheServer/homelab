# HashiCorp Vault

This homelab uses HashiCorp Vault for two distinct purposes:

1. **Secrets store** — all service credentials live at `secret/data/ansible`; Ansible fetches them at runtime so nothing sensitive is ever committed to the repo
2. **SSH certificate authority** — Vault signs short-lived SSH certificates instead of distributing static keys; every SSH session is audited and time-limited

Vault lives at `vault.home.philippthesurfer.com` (LAN/VPN only — never public).  
Authentication for humans uses OIDC via Keycloak. CI/CD uses AppRole.

---

## Table of Contents

- [Install the Vault CLI](#install-the-vault-cli)
- [Connect and authenticate](#connect-and-authenticate)
- [SSH certificate setup](#ssh-certificate-setup)
  - [macOS](#macos)
  - [Linux](#linux)
  - [Windows](#windows)
- [Daily SSH workflow](#daily-ssh-workflow)
- [Managing secrets](#managing-secrets)
  - [View secrets](#view-secrets)
  - [Add or update a secret field](#add-or-update-a-secret-field)
  - [Remove a secret field](#remove-a-secret-field)
  - [Delete a secret version](#delete-a-secret-version)
- [Using secrets in Ansible](#using-secrets-in-ansible)
  - [How it works](#how-it-works)
  - [The secrets section of vars.yml](#the-secrets-section-of-varsyml)
  - [Running playbooks](#running-playbooks)
  - [CI/CD via AppRole](#cicd-via-approle)
  - [Adding a new secret](#adding-a-new-secret)

---

## Install the Vault CLI

The `vault` binary is used to authenticate, sign SSH keys, and manage secrets from your local machine.

### macOS

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/vault
```

### Linux (Debian/Ubuntu)

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vault
```

### Windows

With [Chocolatey](https://chocolatey.org/):
```powershell
choco install vault
```

With [Scoop](https://scoop.sh/):
```powershell
scoop bucket add main
scoop install main/vault
```

Or download the binary directly from [releases.hashicorp.com/vault](https://releases.hashicorp.com/vault/) and add it to your `PATH`.

### Verify installation

```bash
vault version
```

---

## Connect and authenticate

Point the CLI at your Vault instance. Add this to your shell profile (`.zshrc`, `.bashrc`, or Windows environment variables) so it persists across sessions:

```bash
export VAULT_ADDR=https://vault.home.philippthesurfer.com
```

Then log in via Keycloak OIDC:

```bash
vault login -method=oidc
```

This opens a browser, you authenticate with your Keycloak account, and the resulting token is saved to `~/.vault-token`. The token is valid for **8 hours**.

To check you're authenticated:

```bash
vault token lookup
```

To log out and revoke the token:

```bash
vault token revoke -self
```

---

## SSH certificate setup

Instead of distributing static authorized keys, Vault acts as an SSH CA. You generate a key pair once, and before each session you ask Vault to sign your public key. The host trusts any certificate signed by the Vault CA — so adding new machines or rotating keys requires no coordination.

- Certificates are valid for **8 hours** (`user` role)
- Every signing request is recorded in the Vault audit log
- Removing your Vault account immediately revokes future access

### macOS

**One-time setup:**

```bash
# 1. Generate a dedicated SSH key pair for Vault-signed access
ssh-keygen -t ed25519 -f ~/.ssh/vault -C "vault-ssh" -N ""

# 2. Add to ~/.ssh/config so SSH knows to use the signed cert for homelab hosts
cat >> ~/.ssh/config << 'EOF'

Host *.home.philippthesurfer.com
    User philipp
    IdentityFile ~/.ssh/vault
    CertificateFile ~/.ssh/vault-signed.pub
EOF
```

**Sign your key (run once per session, after `vault login`):**

```bash
export VAULT_ADDR=https://vault.home.philippthesurfer.com
vault login -method=oidc
vault write -field=signed_key ssh/sign/user \
  public_key="$(cat ~/.ssh/vault.pub)" > ~/.ssh/vault-signed.pub
chmod 600 ~/.ssh/vault-signed.pub
```

**SSH in:**

```bash
ssh minipc.home.philippthesurfer.com
```

---

### Linux

The workflow is identical to macOS.

**One-time setup:**

```bash
ssh-keygen -t ed25519 -f ~/.ssh/vault -C "vault-ssh" -N ""

cat >> ~/.ssh/config << 'EOF'

Host *.home.philippthesurfer.com
    User philipp
    IdentityFile ~/.ssh/vault
    CertificateFile ~/.ssh/vault-signed.pub
EOF
chmod 600 ~/.ssh/config
```

**Sign your key:**

```bash
export VAULT_ADDR=https://vault.home.philippthesurfer.com
vault login -method=oidc
vault write -field=signed_key ssh/sign/user \
  public_key="$(cat ~/.ssh/vault.pub)" > ~/.ssh/vault-signed.pub
chmod 600 ~/.ssh/vault-signed.pub
```

**Automate with a shell function** — add to `.bashrc`/`.zshrc`:

```bash
vault-ssh-refresh() {
  vault login -method=oidc
  vault write -field=signed_key ssh/sign/user \
    public_key="$(cat ~/.ssh/vault.pub)" > ~/.ssh/vault-signed.pub
  chmod 600 ~/.ssh/vault-signed.pub
  echo "SSH cert valid for 8h"
}
```

Then just run `vault-ssh-refresh` at the start of each working session.

---

### Windows

Windows 10/11 ships with OpenSSH. The certificate workflow works natively — no WSL required.

**One-time setup (PowerShell as your regular user):**

```powershell
# 1. Generate SSH key pair
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\vault" -C "vault-ssh" -N '""'

# 2. Add to SSH config
$sshConfig = "$env:USERPROFILE\.ssh\config"
if (-not (Test-Path $sshConfig)) { New-Item $sshConfig -Force }

Add-Content $sshConfig @"

Host *.home.philippthesurfer.com
    User philipp
    IdentityFile ~/.ssh/vault
    CertificateFile ~/.ssh/vault-signed.pub
"@
```

**Sign your key (PowerShell, run once per session):**

```powershell
$env:VAULT_ADDR = "https://vault.home.philippthesurfer.com"
vault login -method=oidc

vault write -field=signed_key ssh/sign/user `
  public_key=(Get-Content "$env:USERPROFILE\.ssh\vault.pub" -Raw).Trim() `
  | Set-Content "$env:USERPROFILE\.ssh\vault-signed.pub"
```

**SSH in:**

```powershell
ssh minipc.home.philippthesurfer.com
```

**Optional — set VAULT_ADDR permanently:**

```powershell
# Adds it to your user environment (persists across sessions)
[System.Environment]::SetEnvironmentVariable("VAULT_ADDR", "https://vault.home.philippthesurfer.com", "User")
```

**WSL2 note:** If you use WSL2, install the Vault CLI inside WSL and follow the Linux instructions. The SSH config inside WSL is separate from Windows' `~/.ssh/config`.

---

## Daily SSH workflow

Every morning (or whenever your token expires):

```bash
# 1. Authenticate to Vault
vault login -method=oidc        # opens browser → Keycloak login → token saved to ~/.vault-token

# 2. Sign your SSH key (8h cert)
vault write -field=signed_key ssh/sign/user \
  public_key="$(cat ~/.ssh/vault.pub)" > ~/.ssh/vault-signed.pub

# 3. SSH into the mini PC
ssh minipc.home.philippthesurfer.com
```

Steps 1–2 can be combined into the `vault-ssh-refresh` shell function shown in the Linux section above.

---

## Managing secrets

All homelab secrets live in a single KV v2 secret at the path `secret/ansible`.  
KV v2 keeps full version history — every `put`/`patch` creates a new version, old versions remain recoverable.

> **Prerequisite:** `vault login -method=oidc` with a valid session.

### View secrets

```bash
# List all fields and their values
vault kv get secret/ansible

# Get a single field
vault kv get -field=namecheap_api_key secret/ansible

# View a specific historical version
vault kv get -version=2 secret/ansible

# List all versions (metadata only)
vault kv metadata get secret/ansible
```

### Add or update a secret field

**`kv put` replaces the entire secret.** Always use `kv patch` to add or change individual fields — otherwise you'll wipe every other key.

```bash
# Add a new field or update an existing one (safe — leaves all other fields untouched)
vault kv patch secret/ansible new_service_password="hunter2"

# Update multiple fields at once
vault kv patch secret/ansible \
  gitlab_oidc_secret="new-secret-here" \
  harbor_oidc_secret="another-new-secret"
```

`kv patch` requires the KV v2 engine, which is already enabled at `secret/`. It will fail on a non-existent path — use `kv put` to create the secret initially (as done during bootstrap), then `kv patch` for all subsequent changes.

### Remove a secret field

There is no single-field delete in KV v2. The workflow is: read → remove the field locally → write back.

```bash
# 1. Export all current values to a temp file
vault kv get -format=json secret/ansible | jq '.data.data' > /tmp/vault-secrets.json

# 2. Remove the unwanted field from the file
# Edit /tmp/vault-secrets.json and delete the key

# 3. Write back from the file (this creates a new version)
vault kv put secret/ansible @/tmp/vault-secrets.json

# 4. Clean up
rm /tmp/vault-secrets.json
```

> **Never leave secrets in temp files longer than necessary.** Consider using `shred` or macOS `srm` to securely delete.

### Delete a secret version

```bash
# Soft-delete (recoverable): marks version as deleted, data still in Vault
vault kv delete secret/ansible                       # deletes latest version
vault kv delete -versions=2,3 secret/ansible        # deletes specific versions

# Permanently destroy versions (unrecoverable)
vault kv destroy -versions=2 secret/ansible

# Permanently delete all versions and metadata (nuclear option)
vault kv metadata delete secret/ansible
```

---

## Using secrets in Ansible

### How it works

```
vault login -method=oidc
        │
        ▼
~/.vault-token  (or $VAULT_TOKEN for CI)
        │
        ▼
ansible-playbook site.yml
        │
        ▼  (at variable resolution time, before any task runs)
community.hashi_vault lookup  ──► GET /v1/secret/data/ansible
        │
        ▼
vault_* variables populated in memory
        │
        ▼
Jinja2 templates rendered with real values
        │
        ▼
docker-compose.yml written to Mini PC  (secrets never hit disk on local machine)
```

The `community.hashi_vault.hashi_vault` lookup plugin (from `ansible/requirements.yml`) makes a single API call to Vault at the start of the play. All `vault_*` variables are derived from that one response — no per-variable round-trips.

### The secrets section of vars.yml

`ansible/group_vars/all/vars.yml` contains a secrets block at the bottom that maps the `vault_*` variable names that templates reference to fields in the HC Vault KV entry. A single lookup fetches all fields at once — no per-variable round-trips:

```yaml
_vault_secrets: "{{ lookup('community.hashi_vault.hashi_vault', 'secret/data/ansible') }}"

vault_minipc_passwd:            "{{ _vault_secrets.data.minipc_passwd }}"
vault_namecheap_api_user:       "{{ _vault_secrets.data.namecheap_api_user }}"
# ... (full list in vars.yml)
```

The lookup URL and auth method are configured in `ansible/ansible.cfg`:

```ini
[hashi_vault_collection]
url = https://vault.home.philippthesurfer.com
auth_method = token
```

`auth_method = token` means the plugin reads from `VAULT_TOKEN` env var first, then falls back to `~/.vault-token`.

> **Note on `vault_hcvault_root_token`:** this variable is intentionally absent from the lookup list. The `hcvault` configure tasks pass the operator's `VAULT_TOKEN` env var directly into `docker exec` — using the same token Ansible already needs to run the play. Storing the root token inside Vault itself would be circular.

### Running playbooks

```bash
# One-time: install the hashi_vault collection
ansible-galaxy collection install -r ansible/requirements.yml

# Every session before running playbooks
export VAULT_ADDR=https://vault.home.philippthesurfer.com
vault login -method=oidc     # sets ~/.vault-token; valid 8h

# Run a playbook — no --ask-vault-pass, no secrets in the command
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml --tags traefik

# Run everything
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml
```

If you are already on the LAN the Vault URL resolves via Pi-hole. If you are remote, connect via Headscale VPN first.

### CI/CD via AppRole

GitLab CI doesn't do interactive OIDC logins. It authenticates with an AppRole (already configured by the `hcvault` Ansible role):

```bash
# One-time: retrieve the AppRole credentials and store them as
# GitLab CI/CD secret variables (Settings → CI/CD → Variables):
#   VAULT_ROLE_ID   (not masked — it's not sensitive)
#   VAULT_SECRET_ID (masked)

vault read -field=role_id auth/approle/role/ci-cd/role-id
vault write -field=secret_id -f auth/approle/role/ci-cd/secret-id
```

In `.gitlab-ci.yml`:

```yaml
before_script:
  - export VAULT_ADDR=https://vault.home.philippthesurfer.com
  - export VAULT_TOKEN=$(vault write -field=token auth/approle/login
      role_id="$VAULT_ROLE_ID"
      secret_id="$VAULT_SECRET_ID")

deploy:
  script:
    - ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml
```

The `ci-cd` AppRole has read-only access to `secret/data/ansible` and `secret/data/ci-cd/*`. It cannot write or delete secrets.

### Adding a new secret

Say you're adding a new service `ntfy` with an admin password. The full workflow:

**1. Add the value to Vault**

```bash
vault kv patch secret/ansible ntfy_admin_password="$(openssl rand -base64 32)"
```

**2. Add the mapping to `vars.yml`**

```yaml
# ansible/group_vars/all/vars.yml — add one line to the secrets block:
vault_ntfy_admin_password: "{{ _vault_secrets.data.ntfy_admin_password }}"
```

**3. Reference it in your Jinja2 template**

```yaml
# services/ntfy/docker-compose.yml.j2
environment:
  NTFY_AUTH_DEFAULT_ACCESS: "deny-all"
  NTFY_AUTH_FILE: /var/lib/ntfy/user.db
  NTFY_ADMIN_PASSWORD: "{{ vault_ntfy_admin_password }}"
```

**4. Nothing else changes.** The single `_vault_secrets` lookup already fetches the entire `secret/ansible` dict — the new key is included automatically on the next Ansible run.

---

## Quick reference

| Task | Command |
|---|---|
| Log in | `vault login -method=oidc` |
| Sign SSH cert | `vault write -field=signed_key ssh/sign/user public_key="$(cat ~/.ssh/vault.pub)" > ~/.ssh/vault-signed.pub` |
| View all secrets | `vault kv get secret/ansible` |
| Add/update a field | `vault kv patch secret/ansible key="value"` |
| Get one field | `vault kv get -field=key secret/ansible` |
| Version history | `vault kv metadata get secret/ansible` |
| Get old version | `vault kv get -version=N secret/ansible` |
| Soft-delete latest | `vault kv delete secret/ansible` |
| Destroy a version | `vault kv destroy -versions=N secret/ansible` |
| Check token | `vault token lookup` |
| Revoke token | `vault token revoke -self` |
| AppRole role-id | `vault read -field=role_id auth/approle/role/ci-cd/role-id` |
| AppRole secret-id | `vault write -field=secret_id -f auth/approle/role/ci-cd/secret-id` |
