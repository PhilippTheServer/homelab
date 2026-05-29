# Homelab — Claude Code Rules

## Secrets

All secrets live in HashiCorp Vault at `secret/ansible`. No exceptions.

- Never write a secret value into a file, template, playbook, or variable file
- Secrets are referenced in `ansible/group_vars/all/vars.yml` as `vault_*` variables mapped from `_vault_secrets`
- To add a new secret: `vault kv patch secret/ansible <field>="..."`, then add the `vault_<field>` mapping to `vars.yml`
- Never commit `.env` files, plaintext credentials, or any file containing a real secret value

## Infrastructure Changes

All changes are applied through Ansible playbooks. No manual configuration on the Mini PC.

- Changes to service config → edit the Jinja2 template in `services/<name>/`, re-run the role
- Changes to host-level config → edit the role tasks, re-run the role
- Never SSH in and edit files directly as a fix — make the change in the repo and deploy it
- To apply: `ansible-playbook ansible/site.yml --tags <role>`

## New Ansible Roles

Every new role must have a `README.md` in `ansible/roles/<name>/README.md`.

Follow the structure of the existing role READMEs exactly:

```
# <role-name>

One-line description.

## Overview
## Prerequisites
## Containers
## Configuration
## Secrets
## URL
## Deploy
## Notes
```

## Documentation

Keep documentation current at all times. Update docs in the same change as the code — never after.

- **New service added** → update `docs/architecture.md` (service stacks section + repo layout + bootstrap order), update `README.md` stack table, add role README
- **Service removed** → remove from all of the above
- **URL changed** → update `README.md` stack table and the role README
- **New secret added** → update `docs/bootstrap.md` vault kv put command and the role README secrets table
- **Operational procedure changed** → update `docs/operations.md`
- **DDNS / cert / network change** → update `docs/ddns.md`
- **Vault / SSH workflow changed** → update `docs/vault.md`

The documentation lives in:
```
README.md                  — entrypoint, stack table, setup overview
docs/architecture.md       — network diagram, service stacks, traffic flows
docs/bootstrap.md          — first deploy guide
docs/ddns.md               — DDNS + TLS cert setup
docs/vault.md              — Vault CLI, SSH certs, secrets management
docs/operations.md         — day-to-day ops reference
ansible/roles/*/README.md  — per-role reference
```
