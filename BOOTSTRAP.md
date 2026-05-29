## Bootstrap

### Prerequisites (control machine)

```bash
# macOS: prevent "A worker was found in a dead state" fork-safety crash
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES  # add to ~/.zshrc to make permanent

# Install hvac into Ansible's Python — required for community.hashi_vault lookups
# Pin to 1.x: hvac 2.x changed response types, breaking community.hashi_vault 7.x lookups
/opt/homebrew/Cellar/ansible/*/libexec/bin/python -m pip install "hvac"

# Install Ansible collections
ansible-galaxy collection install -r ansible/requirements.yml
```

### Deployment order

Infrastructure must be bootstrapped in dependency order. Priority reflects architectural importance; deployment order reflects technical dependencies.

```
[infrastructure]  common     → Docker, UFW, deploy user (prerequisite for everything)
[infrastructure]  traefik    → SSL and routing (prerequisite for all HTTPS services)

[priority 1]      keycloak   → SSO — must exist before any service is wired to it

[priority 2]      headscale  → VPN — wire Keycloak OIDC, then port forward goes live

[priority 3]      vault      → HashiCorp Vault — secrets automation
[priority 3]      vaultwarden → Password manager

[priority 4]      pihole     → Internal DNS (or migrate existing Pi-hole to IaC)
[priority 4]      gitlab     → Push this repo here; CI takes over re-deploys
[priority 4]      harbor     → Registry
```

<details>
<summary>exec commands for deployments</summary>

```bash
# 0. Verify connectivity
ansible minipc -m ping -i ansible/inventory/hosts.yml --ask-vault-pass

# 1. Provision host
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags common --ask-vault-pass

# 2. Traefik (SSL + DDNS prerequisite)
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags traefik --ask-vault-pass

# 3. Keycloak
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags keycloak --ask-vault-pass

# 4. Headscale (wire Keycloak OIDC first — see post-bootstrap below)
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags headscale --ask-vault-pass

# 5. HashiCorp Vault
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags hcvault --ask-vault-pass

# 6. Vaultwarden
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags vaultwarden --ask-vault-pass

# 7. Pi-hole (migrate existing or fresh deploy)
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags pihole --ask-vault-pass

# 8. GitLab + Harbor
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags gitlab,harbor --ask-vault-pass
```

</details>

### Pre-bootstrap: Namecheap + FritzBox (one-time, before deploying Traefik)

#### Namecheap

| Step | Where | Action |
|------|-------|--------|
| Enable DDNS | Domain List → Manage → Advanced DNS → Dynamic DNS | Toggle on, note the DDNS password |
| Enable API | Profile → Tools → API Access | Enable, whitelist home IP (`curl -4 -s https://icanhazip.com`) |
| Add A record | Domain List → Manage → Advanced DNS | Type: A, Host: `vpn`, Value: current home IP |

#### FritzBox

Navigate to **http://fritz.box → Internet → Permit Access → Port Sharing**, target `192.168.178.20`:

| Protocol | Port | Purpose |
|----------|------|---------|
| TCP | 443 | Traefik HTTPS / Headscale coordination |
| TCP | 80 | Let's Encrypt HTTP fallback |
| UDP | 3478 | STUN for Tailscale NAT traversal |

FritzBox automatically creates both an IPv4 NAT rule and an IPv6 firewall allow rule from each port sharing entry.

### Post-bootstrap: wire SSO

After Keycloak is running, create OIDC clients for each service before deploying them:

1. Log into `auth.philippthesurfer.com` → create realm `homelab`
2. Create OIDC clients: `headscale`, `gitlab`, `harbor`, `vaultwarden`
3. Add each client secret to Vault:
   ```bash
   ansible-vault edit ansible/group_vars/all/vault.yml
   # fill in vault_*_oidc_secret fields
   ```
4. Re-run the affected roles to apply SSO config

### Post-bootstrap: initialize HashiCorp Vault

HashiCorp Vault requires a one-time init after first deploy:

```bash
ssh deploy@192.168.178.20
docker exec -it vault vault operator init

# Save the 5 unseal keys and root token in Vaultwarden immediately
# Vault needs 3 of 5 keys to unseal after every restart
docker exec -it vault vault operator unseal  # run 3 times with different keys
```

Consider configuring auto-unseal later (e.g. using a cloud KMS or a dedicated key) to avoid manual unsealing after reboots.

### Post-bootstrap: migrate this repo to GitLab

```bash
git remote add homelab https://gitlab.home.philippthesurfer.com/<user>/homelab.git
git push homelab main
```

From this point, GitLab CI handles re-deploys on push.

---
