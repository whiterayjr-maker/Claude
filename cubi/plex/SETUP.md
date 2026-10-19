# Plex on the MSI Cubi 5 — Setup Guide (DRAFT)

**Status:** draft, not deployed. Nothing here has been run on any server yet.

## What and why

- **What:** Plex Media Server moves from the Synology DS923+ to the Cubi 5 (10th-gen Intel), running as a Docker container next to Project N.O.M.A.D.
- **Why:** the DS923+ CPU (AMD Ryzen R1600) has no graphics chip, so it can't hardware-transcode. The Cubi's Intel graphics (Quick Sync) can, and they handle H.264 and HEVC (including 10-bit and 4K).
- **Media stays on the Synology.** The Cubi reads it over the network, read-only.
- **Requires Plex Pass** for hardware transcoding.

## Placeholders to fill in (from the Cubi check output)

| Placeholder | Where | How to find it |
|---|---|---|
| `PUID` / `PGID` | docker-compose.yml | run `id` on the Cubi |
| `<CUBI_USER>` | docker-compose.yml | your login name on the Cubi |
| `TZ` | docker-compose.yml | confirm `America/Chicago` is right |
| Port 32400 free | — | `docker ps` output must not show 32400 in use by N.O.M.A.D. |
| `/dev/dri` exists | — | `ls -l /dev/dri` should list `card0`/`card1` and `renderD128` |
| `<PLEX_CLAIM_TOKEN>` | docker-compose.yml | https://plex.tv/claim, at deploy time only (expires in 4 min) |

## Step 1 — Synology: read-only account for Plex (recommended)

Plex only needs to read media. Use an account that can't modify or delete anything instead of `RayAdmin`.

1. DSM → Control Panel → User & Group → Create → name it e.g. `plex-ro`.
2. Permissions: `video` share = **Read only**. No access to anything else.
3. Store the password in Vaultwarden.

## Step 2 — Cubi: mount the Synology video share

This also fixes, from day one, two open items from the OptiPlex notes: the plaintext password in `/etc/fstab`, and containers starting before the share is mounted.

```bash
sudo apt install cifs-utils
sudo nano /etc/samba/nas-creds
```
Put these two lines in the file (password from Vaultwarden), save with `Ctrl+O`, exit with `Ctrl+X`:
```
username=plex-ro
password=YOUR_PASSWORD_HERE
```
Lock it down so only root can read it:
```bash
sudo chmod 600 /etc/samba/nas-creds
sudo mkdir -p /mnt/media
sudo nano /etc/fstab
```
Add this line at the bottom (one line):
```
//10.55.2.65/video/Ray /mnt/media cifs credentials=/etc/samba/nas-creds,vers=3.0,uid=1000,gid=1000,ro,_netdev,nofail,x-systemd.automount,x-systemd.mount-timeout=30 0 0
```
- `ro` — read-only at the mount level too.
- `x-systemd.automount` — mounts the share the moment something touches `/mnt/media`, so Plex doesn't start against an empty folder after a reboot.
- `nofail` — the Cubi still boots normally if the Synology is off.

Apply and verify:
```bash
sudo systemctl daemon-reload
sudo mount -a
ls /mnt/media
```
You should see your movie folders plus `TV-Series`, `Music`, `downloads`.

## Step 3 — Cubi: create Plex's folders

```bash
mkdir -p ~/docker/plex/config ~/docker/plex/transcode
```

## Step 4 — Deploy in Portainer

1. Get a claim token at https://plex.tv/claim (expires in 4 minutes).
2. Portainer → Stacks → Add stack → name `plex` → paste `docker-compose.yml` with placeholders filled → Deploy.
3. After the first successful start, the claim token is no longer needed — remove it from the stack and redeploy so it isn't left sitting there.

## Step 5 — First-time Plex setup

Open `http://<cubi-ip>:32400/web` and sign in.

**Libraries:**
| Library | Folder |
|---|---|
| Movies | `/media` |
| TV Shows | `/media/TV-Series` |
| Music | `/media/Music` |

**Watch out:** movies live in the root of the share, next to `TV-Series`, `Music` and `downloads`. Without a fix, the Movies library will also scan those folders. Check whether the Synology already has a `.plexignore` file in `/volume1/video/Ray`; if not, create one there (via DSM File Station, since the Cubi mount is read-only) containing:
```
TV-Series/*
Music/*
downloads/*
```

**Hardware transcoding:** Settings → Transcoder:
- [ ] Use hardware acceleration when available
- [ ] Use hardware-accelerated video encoding
- [ ] Transcoder temporary directory: `/transcode`

## Step 6 — Test

1. Play a movie, and in the player pick a lower quality (e.g. 720p) to force a transcode.
2. Settings → Dashboard: the stream should show **(hw)** next to the video. If it doesn't, the graphics device isn't reaching the container — check `/dev/dri` in the placeholders table.

## Step 7 — Cut over (only after testing passes)

- [ ] Radarr and Sonarr → Settings → Connect → Plex: change the host from the Synology to the Cubi's IP.
- [ ] Stop Plex on the Synology (DSM → Package Center → Plex → Stop). Keep it installed until the Cubi has run cleanly for a week.
- [ ] Firewalla: if you want Plex away from home, keep the Cubi **out** of the NordVPN group.
- [ ] Add Plex to Homepage and Uptime Kuma.
- [ ] Update the vault: Project Notes, Handoff notes, today's daily note.

## Open questions

- Watch history: start fresh (Plex account sync carries some watch state), or migrate the Synology Plex database? Decide before Step 7.
- Backups: the Cubi isn't covered by the OptiPlex's nightly backup script. `~/docker/plex/config` needs its own backup plan.
