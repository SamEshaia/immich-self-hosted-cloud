# Security Notes

## What's exposed, and what isn't

| Asset | Exposed? | Path |
|---|---|---|
| Immich web/app API | Yes | VPS 443 → nginx → wg1 → VM:2283 |
| Immich on LAN | LAN only | `http://192.168.83.50:2283` |
| Immich admin surface | Tailnet + LAN only | Tailscale 100.64.7.115 |
| Home router | **No open ports** | all tunnels dial outward |

The home WAN IP is never in DNS for this service. Scanners hit the VPS and see nginx. This matches the preexisting pattern: the game-server tunnel uses the same VPS-as-public-face design.

## Accepted exposure

- **No DDoS shield.** Leaving Cloudflare's proxy removed CF's edge mitigation on this hostname — a targeted flood can take the VPS down. Acceptable for a photo service; rollback makes restoring CF-fronting trivial.
- **Public account creation on Immich.** Out of the box, anyone reaching the URL can register an account. Mitigations: disable public registration, or front it with an auth provider. Planned: OAuth via Authentik/Pocket ID (also protects share links); Immich's native passkey support is an interim option.

## Hardening done

- Key-only SSH everywhere (root or sladmin); passwords never travel
- WireGuard: private keys 600-perms, `AllowedIPs` pinned to exact tunnel addresses (`/32` on the VPS peer), keepalive so only the *inside* host dials — the VPS never needs to initiate into the home network, and no WAN listener exists at home
- `IMMICH_TRUSTED_PROXIES=10.66.71.1` — Immich trusts exactly the tunnel peer, so client IPs and rate limits are real, not spoofed
- Secrets hygiene in this repo: every committed config is an `.example` with placeholders; `.gitignore` blocks `*.env`, `*.key`, `*.pem`, real `configs/*.conf` — mirrors the pattern from the dayz repo

## Hardening TODO (next session material)

1. **fail2ban on the VPS** for nginx + sshd — auth-log bans for the new public face. Highest-value item.
2. **OAuth in Immich** (Authentik or Pocket ID container, next to or off the same tunnel) — kills shared-password problems and gives share links an identity provider.
3. UFW on the VPS default-deny, explicit allows for 80/443/51820/51821 + established (currently iptables ACCEPT-all policy — the WireGuard ports are the only listeners, but defense in depth is free).
4. Per-machine SSH key rotation policy (currently one long-lived keypair per box).
5. Backup restore **rehearsal** (not just backups) — see maintenance.md.

## Secrets inventory (where the real values live)

| Secret | Lives at | Never in git |
|---|---|---|
| DB password | Immich VM `~/immich/.env` | x |
| wg1 private keys | VPS `/etc/wireguard/wg1.conf`, VM `/etc/wireguard/wg1.conf` | x |
| cloudflared tunnel creds | VM `/etc/cloudflared/<uuid>.json` | x |
| LE private key | VPS `/etc/letsencrypt/live/.../privkey.pem` | x |
| Tailscale identity | tailnet management | x |