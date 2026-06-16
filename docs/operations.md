# Operations

Day-to-day reference for running and maintaining the homelab.

---

## Update a Service Version

1. Edit the version in `ansible/group_vars/all/vars.yml`
2. Push to GitLab → CI re-deploys the affected stack automatically
3. Or manually: `ansible-playbook ansible/site.yml --tags <service>`

---

## Restart a Stack

```bash
ssh philipp@192.168.178.20
docker compose -f /opt/homelab/services/<service>/docker-compose.yml restart
```

For Harbor (installer-managed):
```bash
cd /opt/homelab/harbor
docker compose -f docker-compose.yml -f docker-compose.override.yml restart
```

---

## View Logs

```bash
ssh philipp@192.168.178.20
docker compose -f /opt/homelab/services/<service>/docker-compose.yml logs -f
```

Or tail a single container:
```bash
docker logs -f <container-name>
```

---

## Rotate a Secret

1. Generate a new value locally
2. Update it in Vault: `vault kv patch secret/ansible <field>=<new-value>`
3. Re-run the affected role: `ansible-playbook ansible/site.yml --tags <service>`

---

## Unseal HashiCorp Vault (after restart)

Vault seals itself whenever the container restarts. Unseal with 3 of the 5 unseal keys (stored in Vaultwarden):

```bash
docker exec -it vault vault operator unseal
```

Run three times with three different keys. Check status:

```bash
docker exec -it vault vault status
```

---

## SSH into the Mini PC

Vault-signed SSH cert is required. Cert is valid 8 hours.

```bash
# Authenticate and sign (run once per session)
export VAULT_ADDR=https://vault.home.philippthesurfer.com
vault login -method=oidc
vault write -field=signed_key ssh/sign/user \
  public_key="$(cat ~/.ssh/hcvault.pub)" > ~/.ssh/hcvault-cert.pub

# SSH in
ssh minipc.home.philippthesurfer.com
```

See [vault.md](vault.md) for full SSH setup and per-OS instructions.

---

## Register a New Tailscale Device

1. Open Headplane at `https://headscale.home.philippthesurfer.com/admin`
2. Go to **Pre-auth Keys** → create a key (single-use recommended)
3. On the new device:
   ```bash
   tailscale up --login-server=https://vpn.philippthesurfer.com --authkey=<key>
   ```

---

## Push a Docker Image to Harbor

Log in with your OIDC CLI secret (not your Keycloak password directly):

1. In Harbor UI → click your username → **User Profile** → copy **CLI secret**
2. ```bash
   docker login harbor.home.philippthesurfer.com
   # username: <keycloak username>
   # password: <CLI secret from Harbor UI>
   
   docker tag myimage:latest harbor.home.philippthesurfer.com/<project>/myimage:latest
   docker push harbor.home.philippthesurfer.com/<project>/myimage:latest
   ```

The Docker registry endpoint (`registry.home.philippthesurfer.com`) also works for `docker login/push/pull` and points to the same Harbor instance.

---

## Add a DNS Record for a New Service

Pi-hole handles internal DNS. Add a record in one of two ways:

**Via Pi-hole admin UI:**  
`http://192.168.178.20:8080` → Local DNS → DNS Records → add `<name>.home.philippthesurfer.com` → `192.168.178.20`

**Via config file (persists across redeploys):**  
Append to `services/pihole/etc-pihole/custom.list.j2` (Jinja2 template), then re-run the pihole role:
```bash
ansible-playbook ansible/site.yml --tags pihole
```

---

## Add a New Service (full workflow)

1. Create `services/<name>/docker-compose.yml.j2` (Jinja2 template with Traefik labels)
2. Create `ansible/roles/<name>/tasks/main.yml`:
   - Creates target directory on Mini PC
   - Renders template via `template:` module
   - Runs `docker compose up -d`
3. Add any new secrets to Vault:
   ```bash
   vault kv patch secret/ansible <field>="$(openssl rand -base64 32)"
   ```
4. Add the `vault_*` mapping to `ansible/group_vars/all/vars.yml`
5. Add role to `ansible/site.yml` with a matching tag
6. Create OIDC client in Keycloak if the service supports SSO
7. Add Pi-hole DNS record (see above)

---

## Check GitLab Runner Status

```bash
ssh philipp@192.168.178.20
docker exec -it gitlab-runner gitlab-runner list
docker exec -it gitlab-runner gitlab-runner verify
```

If the runner is stuck or needs re-registration:
```bash
docker exec -it gitlab-runner gitlab-runner register \
  --url https://gitlab.home.philippthesurfer.com \
  --executor docker \
  --docker-image alpine
```

---

## Force Cert Renewal (Let's Encrypt)

Traefik renews automatically ~30 days before expiry. To force an immediate renewal:

```bash
# Delete the acme.json volume entry for the domain, then restart Traefik
ssh philipp@192.168.178.20
docker restart traefik
```

Traefik will attempt renewal on startup. Check logs:
```bash
docker logs traefik | grep -i acme
```

---

## Troubleshooting

**Can't reach a service at `*.home.philippthesurfer.com`**
- Confirm your device is on the LAN or connected via Headscale VPN
- Check Pi-hole has a DNS record for the hostname pointing to `192.168.178.20`
- Check Traefik dashboard (`https://traefik.home.philippthesurfer.com`) — verify the router is registered

**Let's Encrypt cert not issuing**
- Check Traefik logs: `docker logs traefik | grep -i acme`
- Verify Namecheap API key is correct and the home IP is whitelisted in Namecheap API settings
- Set `letsencrypt_staging: true` in `group_vars/all/vars.yml` and re-deploy to test without hitting rate limits
- See [ddns.md](ddns.md) for the full diagnostic checklist

**Headscale devices can't connect remotely**
- Check ddclient logs: `docker logs ddclient` — confirm A record is current
- Verify `vpn.philippthesurfer.com` resolves to your home IP: `dig vpn.philippthesurfer.com +short`
- Check FritzBox port forwarding: TCP 443 → `192.168.178.20`
- See [ddns.md](ddns.md) for the full diagnostic checklist

**Keycloak OIDC login failing**
- Verify the redirect URI in the Keycloak client matches the service URL exactly
- Check client secret in Vault matches what Keycloak shows under Credentials
- Re-run the affected role after any Vault change

**HashiCorp Vault sealed after reboot**
- Vault seals itself on restart by design — unseal manually with 3 of 5 keys (stored in Vaultwarden):
  ```bash
  docker exec -it vault vault operator unseal
  ```

**GitLab takes forever to start**
- GitLab CE typically takes 3–5 minutes on first boot while it initializes the database and compiles assets. Monitor with:
  ```bash
  docker logs -f gitlab
  ```
