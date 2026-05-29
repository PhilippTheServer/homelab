## Day-to-Day

### Update a service version

1. Edit the version in `ansible/group_vars/all/vars.yml`
2. Push to GitLab → CI re-deploys the affected stack
3. Or manually: `ansible-playbook ansible/site.yml --tags <service>`

### Restart a stack

```bash
ssh deploy@192.168.178.20
docker compose -f /opt/homelab/services/<service>/docker-compose.yml restart
```

### View logs

```bash
ssh deploy@192.168.178.20
docker compose -f /opt/homelab/services/<service>/docker-compose.yml logs -f
```

### Rotate a secret

1. `vault kv patch secret/ansible <field>=<new-value>` — update in HashiCorp Vault
2. Re-run the role: `ansible-playbook ansible/site.yml --tags <service>`
