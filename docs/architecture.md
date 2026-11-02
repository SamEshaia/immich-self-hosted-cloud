# Architecture

## What serves what

A single HTTPS URL (`photos.<domain>`) serves both LAN users and travelers. The public face is a small VPS running nginx; Immich lives at home in a VM.

```
Phone (anywhere) ──HTTPS──▶ VPS nginx ──wg1 tunnel──▶ Immich VM (docker)
Browser (LAN)    ──HTTPS──▶        │
                                 └── no upload cap, LE TLS, streams through
```

## Addressing table

> All addresses in this table are fictional placeholders (see the note in the README). The topology and subnet scheme are real; substitute your own values when reproducing.

| Endpoint | Address | Role |
|---|---|---|
| VPS public IP | `203.0.113.10` | nginx + LE cert, public DNS target |
| VPS wg1 | `10.66.71.1/24` | Immich tunnel endpoint, :51821/udp |
| Home host wg0 | `10.66.70.2/24` | pre-existing dayz↔VPS tunnel (untouched) |
| VPS wg0 | `10.66.70.1/24`, :51820/udp | pre-existing, isolated from wg1 |
| Immich VM LAN | `192.168.83.50/24` (bridge1, MAC `52:54:00:00:31:50`) | Docker host, LAN access |
| Immich VM tailnet | `100.64.7.115` | Tailscale admin access |
| Immich VM wg1 | `10.66.71.2/24` | tunnel peer, dials out with keepalive 25 |

## Ports

| Box | Port | Purpose |
|---|---|---|
| VPS | 80/tcp, 443/tcp | nginx (public) |
| VPS | 51821/udp | wg1 (WireGuard for Immich) |
| VPS | 51820/udp | wg0 (pre-existing) |
| Home router | **none opened** | everything dials out |
| Immich VM | 2283 | Immich web UI (LAN + docker) |

## Why a dedicated second tunnel (wg1) instead of reusing wg0

The original plan layered Immich onto the existing game-server tunnel by adding the VM subnet to wg0's `AllowedIPs` plus iptables MASQUERADE + MSS clamping on the home KVM host. That pays for itself by requiring NAT and policy edits on the busiest box in the house. The dedicated tunnel instead:

- **Zero changes to the dayz host** — no iptables, no MASQUERADE, no MSS clamp, no edits to wg0 peers. The home router sees one more outbound UDP flow, same as the Tailscale flow the VM already makes.
- **No NAT in the path, no clamp needed** — MTU 1420 is set on both wg1 ends at creation, so large streaming uploads have nothing to trip on.
- **Isolation** — a broken wg1 cannot affect wg0, and vice versa. The game servers keep their own path.
- **Same reliability rationale that chose WireGuard**: wg-quick systemd units, no key expiry, no periodic re-auth.

## Two tunnels, two transport lessons

The project hit both transport classes of upload stall:

1. **Cloudflare QUIC + phone VPN** — uploads through the CF tunnel died with iOS `-1001`/`-1005` errors whenever a client VPN was on: double tunneling (QUIC-in-VPN) starves the stream. Fixed by `protocol: http2` in the tunnel config; kept in the fallback config. See [troubleshooting](troubleshooting.md).
2. **Cloudflare's hard 100MB request cap** — not a config bug at all, a plan limit. Fixed by the nginx/VPS path; documented in [upload-cap-migration](upload-cap-migration.md).

## IP-change resilience

Public DNS points at the VPS only. Cloudflare tunnel mode is also IP-independent. Nothing in the stack depends on the home public IP: wg1 and Tailscale both re-dial from inside after an ISP change, and a few seconds later the URL self-heals. This is why no DDNS exists anywhere in this deployment.

## Boot chain

All of it self-starts after a power cut: docker (systemd enabled) starts the four Immich containers; `wg-quick@wg1` enabled on the VM, systemd unit `cloudflared.service` enabled as fallback; nginx + certbot + wg1 on the VPS are already running. Only Tailscale needs periodic manual auth per machine policy.