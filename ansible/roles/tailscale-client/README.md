# tailscale-client

Installs and configures the Tailscale client on the Mini PC as a subnet router.

## Overview

The Mini PC joins the Headscale mesh as a node and advertises the home LAN subnet (`192.168.178.0/24`) to all other peers. This makes Traefik at `192.168.178.20` reachable from any device connected to the VPN — which is required for split DNS and `*.home.philippthesurfer.com` to work remotely.

## Prerequisites

- `headscale` role running (`vpn.philippthesurfer.com` reachable)

## Containers

None. Tailscale runs as a native systemd service (`tailscaled`).

## Configuration

The role only installs Tailscale and starts `tailscaled`. Authentication and subnet advertisement are done manually after deploy (see Deploy section).

## Secrets

None. Authentication is performed interactively on the machine.

## URL

No web UI. Manage the node via Headplane at `https://headscale.home.philippthesurfer.com/admin`.

## Deploy

**Step 1 — install Tailscale:**

```bash
ansible-playbook ansible/site.yml --tags tailscale-client
```

**Step 2 — authenticate on the Mini PC:**

```bash
ssh philipp@192.168.178.20
tailscale up --login-server=https://vpn.philippthesurfer.com \
             --advertise-routes=192.168.178.0/24 \
             --hostname=minipc
# Prints a URL — open it on any device and log in with your Keycloak account
```

**Step 3 — approve the subnet route:**

```bash
docker exec -it headscale headscale routes list
docker exec -it headscale headscale routes enable -r <route-id>
```

**Step 4 — enable route acceptance on remote devices:**

```bash
tailscale up --accept-routes
```

## Notes

**Route approval is manual.** Headscale requires explicit approval of advertised routes. This is a one-time step and survives re-deploys.
