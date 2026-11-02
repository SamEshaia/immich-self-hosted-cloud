# Maintenance

## Immich upgrade discipline (read before every `docker compose pull`)

Immich occasionally ships **database-breaking releases**, and the Postgres vector extension images must move in lockstep with the app — a mixed pull across releases corrupts the database. The rules:

1. Never pin `IMMICH_VERSION` across a major version jump without reading the release notes for the target version first
2. The postgres container (`postgres:14-vectorchord…-pgvectors…`) must be the one referenced by the release notes for that Immich version — compose files from the same release are pulled together, never mixed across releases
3. Take a fresh database dump before pulling major upgrades (below)
4. After a major upgrade, check the Admin → Jobs screen completes; smart-search/face jobs re-run is normal

## Backups (what and how)

| What | Where | Method |
|---|---|---|
| Originals + thumbs | `/srv/immich` on the VM disk | rsync/restic to the backup target nightly |
| Database | container `immich_postgres` | `docker exec immich_postgres pg_dump -U <db_user> <db_name> \| gzip > <target>/immich-pg-$(date +%F).sql.gz` before major upgrades and weekly |
| Configs | `~/immich/docker-compose.yml`, `.env`, `/etc/cloudflared/config.yml`, `/etc/wireguard/wg1.conf`, nginx vhost | in a secrets manager or backup target; `.env` and wg keys are secrets — handle accordingly |

Test restores: a photo library backup is only real after a restore rehearsal to an empty directory.

## Cert renewal

certbot's nginx plugin installs a renewal timer (verify with `systemctl list-timers | grep certbot` on the VPS). Dry-run quarterly: `certbot renew --dry-run`.

## Health checks (quick set)

```bash
# VM
sudo docker ps --format '{{.Names}}\t{{.Status}}'    # all four: healthy, no restarts
wg show wg1 | grep -A2 peer                           # recent handshake + counters moving
df -h / | tail -1                                     # uploads live here; alert before 90%

# VPS
wg show wg1                                           # handshake from VM side (keepalive)
journalctl -u cloudflared -n 20                       # fallback path alive
curl -sI https://photos.<domain> | head -3            # 200 + server: nginx
```

## Known-cosmetic noise (don't chase)

- `systemd-networkd-wait-online` failed on hosts with unplugged NICs (eno3/eno4) — unused ports, not a fault
- cloudflared `ping_group_range` warnings — ICMP proxy feature only; HTTP serving unaffected
- iOS "Failed to decode image" thumbnails in the app log — viewer decode glitch; fix by reloading the library view
- One-off `ML request failed EPIPE` during a huge burst — model was mid-load; jobs auto-retry

## Storage outlook

The VM disk is a 600G LV carved from a RAID6 pool with ~933G free at deployment. Expansion is `lvextend` on the host + `growpart`/`resize2fs` in-guest. Watch `/srv/immich` growth monthly; the first 1,000 assets ran ~1.9G.