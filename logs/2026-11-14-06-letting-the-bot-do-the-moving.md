---
title: "Letting the bot do the moving"
date: 2026-11-15
slug: letting-the-bot-do-the-moving
tags: [homelab, docker, arcane, synology, rsync, nfs, mealie, romm, sonarr, radarr, lidarr, prowlarr, seerr, pocketbase, yamtrack, openmausbot, metamcp, nginx-proxy-manager, tailscale]
status: draft
---

# Letting the bot do the moving

**Log #6 · 14–15 November 2026 (Asia/Dubai)**

I already pay for Claude, ChatGPT and Gemini, so why not put them to work? MausBot runs on the mini PC, so it can create folders, write compose files and deploy stacks through Arcane. My job is copying the data and telling it what not to guess.

This log starts with the leftover laptop stacks, then the bigger move: taking almost everything off the Synology (`dna-nas`, `192.168.1.18`) and onto the GEEKOM (`192.168.1.14`).

> Throughout this log, `<tailscale-ip>` is the homelab's Tailscale address (a `100.x.y.z` IP).

---

## Part 1: The split

- Me: one copy command to the mini PC (`rsync`).
- MausBot: makes folders under `/opt/docker`, writes the compose with remapped NAS paths, deploys in Arcane, checks it's running, then (because of the automations below) adds the NPM host and the Kuma monitor.

### Scorecard, laptop first

- **Dozzle:** deployed from just "deploy Dozzle", no link or port given. Worked.
- **Audiobookshelf:** done by hand as the template (rsync config + metadata, NAS paths remapped to `/mnt/nas/...`; podcasts needed the library folder fixed and a rescan).
- **LazyLibrarian:** done by MausBot after I copied the files over. Worked.
- **Aura** (poster downloader for Plex/Jellyfin): MausBot gave me the rsync command, then deployed it with the SQLite DB and poster cache (~6.5 GB), added `aura.homelab` in NPM and an Uptime Kuma monitor (Media group). Worked.
- **Streamystats:** first rsync hit `Permission denied` on the postgres folder (Postgres keeps its data dir at `0700`, owned by its own user). MausBot diagnosed it and said to rerun with sudo; that worked, then it deployed the stack, proxy host and monitor.

### Standing automations I gave MausBot

- First test prompt: create an Arcane project called dozzle with the latest image, bound to my Tailscale IP ("don't guess, ask if you don't know"), restart unless-stopped as my default, then make an NPM proxy host for `dozzle.homelab` and tell me when it's live.
- Then: find out whether Dozzle has an MCP, tell me how to add it to MetaMCP, and once I'd added it MausBot could talk to Dozzle directly.
- Daily at 9am, chained: first an Arcane-to-NPM sync (spot new stacks and create proxy hosts with the right details). Only when that finishes, a Kuma check that adds and categorizes any services missing from Uptime Kuma.
- That's why the later moves came with NPM hosts and Kuma monitors "for free".

### Supporting work

The laptop is now an Arcane Edge environment managed from the mini PC. Direct agents are deprecated as of Arcane v2.17.3 and go away in v3; Edge connects outbound over gRPC. mTLS left off since it's LAN/Tailscale only.

The plan for what's left on the laptop: it keeps qBittorrent, slskd, soularr and the beszel-agent; hermes, old uptime-kuma and old NPM retire; meilisearch left alone for now.

---

## Part 2: Starting on the NAS

Portainer on the NAS listed everything still running there. I didn't start with Plex or Jellyfin. I started with the exited and crashing apps, and I moved them as-is rather than "improving" them on the way.

Mealie (with Postgres 17), Tautulli and watchstate were first. Their logs were clean shutdowns, not mysterious crashes. Watchstate's healthcheck had been timing out because the NAS CPU is weak, not because the app was actually broken. I also deleted navet, an old Home Assistant dashboard I didn't need any more.

### The copy pattern

The data lives on the NAS, so the copy is NAS → mini PC. First wrinkle: the mini PC had no SSH server. I installed `openssh-server`, then rsync'd the config folders over.

The second wrinkle was permissions, the same shape as Streamystats. Some data dirs are mode `0700` and owned by a container user, so a plain rsync as `danish` gets `Permission denied`. The fix that stuck:

1. On the NAS, rsync with `--chmod=u+rwX` (and usually a `chown` to `danish` so the copy can actually be read).
2. On the mini PC, restore the real owners before starting the stack — `1027:100` for the linuxserver-style apps, `999` for Postgres, and so on.
3. Give MausBot just enough filesystem access to write compose files (more on that in the next part), without recursively opening up database directories.

Stop a stack that has its own database before you copy its data. I already knew that from the laptop moves; it still applies.

---

## Part 3: RomM, and teaching MausBot to write files

RomM came off the laptop, not the NAS: MariaDB plus RomM, with the game library already on the NAS at `/mnt/nas/media-expansion/Emulation`.

This is where MausBot first hit a wall. I'd already created `/opt/docker/romm` for the copied data, and Arcane's project-create API returned HTTP 422 because the folder already existed. Worse, MausBot's Arcane connector could read the environment but couldn't write files into the stack folder, so it couldn't drop a `compose.yaml` in place either.

The fix was giving it a filesystem MCP: `@modelcontextprotocol/server-filesystem`, scoped to `/opt/docker`, added in MetaMCP, with `/opt/docker` mounted into the MetaMCP container. That almost worked, then failed with `EACCES`. MetaMCP runs as uid `1001` (the `nextjs` user), and `/opt/docker` wasn't writable by that user.

The ACL that unblocked it, on each stack folder only, not recursive into database dirs:

```bash
sudo setfacl -m u:1001:rwx /opt/docker/<stack>
```

That's now part of the prep I do after rsync and before I hand the stack to MausBot.

Once RomM was up, it treated the ES-DE folders in the library (`media`, `metadata`, `systeminfo`) as if they were games. The fix was adding them to `exclude.roms.multi_file.names` in RomM's `config.yml`.

---

## Part 4: Actual Budget, then the *arrs

Actual Budget moved next, along with `actual-http-api`. I exported the Actual workflow from the old n8n on the NAS and imported it into the new n8n on the mini PC, so the automations keep talking to the same budget.

Then the media automation stacks, in two goes:

- **servarr-support:** Bazarr, Cleanuparr and pulsarr.
- **servarr-main:** Sonarr (shows), Sonarr (anime), Radarr, Radarr-4k, Lidarr (nightly) and Prowlarr.

Configs were copied first and left alone. MausBot recreated the NAS compose with the same images, container names, ports and in-container paths, with the host side remapped: `/volume1/Media` → `/mnt/nas/media`, `/volume3/Media - Expansion` → `/mnt/nas/media-expansion`, and `/volume2/docker-ssd/servarr` → `/opt/docker/servarr-main`. Ports bind to `<tailscale-ip>`, restart is `unless-stopped`, and the timezone stays `Asia/Dubai`.

### The downloads twist

The *arrs also need to see qBittorrent's downloads, which I first assumed lived on the NAS Torrents share (`/volume3/Torrents`). Trying to add NFS permissions for the mini PC on that share, DSM refused: the share has a remote folder mounted inside it, pointing at the laptop. The files don't really live on the NAS. Clicking Cancel on that dialog was the right call; there was nothing to fix on the NAS.

The laptop's qBittorrent already maps `/home/danish/torrents` to `/downloads-laptop` inside the container, and that's the folder the *arrs expect. So I exported it over NFS directly to the mini PC.

The laptop's Ethernet address is `192.168.1.21` (Wi-Fi is a different address, and I shouldn't use that one). The export is limited to the mini PC, and it squashes writes to the same user qBittorrent uses, so the two sides don't fight over file ownership.

The first mount from the mini PC timed out. A later `mount -a` worked, and the download folders showed up under `/mnt/laptop/torrents`. Writes still failed, though. The folder is owned by user `911` (the linuxserver default, which is what qBittorrent runs as), but the export was squashing to `1000` (my laptop login). The fix was pointing `anonuid` / `anongid` at `911` and re-exporting:

```bash
# on the laptop, after editing /etc/exports
sudo exportfs -ra
```

On the mini PC the mount lives at `/mnt/laptop/torrents`, and in the *arr compose `/volume3/Torrents/Homelab-Laptop/torrents` becomes that path. The other `/downloads` mount, the media-expansion Torrents folder, still goes through `/mnt/nas/media-expansion`.

A DHCP reservation for `192.168.1.21` on the router is worth having, so the laptop keeps that Ethernet address.

Once the *arrs are on the mini PC, a few things still talk to them by the old NAS IP and need pointing at the new box: Prowlarr's app connections, Bazarr, Cleanuparr, pulsarr, Seerr, and Soularr on the laptop. The download-client settings in each *arr already point at the laptop's qBittorrent, so those stay as they are.

---

## Part 5: Seerr, PocketBase, Yamtrack, and a dashboard

**Seerr** came over next. The NAS compose used `network_mode: host`, which also made the `ports:` line a no-op. I dropped host networking and bound `5055` to `<tailscale-ip>`, same as the other stacks. I also switched its log level from debug to info; debug was only useful while I was troubleshooting. Plex and Jellyfin stay on the NAS IP in Seerr until those move.

**PocketBase** was a straight copy of `pb_data` and `pb_migrations`. It ran as `1026:100` on the NAS, with a Synology-only `group_add` so it could get through the ACLs on `/volume2/web`. That group can go; the mini PC has no Synology ACLs. Same image, same port `8090`, same healthcheck.

**Yamtrack** (plus its Redis) moved with its database left untouched. The NAS compose had `GUID: 100`, which is a typo — the variable the image actually reads is `PGID` — so it was probably ignoring the group the whole time. Secrets (the app secret and the Trakt / Steam keys) go in a `chmod 600` `.env`, not in the compose, and the same secret as on the NAS is what keeps existing logins working.

I've now got enough services that I want a single page of tiles, with a green or red dot, that opens the app when I click it. Homepage is the one I'm planning to use: YAML config, so MausBot can design the dashboard and keep it updated when I add a stack, without me touching the YAML. Homarr would be the GUI alternative. I wrote MausBot a prompt to deploy it as `home.homelab` and lay out groups (Media, *arrs, Downloads, Apps, Infra, Home) from the Arcane stacks, but I haven't confirmed that deploy in this session — so this is planned, not done.

---

## What's left

The remaining big three are still on the NAS: **Plex**, **Jellyfin** and **Home Assistant**. Everything else is basically done.

Later cleanup, not now:

- Remove the old n8n and NPM on the NAS.
- Replace the NAS Beszel with the new Beszel.
- Add an Arcane agent to the NAS, so that box shows up in the same UI.

---

## Gotchas

- `ssh host 'sudo ...'` needs `-t` for the password prompt; a silent "Connection closed" means it worked.
- Containers should reach Jellyfin and Plex by NAS IP, not localhost, for as long as those still live on the NAS.
- Stop a stack with its own database before copying its data.
- The mini PC needed `openssh-server` before rsync from the NAS would work.
- `0700` data dirs (Postgres especially) need `--chmod=u+rwX` on the copy, then the original owners put back (`1027:100`, Postgres `999`, PocketBase `1026:100`, and so on).
- Don't let Arcane "create" a project folder that already exists; you'll get HTTP 422. Drop the compose into the existing folder instead.
- MetaMCP runs as uid `1001`. `setfacl -m u:1001:rwx` on the stack folder is enough; don't recurse that ACL into database directories.
- DSM will refuse NFS permissions on a shared folder that already has a remote mount inside it. If the files actually live on the laptop, export the laptop folder directly.
- An NFS export that squashes to the wrong uid will mount fine and still fail writes. Match `anonuid` / `anongid` to the owner of the folder (here, `911` for linuxserver/qBittorrent), not to your login uid.
- Don't rerun a "tee -a /etc/fstab" command after a failed mount; the line is already there. Just retry `mount -a`.
- `network_mode: host` ignores the `ports:` line. If everything else is bound to the Tailscale IP, Seerr should be too.
- Yamtrack's `GUID` is a typo for `PGID`.
