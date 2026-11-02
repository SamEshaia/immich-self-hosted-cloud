# Immich Self-Hosted Cloud

Self-hosted Immich photo management running as a KVM virtual machine, fronted by nginx on a VPS over a dedicated WireGuard tunnel — built to escape Cloudflare's 100MB request-body cap while keeping the home IP out of DNS, with the original Cloudflare tunnel kept as a rollback path.

## What This Is

This repo documents a real, production Immich deployment: an Ubuntu VM under KVM/libvirt running the official Docker Compose stack, published at a single HTTPS URL for LAN and traveling users. It covers the parts most guides skip — why Cloudflare proxying caps photo/video uploads at 100MB, moving the public endpoint to an nginx VPS without opening home ports, and running a second WireGuard tunnel alongside an existing one without touching it.

All IP, WireGuard, and MAC addresses in this repository are placeholders. Substitute your own values; the topology and subnet structure are what matter.

## Architecture

```
            https://photos.<domain>   (public DNS → VPS, no CDN proxy)
                          │
            ┌─────────────▼──────────────────┐
            │  VPS (<vps_public_ip>)         │
            │  nginx + certbot (LE renewal)  │
            │  :51821/udp for wg1            │
            └─────────────┬──────────────────┘
                          │ wg1 — dedicated WireGuard tunnel,
                          │ isolated from the existing wg0
            ┌─────────────▼──────────────────┐
            │  KVM host (<kvm_host_name>)    │
            │  └─ VM: <vm_name>              │
            │     Ubuntu 24.04, 8 vCPU / 16G │
            │     600G disk (RAID6 LVM)      │
            │     Docker Compose:            │
            │       immich-server :2283      │
            │       immich-postgres          │
            │       immich-machine-learning  │
            │       immich-redis (valkey)    │
            │     uploads land in /srv/immich│
            │     Tailscale (admin access)   │
            └────────────────────────────────┘

  Rollback path (kept warm, not in production DNS):
  photos-cf.<domain> → Cloudflare edge → cloudflared → :2283
```

## Key Features

- **No upload cap** — nginx streams assets through (`client_max_body_size 50000M`, buffering off); 380MB videos land clean
- **Home IP never in DNS** — every tunnel dials outward, zero port-forwards
- **Dedicated tunnel (wg1)** — a second WireGuard interface just for this service; can't affect the existing game-server tunnel
- **Official Immich compose stack** — v3 line with the pinned Postgres vector extensions
- **`IMMICH_TRUSTED_PROXIES` set** — real client IPs and rate limiting behind the proxy
- **Cloudflare tunnel kept as rollback** — old path stays live on a renamed DNS record
- **Let's Encrypt automation** — certbot nginx plugin with renewal timer

## Documentation

- [Architecture](docs/architecture.md) — network path, addressing, ports, what's exposed
- [Installation](docs/installation.md) — VM disk reuse, VM creation, compose stack, tunnel, nginx, certbot
- [Upload-Cap Migration](docs/upload-cap-migration.md) — the 100MB Cloudflare problem and the fix
- [Maintenance](docs/maintenance.md) — Immich upgrade discipline, backups, cert renewal, health checks
- [Security Notes](docs/security.md) — what's exposed, hardening state, secrets locations
- [Troubleshooting](docs/troubleshooting.md) — real failures from this deployment and the fixes

## Config Examples

All config files in `configs/` are sanitized templates with placeholder values. Replace `<placeholder>` text before use. `configs/docker-compose.yml` is the official unmodified Immich compose file for the v3 line.

## Hardware Used

- **KVM host:** x86 server, ~160GB RAM, RAID6 LVM volume group backing VM disks
- **VPS:** RackNerd cloud VPS, 1 vCPU, nginx proxy only — uploads stream, never stored
- **VM:** 8 vCPU / 16GB RAM guest, Ubuntu Server 24.04 LTS, 600G virtio disk
- **Network:** pfSense router with VLAN-segmented server subnet, Tailscale for admin, WireGuard for the service path

## Requirements

- Ubuntu Server 24.04 LTS for the VM
- KVM/libvirt host with VG headroom (or any Docker-capable Linux for Immich itself)
- A VPS reachable on 80/443 with a free UDP port for WireGuard
- A domain — Cloudflare is used as DNS only, no proxying on the production path
- Clients: any browser; official Immich mobile apps for background backup

## License

MIT — see [LICENSE](LICENSE). This is documentation and configuration, not Immich software itself.