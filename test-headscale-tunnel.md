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
Cloudflare edge          ← TLS termination happens here (Cloudflare negotiates h2/h3)
    │ Cloudflare Tunnel
    ▼
cloudflared container
    │ HTTP  http://headscale:8080  (Docker proxy network)
    ▼
headscale container      ← Traefik is NOT in this path
```

---

## Root cause (confirmed 2026-05-29)

Cloudflare negotiates **HTTP/2** with external clients. The `Upgrade: websocket` header
is an HTTP/1.1 construct — RFC 7540 explicitly forbids it in H2. HTTP/2 uses a
different WebSocket mechanism (`CONNECT` with `:protocol: websocket` per RFC 8441).
Cloudflare does **not** translate HTTP/2 WebSocket semantics back to HTTP/1.1
`Upgrade` headers when forwarding through the tunnel, so headscale receives the
request with no `Upgrade` header and returns 500.

Confirmed by:
```
External (HTTP/2 via Cloudflare): HTTP/2 500  + headscale: "No Upgrade header in TS2021 request"
Internal (HTTP/1.1 via Traefik):  HTTP/1.1 101 Switching Protocols  ✅
```

The `no-h2` Traefik TLS option that fixes LAN connections has zero effect on external
traffic — Cloudflare terminates the TLS independently before the tunnel.

---

## Fix — disable HTTP/2 at the Cloudflare zone level

Cloudflare dashboard → **philippthesurfer.com → Speed → Optimization → Protocol Optimization**

| Setting | Value |
|---------|-------|
| HTTP/2  | **Off** |
| HTTP/3 (QUIC) | **Off** |

This forces external clients to negotiate HTTP/1.1, preserving the `Upgrade: websocket`
header end-to-end through the tunnel to headscale.

**Verify the fix** (from external network):
```bash
# Must return 101, not 500
curl --http1.1 -sv \
  -H "Upgrade: websocket" \
  -H "Connection: Upgrade" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  https://vpn.philippthesurfer.com/ts2021 2>&1 | grep "< HTTP"
```

---

## Layer-by-layer diagnostics

Run these if the fix above doesn't resolve the issue, or to diagnose a regression.

### Layer 1 — Is the tunnel receiving the request at all?

Trigger the error from an external network, then immediately on the server:

```bash
docker logs cloudflared --tail=30
```

**Connection logged** → tunnel is alive, skip to Layer 3.  
**Nothing logged** → Cloudflare's edge is returning the error before the tunnel. Check:
- **philippthesurfer.com → Network → WebSockets → On**
- Zero Trust → Networks → Tunnels → tunnel status is **Healthy**
- Speed → Optimization → HTTP/2 is **Off** (see fix above)

---

### Layer 2 — Can cloudflared reach headscale?

```bash
docker exec cloudflared wget -qO- http://headscale:8080/key?v=138
```

**Returns JSON** → connectivity fine, skip to Layer 3.  
**Connection refused / empty** → cloudflared cannot resolve `headscale` on the Docker network.

```bash
# Both containers must appear in the proxy network
docker network inspect proxy | grep -A3 '"Name"'
```

---

### Layer 3 — Does headscale receive the request?

Watch headscale logs while triggering the error from outside:

```bash
docker logs headscale -f --tail=0
```

| Headscale log says | Meaning |
|--------------------|---------|
| `No Upgrade header in TS2021 request` | Request reaches headscale but WebSocket headers were stripped — HTTP/2 issue (see fix above) |
| `noise upgrade failed: Unexpected subprotocol ""` | WebSocket upgrade reached headscale but was missing `Sec-WebSocket-Protocol` — client/proxy header issue |
| Nothing | cloudflared is failing before forwarding — check cloudflared logs |

---

### Layer 4 — Isolate HTTP/2 vs WebSocket

From an external machine:

```bash
# Step 1: plain GET — must return 200
curl -sv https://vpn.philippthesurfer.com/key?v=138 2>&1 | grep "< HTTP"

# Step 2: WebSocket over whatever protocol curl negotiates — likely 500 if H2 is on
curl -sv \
  -H "Upgrade: websocket" \
  -H "Connection: Upgrade" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  https://vpn.philippthesurfer.com/ts2021 2>&1 | grep "< HTTP"

# Step 3: same but force HTTP/1.1 — must return 101 if tunnel is working
curl --http1.1 -sv \
  -H "Upgrade: websocket" \
  -H "Connection: Upgrade" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  https://vpn.philippthesurfer.com/ts2021 2>&1 | grep "< HTTP"
```

| Step 1 | Step 2 | Step 3 | Diagnosis |
|--------|--------|--------|-----------|
| 200 | 500 | 101 | HTTP/2 stripping Upgrade headers → disable H2 at zone level |
| 200 | 500 | 500 | Tunnel or headscale issue, not H2 related |
| 500 | — | — | Tunnel not working for any traffic — check dashboard route |

---

### Layer 5 — Cloudflare Tunnel dashboard route

In **Zero Trust → Networks → Tunnels → your tunnel → Edit → Public Hostname**:

| Field | Required value |
|-------|---------------|
| Subdomain | `vpn` |
| Domain | `philippthesurfer.com` |
| Type | `HTTP` |
| URL | `headscale:8080` |

`headscale:8080` works because cloudflared and headscale share the `proxy` Docker
network. Do **not** use `traefik:80` — Traefik 301-redirects to HTTPS, cloudflared
follows back to Cloudflare's edge, request loops, Cloudflare returns 500.

---

## Quick reference — all checks at once

```bash
# Terminal 1 (server): watch cloudflared
docker logs cloudflared -f --tail=0

# Terminal 2 (server): watch headscale
docker logs headscale -f --tail=0

# Terminal 3 (external): plain GET
curl -sv https://vpn.philippthesurfer.com/key?v=138 2>&1 | grep "< HTTP"

# Terminal 4 (external): WebSocket over HTTP/1.1 — the definitive test
curl --http1.1 -sv \
  -H "Upgrade: websocket" \
  -H "Connection: Upgrade" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  https://vpn.philippthesurfer.com/ts2021 2>&1 | grep -E "< HTTP|Switching|error"
```
