# Installation

Real values from the production deployment; placeholders in `<angle brackets>` for anything secret or site-specific.

## 1. VM disk: repurpose a retired VM's logical volume

The 600G LV belonged to a retired VM (`sl-srv-dc1`, confusingly named `Plex`). After a read-only loop-device mount confirmed nothing valuable inside:

```bash
# On the KVM host — read-only peek first
LOOPDEV=$(sudo losetup -f --show -P /dev/ubuntu-vg/Plex)
lsblk "$LOOPDEV"
sudo mkdir -p /mnt/plex-check
sudo mount -o ro "${LOOPDEV}p2" /mnt/plex-check && ls -la /mnt/plex-check
sudo umount /mnt/plex-check && sudo losetup -d "$LOOPDEV"

# Retire the VM, keep its disk
sudo virsh dumpxml sl-srv-dc1 > ~/sl-srv-dc1-xml-backup.xml
sudo virsh undefine sl-srv-dc1 --nvram
sudo lvrename /dev/ubuntu-vg/Plex /dev/ubuntu-vg/immich-disk
```

**Gotchas:**
- `lvchange -an` on the LV will fail "in use" if a loop device still holds it — release with `sudo losetup -d` (check `losetup -a | grep -v snap`). Leaving the LV *active* is actually correct: virt-install needs it.
- Don't delete the VM's LV by mistake: the host root lives in the same volume group — confirm the LV name with `lvs` before any destructive command.

## 2. Create the VM

```bash
sudo virt-install --name sl-srv-immich --memory 16384 --vcpus 8 \
  --disk path=/dev/ubuntu-vg/immich-disk,bus=virtio,format=raw \
  --cdrom=/var/lib/libvirt/images/ubuntu-24.04.2-live-server-amd64.iso \
  --os-variant ubuntu24.04 --network bridge=bridge1 --graphics spice
```

Headless host: virt-viewer won't launch without a DISPLAY — install `cockpit-machines` on the host and use the browser console at `:9090` instead. Ubuntu installer: DHCP, **Use entire disk**, install OpenSSH. Then resize the LVM root inside the guest from the 100G default to the full ~598G (Storage screen → edit ubuntu-lv → max) **before** confirming — or do it later with `lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv && growpart /dev/vda 3 && resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv`.

After first boot, eject the ISO on the host (`sudo virsh change-media sl-srv-immich <cdrom-target> --eject`), install the guest agent in-guest so the host can see its filesystems (`virsh domfsinfo`), and pin the IP via DHCP reservation.

## 3. Immich stack

```bash
sudo apt update && sudo apt install -y docker.io docker-compose-v2 nano
sudo usermod -aG docker <user>    # re-login for the group to apply

mkdir -p ~/immich && cd ~/immich
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
nano .env   # UPLOAD_LOCATION=/srv/immich, DB_PASSWORD=<random alnum>, TZ, IMMICH_VERSION=v3

sudo mkdir -p /srv/immich && sudo chown <user> /srv/immich
cd ~/immich && sudo docker compose up -d
```

Register the first account at `http://<vm-ip>:2283` — first signup becomes admin. Create separate accounts per person (photos of two people in one account loses per-person face models and share granularity).

## 4. WireGuard tunnel (wg1) — dedicated, isolated

Generate keypairs **on the VPS**, build the VPS config from [`configs/wireguard-vps-wg1.conf.example`](../configs/wireguard-vps-wg1.conf.example), then push the VM-side installer from [`configs/wg-vm-setup.sh.example`](../configs/wg-vm-setup.sh.example) and run it on the VM:

```bash
sudo bash ~/wg-vm-setup.sh    # installs wireguard-tools, writes wg1.conf, enables unit,
                              # adds IMMICH_TRUSTED_PROXIES to .env, recreates containers
```

Verify from the VPS:

```bash
ping -c3 10.66.71.2
curl -I http://10.66.71.2:2283        # expect 200
```

The script aborts if a wg1.conf already exists, and never touches wg0.

## 5. nginx on the VPS + certbot

Use [`configs/nginx-photos-vhost.conf.example`](../configs/nginx-photos-vhost.conf.example). Order matters:

1. vhost on **port 80 only** first, `nginx -t && systemctl reload nginx`
2. **Then** flip DNS: `photos.<domain>` → A record `<VPS_IP>`, DNS-only. Keep the old record renamed (e.g. `photos-cf`) as rollback.
3. `apt install certbot python3-certbot-nginx && certbot --nginx -d photos.<domain>`

If Let's Encrypt gets a 403 on the challenge: another vhost on the box is catching `/.well-known/acme-challenge/` — add the explicit ACME location block from the example config. See [troubleshooting](troubleshooting.md).

## 6. Verify end to end

```bash
curl -I https://photos.<domain>       # 200, Immich's x-powered-by: Express
curl -sI http://<VPS_IP> -H "Host: photos.<domain>"   # 200 before certs, 301 after
```

Then the acceptance test that matters: upload a **>100MB video from a phone on cellular**. Watch it land with `wg show wg1` transfer counters on either end.

## 7. Mobile app

One URL for everything: `https://photos.<domain>`. Background backup enabled in-app. Traveling users need no VPN and no per-network config.