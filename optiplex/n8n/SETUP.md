# n8n on the OptiPlex — Setup Guide (DRAFT)

**Status:** draft, not deployed. Nothing here has been run on any server yet.

## What and why

- **What:** n8n is a visual automation builder: "when X happens, do Y". It's like Zapier or IFTTT but self-hosted, with no per-task fees. Examples for this setup:
  - Uptime Kuma or Frigate alert → push notification or email.
  - Every Sunday: check Synology disk space → send a summary.
  - New Plex/Radarr download → message.
  - Gmail with a receipt attached → save the PDF to a NAS folder (and later into Homebox).
- **Why the OptiPlex:** light (~200–300 MB RAM when idle), and it sits next to the apps it will talk to.
- **Database:** the built-in SQLite. Plenty for one household; Postgres is only worth it at heavy business volume.

## Placeholders to fill in

| Placeholder | Where | How to find it |
|---|---|---|
| `<OPTIPLEX_USER>` | docker-compose.yml | your login name on the OptiPlex |
| `<OPTIPLEX_IP>` | docker-compose.yml (2 places) | Firewalla → Devices → OptiPlex (should be a reserved IP) |
| `<N8N_ENCRYPTION_KEY>` | docker-compose.yml | run `openssl rand -hex 32` once and **store it in Vaultwarden before deploying** |
| `TZ` / `GENERIC_TIMEZONE` | docker-compose.yml | confirm `America/Chicago` |
| Port 5678 free | — | `docker ps` must not show it in use |

## Step 1 — OptiPlex: create folders

n8n runs as user 1000 inside the container, so the folders must belong to uid 1000:
```bash
mkdir -p ~/docker/n8n/data ~/docker/n8n/files
sudo chown -R 1000:1000 ~/docker/n8n
```

## Step 2 — Deploy in Portainer

Portainer → Stacks → Add stack → name `n8n` → paste `docker-compose.yml` with placeholders filled → Deploy.

## Step 3 — First login

Open `http://<OPTIPLEX_IP>:5678` → create the owner account (Vaultwarden). n8n will offer a free license key for extra community-edition features (workflow history, folders); it's worth accepting.

## Step 4 — First workflow (smoke test)

1. New workflow → add **Schedule Trigger** (every day, 8:00).
2. Add a step: **Send Email** (Gmail node) or a push service you already use.
3. Click **Execute workflow** to test it now, then toggle it **Active**.
4. Check that it runs at the right local time the next day. If it's off by hours, `GENERIC_TIMEZONE` is wrong.

## Things to know

- **Webhooks from the internet** (e.g. GitHub or a cloud service calling n8n) won't reach it. It's LAN-only on purpose. If you need that later, use something like a Cloudflare Tunnel for just the `/webhook` path, and plan it separately; don't open port 5678 on Firewalla.
- **Credentials** (Gmail, API keys) are encrypted with `N8N_ENCRYPTION_KEY`. Restoring a backup without that key = re-entering every credential.
- **Updates:** n8n releases often. Update by pulling a new image in Portainer (Stacks → n8n → Update with *Re-pull image*), and read the release notes for breaking changes first.

## Step 5 — Finish

- [ ] Add n8n to Homepage and Uptime Kuma.
- [ ] Add `~/docker/n8n/data` to the OptiPlex nightly backup script.
- [ ] Update the vault: Project Notes, Handoff notes, today's daily note.

## Open questions

- Which automations do you want first? That decides which accounts to connect.
