# Frigate on the MSI Cubi 5 — Setup Guide (DRAFT)

**Status:** draft, not deployed. Nothing here has been run on any server yet.

## What and why

- **What:** Frigate is a security-camera recorder (NVR) that runs AI object detection locally, so you get "person at the front door" alerts instead of "something moved". No cloud and no subscription.
- **Why the Cubi:** its 10th-gen Intel graphics handle both jobs Frigate needs: decoding the camera video (VAAPI) and running detection (OpenVINO). On a CPU-only box, detection would keep the processor busy all the time. Plex uses the same graphics chip for transcoding, and the two can share it.
- **Recordings go to the Synology** (new `frigate` share). Frigate's small database and config stay on the Cubi's own disk; Frigate requires that.

## Before you start: cameras

Frigate needs cameras that offer an **RTSP** stream on your local network. Good choices: Reolink (PoE models), Amcrest, Dahua, Hikvision, and most ONVIF cameras.
**Won't work directly:** Ring, Nest, Arlo, Blink, and most Wyze cameras (cloud-only).

For each camera, write down: IP address, username, password, and the RTSP paths for the **main** (high-res) and **sub** (low-res) streams. Search "<brand> <model> RTSP URL" if you don't know them. Store the password in Vaultwarden.

**Firewalla tip:** put the cameras in their own Firewalla group and **block their internet access**. Frigate only needs them on the LAN, and many cheap cameras send data home to their maker.

## Placeholders to fill in

| Placeholder | Where | How to find it |
|---|---|---|
| `<CUBI_USER>` | docker-compose.yml | your login name on the Cubi (same as the Plex stack) |
| `<CAMERA_PASSWORD>` | docker-compose.yml | camera password (Vaultwarden). If cameras have different passwords, put them straight into config.yml instead |
| `<CAMERA_USER>`, `<CAMERA_IP>` | config.yml | per camera, from the camera's app or Firewalla's device list |
| Camera stream paths | config.yml | the example uses Reolink paths; change them for other brands |
| `detect` width/height | config.yml | resolution of each camera's **sub** stream |
| `shm_size` | docker-compose.yml | 256mb covers about 2–4 cameras at 720p. Raise to 512mb if you go past that |
| Port 8971, 8554, 8555 free | — | `docker ps` on the Cubi must not show them in use |

## Step 1 — Synology: share + account for recordings

1. DSM → Control Panel → Shared Folder → Create → name `frigate`, on Volume 1. Leave the recycle bin **off** (Frigate deletes old footage constantly).
2. DSM → Control Panel → User & Group → Create → `frigate-rw`. Permissions: `frigate` share = **Read/Write**, nothing else.
3. Store the password in Vaultwarden.

**How much space?** One 1080p camera recording 24/7 uses about 20–40 GB per day. The draft config records **only when there's motion**, which usually cuts that by 80–90%. To get a rough total, multiply by the number of cameras and the days kept (7 for motion, 30 for alerts). You can set a quota on the share so Frigate can never fill the NAS: DSM → Shared Folder → Edit → Advanced → quota.

## Step 2 — Cubi: mount the share

The same method as the Plex guide's Step 2, but read-**write**, with a separate credentials file:

```bash
sudo nano /etc/samba/frigate-creds
```
```
username=frigate-rw
password=YOUR_PASSWORD_HERE
```
```bash
sudo chmod 600 /etc/samba/frigate-creds
sudo mkdir -p /mnt/frigate
sudo nano /etc/fstab
```
Add this line (one line):
```
//10.55.2.65/frigate /mnt/frigate cifs credentials=/etc/samba/frigate-creds,vers=3.0,uid=1000,gid=1000,rw,_netdev,nofail,x-systemd.automount,x-systemd.mount-timeout=30 0 0
```
```bash
sudo systemctl daemon-reload
sudo mount -a
touch /mnt/frigate/test && rm /mnt/frigate/test && echo "write OK"
```

## Step 3 — Cubi: create Frigate's folders and config

```bash
mkdir -p ~/docker/frigate/config
nano ~/docker/frigate/config/config.yml
```
Paste the contents of `config.yml` from this folder, with the camera placeholders filled in. Save with `Ctrl+O`, exit with `Ctrl+X`.

## Step 4 — Deploy in Portainer

Portainer → Stacks → Add stack → name `frigate` → paste `docker-compose.yml` with placeholders filled → Deploy.

Get the admin password: Portainer → Containers → `frigate` → Logs. On first start, Frigate prints a generated `admin` password. Save it to Vaultwarden and change it after logging in.

## Step 5 — First look

Open `https://<cubi-ip>:8971`. Your browser will warn about the certificate (it's self-signed); accept it.

- [ ] Live view shows every camera.
- [ ] Settings → System metrics: the detector (`ov`) inference speed should be roughly **10–30 ms**. Much higher means it's running on the CPU, not the GPU.
- [ ] Settings → System metrics: the GPU should show some usage (decoding). If it's 0%, `/dev/dri/renderD128` isn't reaching the container (run `ls -l /dev/dri` on the Cubi).
- [ ] Walk in front of a camera. A "person" detection should appear under Review within a few seconds.
- [ ] Recordings appear in `/mnt/frigate/recordings` (check from DSM File Station).

## Step 6 — Tune (after a few days)

- **Too many false alerts?** Add a motion mask over trees and roads, or zones such as "driveway" or "porch", in the Frigate UI (Settings → Masks / Zones).
- **Want 24/7 recording?** Set `record.continuous.days` to e.g. `3` and keep an eye on NAS usage.
- **Phone notifications:** Frigate has built-in browser and phone notifications (Settings → Notifications). They need HTTPS; the 8971 port already provides it.

## Step 7 — Finish

- [ ] Add Frigate to Homepage and Uptime Kuma.
- [ ] Back up `~/docker/frigate/config` (it is not covered by the OptiPlex nightly backup script; same open item as Plex).
- [ ] Update the vault: Project Notes, Handoff notes, today's daily note.

## Open questions

- Which cameras, and how many? This decides `shm_size`, NAS space, and whether the bundled model is good enough. With 6+ cameras, consider switching to the YOLOv9 model, which is more accurate on Intel GPUs. See the Frigate docs, *Object Detectors → OpenVINO*.
- Remote viewing: same as Plex. Use a VPN (Firewalla VPN server, or Tailscale) rather than opening ports.
- After a reboot, check with `docker exec frigate ls /media/frigate` that the container sees the NAS share and not an empty folder. If it's empty, restart the stack. If that keeps happening, change the volume line to `/mnt/frigate:/media/frigate:rslave`. The same applies to Plex's `/mnt/media`.
