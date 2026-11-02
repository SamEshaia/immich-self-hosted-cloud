# Immich Self-Hosted Cloud

Self-hosted Immich photo management running as a KVM virtual machine, fronted by an nginx reverse proxy on a VPS over a dedicated WireGuard tunnel — built specifically to escape Cloudflare's 100MB request-body upload cap while keeping the home IP invisible, with the original Cloudflare tunnel kept as a one-click DNS-only fallback.

> **Note on addresses:** all IP, WireGuard, and MAC addresses in this repository are fictional placeholders (RFC 5737 / RFC 6598 style). They preserve the subnet structure but are not the real deployment's values; real ones live only on the actual infrastructure.

## What This Is

This repo documents a real, production Immich deployment: an Ubuntu VM under KVM/libvirt running the official Docker Compose stack (server, PostgreSQL with vector extensions, machine learning, Valkey), published at a single HTTPS URL for both LAN and traveling users. The public face is a cheap VPS running nginx; uploads stream through it with **no body-size cap** and hop home over a dedicated WireGuard tunnel that touches nothing else on the home network. It covers the part every guide skips: why Cloudflare tunnels cap large photo/video uploads, and how to migrate off that path without losing the zero-open-ports guarantee.

## Architecture

```
              https://photos.salmonlabs.org
                          │  DNS A record → VPS (grey cloud, no proxy)
                          ▼
        ┌───────────────────────────────────────────────┐
        │  RackNerd VPS (public face)                   │
        │  nginx + Let's Encrypt (certbot auto-renew)   │
        │  client_max_body_size 50000M (uploads stream) │
        │  ← this box absorbs all scanner/abuse noise   │
        └───────────────────────────────────────────────┘
                          │
                          │  wg1: dedicated WireGuard tunnel
                          │  10.66.71.1/24 ↔ 10.66.71.2/24, MTU 1420
                          │  (zero changes to the existing
                          │   dayz↔VPS wg0 / game traffic)
                          ▼
        ┌───────────────────────────────────────────────┐
        │  Immich VM (KVM guest on RAID6 LVM host)      │
        │  Ubuntu 24.04 · 8 vCPU / 16GB · 600G disk     │
        │  ┌─────────────────────────────────────────┐  │
        │  │ Docker Compose (Immich official)        │  │
        │  │ immich-server :2283                     │  │
        │  │ immich-postgres (vectorchord/pgvectors) │  │
        │  │ immich-machine-learning                 │  │
        │  │ immich-redis (valkey:9)                 │  │
        │  └─────────────────────────────────────────┘  │
        │  Uploads land in /srv/immich on the VM disk   │
        │  Tailscale (100.x) for admin access           │
        └───────────────────────────────────────────────┘

  Fallback path (kept warm, not in production DNS):
  photos-cf.<domain> → Cloudflare edge → cloudflared tunnel → :2283
  (100MB request cap applies on this path — rollback only)
```

## Key Features

- **No upload cap** — nginx streams assets straight through (`client_max_body_size 50000M`, `proxy_request_buffering off`); >380MB videos verified landing cleanly
- **Home IP never public** — DNS points at the VPS; the path home is an outbound-dial WireGuard tunnel (no port-forwards, no DDNS)
- **Dedicated tunnel, isolated by design** — a second WireGuard interface (wg1) just for this service, so the existing VPS↔home game-server tunnel (wg0) can never be affected
- **Official Immich Compose stack** — v3 release line with the pinned Postgres vector extensions that Immich requires
- **`IMMICH_TRUSTED_PROXIES` configured** — Immich trusts the VPS as the proxy so rate limiting and client IPs are real
- **Cloudflare tunnel kept as instant rollback** — the old path stays live on a renamed DNS record; migration back is two DNS edits
- **Let's Encrypt automation** — certbot nginx plugin with renewal timer
- **Two-account household** — separate user accounts, share links for non-tech users; one URL for everything, no VPN or per-network config for family members

## Documentation

- [Architecture](docs/architecture.md) — full topology, addressing table, ports, and why a dedicated tunnel
- [Installation](docs/installation.md) — VM creation, Docker Compose stack, WireGuard tunnel, nginx, certbot
- [Upload-Cap Migration](docs/upload-cap-migration.md) — the 100MB Cloudflare problem and the VPS migration that fixed it
- [Maintenance](docs/maintenance.md) — Immich upgrade discipline, backups, cert renewal, tunnel health
- [Security Notes](docs/security.md) — what's exposed, hardening state, secrets hygiene
- [Troubleshooting](docs/troubleshooting.md) — real failures hit during deployment and how they were fixed

## Config Examples

All config files in `configs/` are sanitized templates with placeholder values — real WireGuard private keys, database passwords, and domain-specific values are replaced. Replace `<placeholder>` text before use. `configs/docker-compose.yml` is the official unmodified Immich compose file for the v3 line.

## Hardware Used

- **KVM host:** desktop-class x86 server (~160GB RAM), RAID6 LVM volume group backing VM disks; the Immich VM inherited the 600G logical volume of a retired VM (repurposed, not new storage purchased)
- **VPS:** RackNerd KVM VPS, 1 vCPU class, 6G free disk — proxy only, uploads stream and never rest here
- **Immich VM:** 8 vCPU / 16GB RAM guest, Ubuntu Server 24.04 LTS, 600G virtio disk on RAID6
- **Network:** pfSense router (no port-forwards), VLAN-segmented server subnet, Tailscale for admin, WireGuard for the service path

## Requirements

- Ubuntu Server 24.04 LTS for the VM
- KVM/libvirt host with VG headroom, or any Docker-capable Linux box for Immich itself
- A VPS with nginx (or the ability to install it), reachable on ports 80/443, plus a free UDP port for WireGuard
- A domain with DNS control (Cloudflare is used here, but only as a DNS registrar — no proxying on the production path)
- Clients: any browser; the official Immich mobile apps for automatic background backup

## License

MIT — see [LICENSE](LICENSE). This is documentation and configuration, not Immich software itself.