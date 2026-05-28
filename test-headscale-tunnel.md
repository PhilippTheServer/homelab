# Headscale Cloudflare Tunnel Debug Runbook

Use this when external Tailscale clients fail to connect through the Cloudflare Tunnel
while LAN clients work fine.

**Symptom:**
```
register request: Post "https://vpn.philippthesurfer.com/machine/register":
all connection attempts failed (HTTP: TLS forced: no port 80 dialed,
HTTPS: unexpected HTTP response: 500 Internal Server Error)
```
Traefik access log shows nothing — the request never reaches it. This is expected:
the tunnel routes directly `cloudflared → headscale:8080`, bypassing Traefik.

---

## Architecture

```
External client
    │ HTTPS (443)
    ▼
Cloudflare edge          ← TLS termination happens here
    │ Cloudflare Tunnel
    ▼
cloudflared container
    │ HTTP  http://headscale:8080  (Docker proxy network)
    ▼
headscale container      ← Traefik is NOT in this path
```

---

## Layer 1 — Is the tunnel receiving the request at all?

Trigger the error from an external network, then immediately on the server:

```bash
docker logs cloudflared --tail=30
```

**200 / connection logged** → tunnel is alive, skip to Layer 3.  
**Nothing logged** → Cloudflare's edge is returning 500 before the tunnel. Check:
- Cloudflare dashboard → **philippthesurfer.com → Network → WebSockets → On**
- Zero Trust → Networks → Tunnels → confirm tunnel status is **Healthy**

---

## Layer 2 — Can cloudflared reach headscale?

```bash
docker exec cloudflared wget -qO- http://headscale:8080/key?v=138
```

**Returns JSON with a public key** → connectivity is fine, skip to Layer 3.  
**Connection refused / empty** → cloudflared cannot resolve `headscale` on the Docker network.

```bash
# Confirm both containers share the proxy network
docker network inspect proxy | grep -A3 '"Name"'
```

Both `cloudflared` and `headscale` must appear. If not, check the `networks:` section
in both docker-compose files.

---

## Layer 3 — Does headscale receive the request?

Watch headscale logs while triggering the error from outside:

```bash
docker logs headscale -f --tail=0
```

**Lines appear with a `172.18.x.x` source IP** → headscale receives the request.
The error is in headscale's own handling — check the log line for the specific error.

**Nothing appears** → cloudflared is failing before forwarding. Check cloudflared logs
for backend errors:

```bash
docker logs cloudflared --tail=50 | grep -i "error\|fail\|502\|503\|500"
```

---

## Layer 4 — Is it WebSocket-specific or all traffic?

From an external machine, test the simplest endpoint first (plain GET, no upgrade):

```bash
curl -v https://vpn.philippthesurfer.com/key?v=138
```

| Result | Meaning |
|--------|---------|
| `200` with JSON body | Tunnel works for plain HTTP. Problem is WebSocket/noise upgrade only. |
| `500` | Problem is before any protocol negotiation — tunnel or headscale unreachable. |
| `SSL` / connection error | DNS or Cloudflare proxy not configured for this hostname. |

If `200` here but Tailscale still fails, test the upgrade path:

```bash
curl -v \
  -H "Upgrade: websocket" \
  -H "Connection: Upgrade" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  https://vpn.philippthesurfer.com/ts2021
```

**Expected:** `101 Switching Protocols`  
**Got `500`:** Cloudflare is blocking the WebSocket upgrade →
confirm WebSockets are **On** in the Cloudflare zone (Layer 1 check above).  
**Got `405`:** The upgrade reached headscale but used the wrong method.

---

## Layer 5 — Cloudflare Tunnel dashboard route

In **Zero Trust → Networks → Tunnels → your tunnel → Edit → Public Hostname**:

| Field | Required value |
|-------|---------------|
| Subdomain | `vpn` |
| Domain | `philippthesurfer.com` |
| Type | `HTTP` |
| URL | `headscale:8080` |

`headscale:8080` works because cloudflared and headscale share the `proxy` Docker
network. Do **not** use `traefik:80` — Traefik would 301-redirect to HTTPS, cloudflared
would follow back to Cloudflare's edge, and the request would loop until Cloudflare
returns 500.

---

## Quick reference — all checks at once

Run these in one pass from the server while an external client triggers the error:

```bash
# Terminal 1: watch cloudflared
docker logs cloudflared -f --tail=0

# Terminal 2: watch headscale
docker logs headscale -f --tail=0

# Terminal 3: test plain HTTP through the tunnel (run from external)
curl -sv https://vpn.philippthesurfer.com/key?v=138 2>&1 | grep -E "< HTTP|error|connected"

# Terminal 4: test WebSocket upgrade through the tunnel (run from external)
curl -sv \
  -H "Upgrade: websocket" \
  -H "Connection: Upgrade" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  https://vpn.philippthesurfer.com/ts2021 2>&1 | grep -E "< HTTP|Switching|error"
```
