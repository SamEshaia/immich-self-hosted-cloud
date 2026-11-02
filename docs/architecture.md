# Architecture

## Network Path

One public URL serves LAN users and travelers. Traffic ends at nginx on the VPS and crosses the wg1 tunnel to the Immich VM. Nothing listens inbound at home.

```
Phones / browsers (anywhere)
        │ HTTPS
        ▼
┌──────────────────────────────────────────────┐
│ VPS <vps_public_ip>                          │
│ nginx :80/:443 — certbot (LE renewal timer)  │
│ client_max_body_size 50000M, buffering off   │
│ :51821/udp — wg1 listen                      │
└──────────────┬───────────────────────────────┘
               │ wg1: <wg1_vps_ip>/24 ↔ <wg1_vm_ip>/24, MTU 1420
               │ keepalive 25s from the VM side
┌──────────────▼───────────────────────────────┐
│ KVM host <kvm_host_name>                     │
│  virsh domain: <vm_name>                     │
│  ├─ virtio disk vda: 600G (LV on RAID6 VG)   │
│  ├─ Immich compose :2283 (server, postgres,  │
│  │   ML, valkey) — files in /srv/immich      │
│  └─ Tailscale <vm_tailnet_ip> (admin)        │
└──────────────────────────────────────────────┘

Rollback (kept warm): photos-cf.<domain> → CF edge → cloudflared → :2283
```

## Addressing

| Endpoint | Address | Role |
|---|---|---|
| VPS public | `<vps_public_ip>` | nginx, target of public DNS |
| VPS wg1 | `<wg1_vps_ip>/24`, :51821/udp | tunnel endpoint |
| VPS wg0 | `<wg0_vps_ip>/24`, :51820/udp | pre-existing game-server tunnel, untouched |
| KVM host wg0 | `<wg0_home_ip>/24` | pre-existing, untouched |
| Immich VM LAN | `<vm_ip>/24` (bridge1, MAC `<vm_mac>`) | Docker host |
| Immich VM wg1 | `<wg1_vm_ip>/24` | tunnel peer, dials out with keepalive |
| Immich VM tailnet | `<vm_tailnet_ip>` | Tailscale admin access |

Pick the wg1 /24 so it doesn't overlap wg0 or the home LANs.

## Ports

| Port | Service | Bound | Exposure |
|------|---------|-------|----------|
| 80   | nginx (HTTP redirect + ACME challenge) | VPS | public |
| 443  | nginx TLS (Immich) | VPS | public |
| 51821 | wg1 | VPS | public, UDP |
| 51820 | wg0 (pre-existing) | VPS | public, UDP |
| 2283 | Immich web | VM | LAN + tunnel only |
| 22   | SSH | both | LAN/Tailscale only |

The home router publishes nothing. The VPS listens only on nginx + its two WireGuard ports.

## TLS Chain

- Browser ↔ VPS: Let's Encrypt certificate for `photos.<domain>`, certbot nginx plugin
- VPS ↔ VM: plain HTTP — the payload of the encrypted WireGuard tunnel; nginx adds `X-Forwarded-*` headers
- Immich trusts the tunnel peer only: `IMMICH_TRUSTED_PROXIES=<wg1_vps_ip>`

## Why a Second WireGuard Tunnel

Reusing the existing game-server tunnel (wg0) would have needed an `AllowedIPs` change, a route, and iptables MASQUERADE + MSS clamping on the home KVM host — the box running live game servers. A dedicated tunnel avoids all of that:

- **Nothing changes on the KVM host** — no iptables, no wg0 edits; the router sees one more outbound UDP flow
- **No NAT or MSS clamp needed** — MTU 1420 set at creation on both ends
- **Isolation** — wg1 can't break wg0, and vice versa
- **No key expiry or re-auth** — static wg-quick systemd units

## IP-Change Resilience

Public DNS points at the VPS. The tunnels dial outward from home, so an ISP IP change costs a few seconds while wg1/Tailscale re-establish — no DDNS anywhere in this deployment.

## Boot Chain

All self-starts after a power cut: docker enabled on the VM (four Immich containers), `wg-quick@wg1` enabled, `cloudflared.service` enabled as fallback, nginx + certbot + wg1 on the VPS already running.

## What's Exposed vs. Internal

**Public internet sees:**
- The Immich login page at `photos.<domain>`
- nginx on the VPS — that box absorbs all scanner traffic

**Stays internal:**
- SSH (LAN/Tailscale only)
- Cockpit on the KVM host (:9090)
- Immich LAN endpoint, Postgres, backups

## Rollback

The Cloudflare tunnel service stays installed and enabled on the VM; the `photos-cf` DNS record stays live. Restoring the old path is two DNS renames and a TTL wait.