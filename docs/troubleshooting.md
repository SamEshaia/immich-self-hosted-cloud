# Troubleshooting

Real failures from this deployment, with fixes. The interesting ones are transport-level, not Immich-level — that's where this stack differs from a stock install.

## Uploads fail through the Cloudflare tunnel: two distinct causes

Sorted by symptom, because they look identical in the app ("Error"):

**1. VPN'd clients stall on QUIC tunnels.** iOS errors `request timed out` (`-1001`) and `network connection was lost` (`-1005`) against `https://photos.<domain>/api/assets`, *only* when a phone VPN is enabled; even tiny heartbeat requests time out mid-sync bursts that succeeded moments before. Cause: double tunneling (VPN's tunnel + CF's QUIC-in-UDP) starves streaming uploads. Fixes, in order:
   - immediate: disconnect the phone VPN
   - durable: `protocol: http2` at the top of `/etc/cloudflared/config.yml`, `sudo systemctl restart cloudflared`; confirm in `journalctl -u cloudflared` that connections register `protocol=http2` and the pre-check banner says "will use 'http2' as primary"
   - the http2 setting rides along in the kept-warm fallback config forever (see `configs/cloudflared-config.yml.example`)

**2. Large files only, any network, nothing in origin logs: the 100MB edge cap.** CF rejects >100MB request bodies at the edge — Immich sends one asset per request, so big videos never reach your server and never log. See [upload-cap-migration](upload-cap-migration.md). The tell that distinguishes this from #1: small files upload fine at the same moment, and the origin's access log has no trace of the failed asset.

## DNS flip done but caps still apply: record still proxied

After migrating to the nginx VPS, uploads still threw "Error" and nginx logs showed **client IPs as Cloudflare edges** (`104.22.x.x`) instead of real client IPs. Cause: the new `A photos → <VPS_IP>` record was left orange-clouded, so traffic still traversed CF's edge (and its cap). Fix: set Proxy status to **DNS only**. Diagnose with `dig +short photos.<domain>` — a CF edge IP (104.2x) instead of the VPS IP means the record is proxying.

## Let's Encrypt 403 during issuance

`certbot -d photos.<domain>` failed the HTTP-01 challenge with a 403 — but a manual curl of `/.well-known/acme-challenge/` worked. Cause: on a box with pre-existing vhosts (here: a catch-all `maplink` vhost), *another* server block was catching the challenge path first. Fix: add the explicit ACME block to the target vhost **before** running certbot:

```nginx
location ^~ /.well-known/acme-challenge/ { root /var/www/html; }
```

## virt-install hangs "Installation may be in progress" with no console

Headless host: `virt-install` with `--graphics spice` can't open a viewer when `DISPLAY` is unset — it then just *watches* the install forever. Ctrl+C is safe (kills only the client; the VM keeps running). Console via browser: install `cockpit-machines` on the host, Virtual Machines → Console. Cockpit won't show VMs at all until that plugin is installed.

## `lvchange -an` fails "Logical volume in use"

When preparing the LV for a new VM. Cause: a forgotten loop device (from a losetup disk peek) still holds the LV. `sudo losetup -a | grep -v snap` finds it; `sudo losetup -d /dev/loop<N>` releases it. Note: it's actually fine to leave the LV active — virt-install needs it active.

## `virsh domfsinfo` errors "QEMU guest agent is not connected"

The guest simply doesn't have `qemu-guest-agent` installed. Install it in-guest (`sudo apt install -y qemu-guest-agent && sudo systemctl enable --now qemu-guest-agent`) — then the host can see guest filesystems. Do this on *every* VM during initial setup.

## Docker compose: "permission denied … docker.sock"

New user got `usermod -aG docker <user>` but didn't re-login — group memberships apply at session start. Either re-login, or run with `sudo docker compose` (this deployment used sudo throughout; consistent and harmless).

## Face/ML jobs fail with `ML request failed EPIPE` during a big upload burst

The ML container's model was still loading when the burst hit. Immich auto-retries these jobs — verify the model is loaded, then let it drain. No action needed unless jobs stay stuck: Admin → Jobs → Smart Search → All.

## App shows "Error loading image" thumbnails

`Failed to decode image` PlatformExceptions in the mobile app log, share links failing to render previews. Usually the thumbnail job lagging a big import burst, or the app's viewer cache — reloading the library view fixes it. Server-side check that the previews exist:

```bash
# on the VM
sudo find /srv/immich/library -type f | wc -l
sudo find /srv/immich/thumbs -name 'preview.jpeg' | wc -l
# if previews ≪ originals: re-run Administration → Jobs → Thumbnail generation → All
```

## Admin login fails but there's no outage

Server log shows the 401: it's a password mistype, not a lockout (multiple devices logged in simultaneously is normal — sessions are per-device tokens). Desktop session = free password manager of the correct credential. Lost for real: `sudo docker exec -it immich_server immich-admin reset-admin-password` resets the password only — photos, albums, and existing sessions survive.

## Staged-then-missing originals scare

Files visible in `/srv/immich/library` but "empty" when checked elsewhere: Immich stages into `/data/upload` inside the container volume, then moves into `library/`. Check both. Also mind shell truncation: piping a `find` through `head` hides deeper directories — count with `wc -l`, not eyeballed listings.