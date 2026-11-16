---
title: "The big ones and a VPN"
date: 2026-11-16
slug: the-big-ones-and-a-vpn
tags: [homelab, docker, arcane, jellyfin, plex, gluetun, pia, qbittorrent, slskd, soularr, lidarr, seerr, vpn, nginx-proxy-manager, synology, rsync]
status: draft
---

# The big ones and a VPN

**Log #7 · 16 November 2026 (Asia/Dubai)**

Last time I left the "big three" on the Synology: Plex, Jellyfin and Home Assistant. Today I moved the two media servers onto the GEEKOM (`homelab`, `192.168.1.14`), pointed Seerr at the new address, gave qBittorrent a proper `torrents.homelab` name, and put the laptop's download clients behind a VPN.

Home Assistant can wait until next week. Everything I moved today is working — though Plex took a second copy, and Soularr needed a few config fixes, before I could say that.

> Throughout this log, `<tailscale-ip>` is the mini PC's Tailscale address, and `<laptop-tailscale-ip>` is the laptop's (both `100.x.y.z` IPs).

---

## What's still on the NAS

By this morning, almost everything compute-related was already off `dna-nas` (`192.168.1.18`). The holdouts were Plex, Jellyfin and Home Assistant. After today, only Home Assistant is still there. Plex, Jellyfin, Seerr, qBittorrent behind the VPN, and slskd behind gluetun are all up and working.

---

## Who actually needs the internet?

Before I dropped the media servers onto the mini PC, I wanted a clear answer for what should be reachable from outside the house, and what can stay on Tailscale.

Needs a public path:

- **Jellyfin**, so people can watch without joining the tailnet.
- **Plex**, but through its own remote access on port `32400`, not through a Cloudflare Tunnel. Cloudflare's terms don't love video streaming through the tunnel.
- **Seerr**, only if people outside the tailnet need to request media. If it's just us, Tailscale is enough.
- **PocketBase**, if any of the sites I deploy (on Vercel, for example) call it. If I do expose it, the `/_/` admin UI should sit behind Cloudflare Access, not on the open internet.
- **Uptime Kuma's status page**, which is already public.

Can stay private:

- **Actual Budget** can live on Tailscale. `actual-http-api` needs no exposure at all — only n8n on the same host talks to it.
- **n8n**: only expose `/webhook/` if something outside actually needs to hit a webhook.
- Everything else is Tailscale-only: the *arrs, qBittorrent, slskd, Arcane, MetaMCP, NPM, AdGuard, Dozzle, Beszel, Homepage, Mealie, Yamtrack, RomM, Audiobookshelf, Tautulli, and Home Assistant when it moves.

---

## Moving Jellyfin

The NAS compose used `network_mode: host` and the official Jellyfin image. A few things in that compose were wrong, and this was a good time to fix them rather than copy the mistakes over.

The official image ignores `PUID` / `PGID` (the NAS compose also had `GUID`, which is a typo — same one Yamtrack had). The way to actually run it as my media user is `user: "1027:100"`. Timezone was set to `Dubai`, which isn't a valid TZ string; it needs to be `Asia/Dubai`.

Sizes on the NAS, before the copy:

- config: 19G
- cache: 73M
- logs: 28M
- jf-avatars: 3.5M

Jellyfin was already stopped, so one rsync pass to `/opt/docker` on the mini PC was enough. No separate "Jellyfin backup" step: the config folder is the backup. It holds the SQLite database, so libraries and watch history come along if I keep the in-container paths the same.

That's the important part. The libraries still see `/movies-vol1`, `/anime-vol3`, and the rest. On the host side those now map through the existing NFS mounts, `/mnt/nas/media` and `/mnt/nas/media-expansion`, instead of `/volume1/...` and `/volume3/...`.

I kept host networking. TVs on the LAN still need to discover it, and cloudflared is already aiming at `localhost:8096`. I also passed through the iGPU at `/dev/dri`, with `group_add` for the render group (gid `990`), so QSV is available in Jellyfin's Transcoding settings.

The wedding folder, which lived under `/volume1/homes/...` on the NAS, I skipped for now. That path isn't on the media NFS exports, so it needs its own mount before I can bring it across.

Jellyfin came up cleanly on the mini PC, with libraries and watch history intact.

---

## Seerr still pointed at the NAS

Seerr couldn't log in after Jellyfin moved, which was the first obvious break. `settings.json` still had Jellyfin at `192.168.1.18`. I stopped Seerr, pointed the `ip` line at `192.168.1.14` with `sed`, and started it again. After that, Seerr logged in normally.

Anything else that still has the old NAS IP hardcoded is in the same boat — I'll sweep those when I get to the leftover cleanup (Prowlarr, Bazarr, Cleanuparr, pulsarr, Seerr's Sonarr/Radarr connections, Soularr, and so on).

---

## Moving Plex

Plex is the linuxserver image, and its config on the NAS is 36G. I thought about skipping `Cache` and `Codecs` (Plex rebuilds those itself), then decided to copy everything. Safest, just slower.

A few compose changes for the new box:

- Drop `PLEX_CLAIM`. Claim tokens are only for a fresh install, and they expire within minutes anyway. The copied config already has the server signed in.
- Keep host networking, same reason as Jellyfin: LAN discovery.
- Pass through `/dev/dri` with `group_add` `990` for hardware transcoding. That needs Plex Pass.
- Stack folder is `/opt/docker/plex-stack` (compose only). The live data stays in `/opt/docker/plex`, and MausBot shouldn't touch it.
- If I use Plex remote access, the router's `32400` port forward has to move from `.18` to `.14`.

Same media mounts as Jellyfin, same skipped wedding folder. Plex did start — and then looked completely broken. That turned out to be the copy, not the compose.

---

## Plex came up empty

In the app, the whole server showed as offline. Every library had a warning icon and "Not found". That looks like a network problem, or like the media mounts didn't attach. I spent a while proving it wasn't either of those.

`/identity` on `localhost:32400` and through the Cloudflare Tunnel at `media.danishandarwa.com` both returned the same `machineIdentifier`, with `claimed=1`. So this was the same claimed server, reachable locally and through the tunnel. Inside the container, `/movies-vol1` was there and readable. The logs were basically fine: `libusb_init` and `recv: Connection reset by peer` are harmless noise.

The tunnel on the mini PC was already pointed at the new box: `media.danishandarwa.com` → `localhost:32400`, `jellyfin.danishandarwa.com` → `localhost:8096`, plus the existing status, budget, budget-api and PocketBase routes. Plex's custom server access URL is `https://media.danishandarwa.com:443`. None of that was the bug.

The real cause was the database. `com.plexapp.plugins.library.db` on the mini PC was only about 777K, and it had been created when Plex first started. On the NAS the same file is 609M. The first rsync never fully landed, and Plex helpfully created a fresh empty library database on top of what little had arrived.

Why the copy was incomplete: I'd already run `chown -R 1027:100 /opt/docker/plex` so the container could read its data, and then the `danish` user doing the rsync receive started getting `Permission denied` — `opendir` and `mkdir` were failing. The lesson is simple: chown to the app user only after the copy is finished, or chown back to `danish` temporarily if you need to rerun rsync.

Trying to re-rsync tens of thousands of tiny metadata files crawled, then hung. Ctrl+C was ignored; I had to `pkill -9 rsync`.

The fix that actually worked was to stop treating it as a tree of tiny files. On the NAS I tar'd the plex folder into one 36G archive and rsync'd that single file. It ran at about 110MB/s and took around five minutes. The first attempt went to `/tmp` on the mini PC and died with `Disk quota exceeded` — `/tmp` is too small for a file that size — so I sent it to `/home/danish` instead. Then I removed `/opt/docker/plex`, extracted the archive into `/opt/docker`, ran `chown -R 1027:100`, and started Plex.

This time it came up with all the libraries and watch history.

---

## Will 500GB be enough?

I did wonder, with all these configs landing on the mini PC's 500GB NVMe, whether I was about to fill the disk. Mid-move, while Plex was still copying (the first time), it looked like this:

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p2  457G  122G  312G  29% /
```

The big tenants under `/opt/docker` were `servarr-main` at 23G (almost all `MediaCover` posters and fanart, which is normal and grows slowly), Jellyfin at 20G, Aura at 6.6G, OpenBot at 3.8G, Watchstate at 1.5G, and Plex still arriving. Once Plex finished — including the second, complete copy — `/opt/docker` was always going to land around 90G. Media itself stays on the NAS, so new movies don't eat this disk.

Most new containers are under 100MB. 300G free fits a lot of them. The sneaky growth is old Docker images after auto-updates, plus metadata slowly accumulating.

Useful habits:

- `docker system df` to see where Docker is actually spending space.
- `docker image prune -a` now and then for unused images.
- A Beszel disk alert at 80%, so I hear about it long before it's a problem.
- `mcpjungle` and `komodo` are leftovers from earlier experiments and can be deleted whenever I like.

---

## torrents.homelab

qBittorrent still lives on the laptop (`192.168.1.21`), but I wanted a name that matches everything else. AdGuard already rewrites `*.homelab` to the mini PC, so I added an NPM proxy host: `torrents.homelab` forwards to `192.168.1.21:9866`, with websockets on.

The first hit came back `Unauthorized`. That's qBittorrent's host header validation, not NPM being wrong. The fix is in qBit's Web UI settings: either add `torrents.homelab` to the allowed hosts, or turn the validation off.

---

## qBittorrent behind a VPN

Torrenting from a home IP in the UAE is a bad idea, so qBittorrent on the laptop now goes out through gluetun, in the same Arcane Edge stack (`danish-homelab-laptop`).

I already have a Private Internet Access subscription, and PIA supports port forwarding, which is what you want if you care about seeding. One catch: gluetun talks to PIA over OpenVPN, not WireGuard, so the credentials are a username and password in a `chmod 600` `.env` — never in the compose, and never in chat.

The shape of the stack:

- gluetun (`qmcgaw/gluetun`) gets `NET_ADMIN`, `/dev/net/tun`, `TZ=Asia/Dubai`, and `VPN_SERVICE_PROVIDER=private internet access` with `VPN_TYPE=openvpn`.
- `VPN_PORT_FORWARDING=on`, and gluetun updates qBittorrent's listening port to the forwarded one automatically.
- `FIREWALL_OUTBOUND_SUBNETS=192.168.1.0/24`, so the LAN and the mini PC can still reach the web UI.
- qBittorrent keeps its existing config, downloads path, PUID/PGID and image. Its published web port (`9866`) moves onto gluetun, and qBit itself uses `network_mode: "service:gluetun"` with `depends_on` gluetun (`condition: service_healthy`). If the VPN drops, qBit loses internet entirely, so nothing leaks out on my real IP.

### Don't pick a GCC server

MausBot's first region suggestions were nearby ones: UAE, Qatar, Saudi. Those are a bad fit. Torrenting is banned in the GCC, PIA's UAE endpoint is a virtual location anyway, and I don't want an exit IP in a country that treats this as a problem. India isn't great either — blocking and enforcement are common.

I went with Switzerland. The Netherlands would have been fine too. Both support port forwarding, both are P2P-friendly, and the extra latency doesn't matter for downloads.

### Checking it actually uses the VPN

The container name is `qBittorrent-VUE`, and it's case-sensitive — `qbittorrent` and `qbittorrent-VUE` both miss. From inside it:

```bash
docker exec qBittorrent-VUE curl -s https://ipinfo.io
```

That came back as a Zürich PIA IP (Switzerland, Datacamp). The same lookup on the laptop host still shows my home IP in Dubai (Emirates Internet). So qBit's traffic is leaving through PIA, and the laptop itself is not.

An optional extra check is the torrent address test on [ipleak.net](https://ipleak.net): activate it, add the magnet link in qBit, wait a minute, and the page should show the Zürich IP, not the Dubai one. The test torrent doesn't download anything; delete it afterwards.

---

## slskd, same idea

slskd does the actual Soulseek peer-to-peer sharing, so it exposes my IP the same way qBittorrent does. It now sits behind gluetun too. Soularr only talks to Lidarr and slskd's API on my network — it never touches peers — so it stays outside the VPN.

slskd had to move into the same stack as gluetun and qBittorrent, because `network_mode: "service:gluetun"` only works inside the same project. Its web UI (`5030`) is published on gluetun, and Soulseek's listen port (`50300`) is in `FIREWALL_VPN_INPUT_PORTS`. The forwarded port stays with qBittorrent; one forwarded port can't serve both apps. slskd still works without a forwarded port, just with fewer incoming connections.

That's in place and working. The web UI still loads on `<laptop-tailscale-ip>:5030`. Getting Soularr and Lidarr to talk to it, and to import what slskd had already downloaded, took a few more fixes.

---

## Soularr, Lidarr and a pile of albums

Once slskd was behind gluetun, Soularr started complaining. First it was a `KeyError: 'download_dir'` from an old config. After that, `Please provide an API-Key, a valid token or username/password`, because the `[Slskd]` `api_key` in `config.ini` was empty — and the `host_url` line had gotten mangled in the edit.

I already had an slskd API key in `slskd.yml`. With `SLSKD_REMOTE_CONFIGURATION=true`, that's editable from the slskd web UI under System → Options, so I just pasted it into Soularr. Soularr also has a small log-viewer web UI on port `8265`, which made the errors easier to see.

Lidarr nightly has a native slskd download client now, so I tried that too. The test came back `403 Forbidden` on `/api/v0/options`. The CIDR allowlist of `0.0.0.0/0` was fine; the problem was the key's role, which was `readwrite`. Lidarr needs `administrator`. I could also tighten the CIDR later to `192.168.1.0/24,100.64.0.0/10`, so only the LAN and the tailnet can use the API. One warning: running Soularr and Lidarr's slskd client at the same time can double-grab the same album.

Then nothing was importing. slskd puts finished files in `slskd/downloads` (incomplete downloads live there too), not in `slskd/complete`. I pointed Soularr's `download_dir` at `/downloads-laptop/slskd/downloads` in the `[Lidarr]` section and `/data/downloads` in the `[Slskd]` section.

Lidarr then warned that "download client Slskd places downloads in `/data/downloads` but this directory does not appear to exist inside the container." That's a remote path mapping: host `<laptop-tailscale-ip>`, remote `/data/downloads/`, local `/downloads-laptop/slskd/downloads/`.

There were already hundreds of old albums sitting in that folder. The way to pick them up is a Manual Import on the whole folder, or Lidarr's `DownloadedAlbumsScan` API command with the path — and if you do it from the CLI, take the API key from an environment variable, not inline. Also, the Lidarr container name is capitalised, same class of gotcha as `qBittorrent-VUE`.

---

## What's left

Home Assistant, including esphome and matter-server, waits until next week. It wants host networking and mDNS, so I'll give it a proper window rather than squeezing it in now.

Still to do, not this week:

- Repoint leftover *arr IPs in Prowlarr, Bazarr, Cleanuparr, pulsarr, Seerr's Sonarr/Radarr connections, and Soularr.
- Update the n8n Actual Budget workflow with the new URL and credentials.
- Clean up the old n8n and NPM on the NAS, replace the NAS Beszel with the new one, and add an Arcane agent to the NAS so that box shows up in the same UI.
- NFS-mount the wedding folder and add it back to Jellyfin and Plex.

---

## Gotchas

- The official Jellyfin image ignores `PUID` / `PGID`. `GUID` is also a typo. Use `user: "1027:100"` if you need that uid/gid.
- `TZ=Dubai` is not a valid timezone; it has to be `Asia/Dubai`.
- Keep the in-container media paths identical (`/movies-vol1`, `/anime-vol3`, and so on) or Jellyfin and Plex will treat the libraries as new and you'll lose watch history.
- You don't need a separate Jellyfin backup. The config folder holds the SQLite database; that's the whole state.
- Don't put `PLEX_CLAIM` in the compose when you're moving an existing server. It's only for a fresh claim, and the token expires in minutes. The copied config already has the login.
- Hardware transcoding needs `/dev/dri` and the render group (gid `990` here). Plex also needs Plex Pass for it. In Jellyfin, turn on QSV in Transcoding settings after the move.
- If you use Plex remote access, move the router's `32400` port forward from the NAS to the mini PC.
- Seerr (and anything else) may still have the old NAS IP in a settings file. Fix that while the app is stopped; a login failure is the typical symptom.
- qBittorrent returns `Unauthorized` on a pretty hostname until you allow that host in the Web UI settings, or turn off host header validation.
- gluetun speaks OpenVPN to PIA, not WireGuard. Credentials go in a `chmod 600` `.env`, not in the compose.
- Don't pick a GCC PIA region for torrents. Use a P2P-friendly, port-forwarding region (Switzerland or the Netherlands). India is also a poor choice.
- `docker exec` container names are case-sensitive. Mine are `qBittorrent-VUE` and `Lidarr`.
- One forwarded port can't be shared. Give it to qBittorrent; slskd can live without an incoming port.
- Soularr does not need the VPN. Only slskd does.
- Set `FIREWALL_OUTBOUND_SUBNETS` to your LAN, or the web UIs behind gluetun won't be reachable from the rest of the house.
- Confirm the VPN from inside the container (`docker exec ... curl https://ipinfo.io`) and compare it to the same lookup on the host. They should disagree.
- Media stays on the NAS, so the mini PC's disk fills with configs, metadata and leftover Docker images, not movies. `docker image prune -a` and a Beszel alert at 80% are the boring safeguards.
- Don't `chown` the destination to the app user before rsync finishes. The user doing the receive will get `Permission denied` mid-copy (`opendir` / `mkdir` fail), and a service that starts against a partial tree can create a fresh empty database. Plex's `com.plexapp.plugins.library.db` went from 609M on the NAS to 777K on the mini PC.
- An "offline" / "Not found" Plex library after a move can be that empty database, not a tunnel or mount problem. Compare `library.db` size with the source, and check `/identity` locally and through the tunnel (`claimed=1`, same `machineIdentifier`).
- `libusb_init` and `recv: Connection reset by peer` in Plex logs are usually harmless.
- Re-rsyncing tens of thousands of tiny metadata files can crawl and then hang; Ctrl+C may not kill it (`pkill -9 rsync`). Tar the folder into one archive and rsync that instead.
- `/tmp` may have a disk quota too small for a 36G archive. Land it in `/home` instead, then extract, `chown` to the app user, and start the container.
- Soularr `KeyError: 'download_dir'` means an old config. An empty `[Slskd]` `api_key` produces the "provide an API-Key / token / username/password" error. Watch for a mangled `host_url` when editing `config.ini`.
- slskd API keys live in `slskd.yml` and can be edited in the web UI (System → Options) when `SLSKD_REMOTE_CONFIGURATION=true`. Soularr's log viewer is on port `8265`.
- Lidarr's native slskd client needs an API key with the `administrator` role, not just `readwrite`. `403 Forbidden` on `/api/v0/options` is the symptom. Optionally tighten slskd's CIDR allowlist to `192.168.1.0/24,100.64.0.0/10`.
- Don't run Soularr and Lidarr's slskd client at the same time, or they can double-grab the same album.
- slskd finishes downloads in `slskd/downloads`, not `slskd/complete`. Point Soularr at `/downloads-laptop/slskd/downloads` (`[Lidarr]`) and `/data/downloads` (`[Slskd]`).
- Lidarr needs a remote path mapping from slskd's `/data/downloads/` to `/downloads-laptop/slskd/downloads/`, with the host set to the laptop's Tailscale IP. For a backlog, use Manual Import on the whole folder, or the `DownloadedAlbumsScan` API command with the path. If you do that from the CLI, take the API key from an environment variable, not inline.
