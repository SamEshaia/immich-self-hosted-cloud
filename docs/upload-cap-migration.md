# Upload-Cap Migration: Cloudflare Tunnel → nginx VPS over WireGuard

## The problem: a hard 100MB cap, not a bug

Immich's mobile app uploads each asset as **one HTTP request**. Cloudflare's Free/Pro plans hard-cap any proxied request body at 100MB, enforced at the edge — a 200MB home video gets rejected as 413 before it ever reaches the origin server, producing a bare "Error" in the app with nothing in any server log. Config on the origin cannot fix it (Business plan or bust), and family members using phones won't split videos.

This deployment had photos and videos fail exactly this way: large originals (196MB–381MB) uploaded by a traveling user to the public URL, rejected at the edge every time. Small files sailed through, which made the failures look random.

## Requirements for the fix

- **One URL for everyone** — a nontechnical family member uses it while traveling, with no VPN and no per-network configuration
- **Home IP stays invisible** — same guarantee as the CF tunnel
- **Zero open ports on the home router**
- **No touching the game-server path** — the existing home↔VPS tunnel carries live game traffic; NAT/MSS edits on that host were ruled out

## The design

Move the public endpoint to nginx on the VPS; reach home over a **dedicated** WireGuard tunnel (wg1) that shares nothing with the existing tunnel (wg0). Full reasoning in [architecture](architecture.md#why-a-dedicated-second-tunnel-wg1-instead-of-reusing-wg0). Per Immich's official reverse-proxy spec, three settings do the heavy lifting:

```nginx
client_max_body_size 50000M;     # no meaningful cap
proxy_request_buffering off;     # stream: don't buffer big uploads to VPS disk
client_body_buffer_size 1024k;
```

The VPS never stores uploads — it's a streaming proxy over a ~20ms tunnel. 6G of VPS disk is plenty.

## Why not fix the tunnel instead?

Two separate failure modes were found and fixed along the way — worth knowing which is which:

| Symptom | Cause | Fix |
|---|---|---|
| Every upload dies, VPN'd clients only, iOS `-1001`/`-1005`, small requests time out too | QUIC (UDP) inside a client VPN: double encapsulation + MTU interplay | `protocol: http2` in cloudflared config (kept forever in the fallback) |
| Large videos fail only, all networks, nothing in origin logs | CF's 100MB edge cap | This migration — nginx with a raised cap |

## Migration sequence (zero-downtime, reversible)

1. Build `wg1` VPS↔VM, verify `curl http://10.66.71.2:2283` inside the tunnel
2. Add the nginx vhost on port 80 (with the ACME block — see [troubleshooting](troubleshooting.md)), reload
3. **One dashboard step:** rename the CF-proxied CNAME `photos` → `photos-cf` (kept proxied = instant rollback record), add `A photos → <vps_public_ip>` **DNS-only (grey cloud)**
4. `certbot --nginx -d photos.<domain>` after the DNS flip (LE validates against the new record)
5. Verify, then run the >100MB cellular test

**Gotcha:** after step 3, uploads were *still* capping — the new A record was **left orange-clouded**. A proxied A record re-enters Cloudflare's edge and the cap applies all over again. The tell was in the nginx access logs: client IPs were CF edge addresses (`104.22.x.x`) instead of real client IPs. Fix: set the record to **DNS only**. After that, all uploads landed, including 381MB videos from two accounts.

## Rollback drill

The CF tunnel still runs (`cloudflared.service`, enabled, `protocol: http2`) and its DNS record (`photos-cf`) stays live. Rollback is renaming the two DNS records back and waiting for TTL — under five minutes, no installs. This is why nothing was torn down on the old path.

## What changed in app behavior

Nothing. Same URL, same tokens, no re-login. Latency shifts from the nearest CF edge POP to one fixed VPS; uploads don't care. The cap is gone.

## Trade-offs

- **CF's DDoS shield is lost on this hostname.** A targeted flood can take down the VPS; CF would have absorbed it. Accepted: the threat model is port scanners and bots, not sustained attacks.
- **ISP bandwidth cost at home:** WAN-in traffic arrives via VPS → tunnel. Trivial for photo volumes; noted for completeness. (A pfSense DNS host-override → LAN IP for home clients is the optional escape hatch, without changing the URL.)
- **VPS gains the public attack surface.** See [security](security.md) for hardening state.