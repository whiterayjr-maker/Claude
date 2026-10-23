# Homebox on the OptiPlex — Setup Guide (DRAFT)

**Status:** draft, not deployed. Nothing here has been run on any server yet.

## What and why

- **What:** Homebox is a home inventory: what you own, where it lives, serial numbers, purchase price, warranty dates, receipts and manuals. It's useful for insurance claims, "which box is the HDMI cable in", and warranty lookups. It can print QR labels for bins and shelves; scan one with the iPhone camera to see what's inside.
- **Why the OptiPlex:** it's tiny (one container, ~50 MB RAM, built-in database). No reason to put it anywhere special.

## Placeholders to fill in

| Placeholder | Where | How to find it |
|---|---|---|
| `<OPTIPLEX_USER>` | docker-compose.yml | your login name on the OptiPlex |
| `TZ` | docker-compose.yml | confirm `America/Chicago` |
| `<API_KEY_PEPPER>` | docker-compose.yml | run `openssl rand -base64 48` once and store the result in Vaultwarden |
| Port 7745 free | — | `docker ps` must not show it in use |

## Step 1 — OptiPlex: create the folder

```bash
mkdir -p ~/docker/homebox/data
```

## Step 2 — Deploy in Portainer

Portainer → Stacks → Add stack → name `homebox` → paste `docker-compose.yml` with placeholders filled → Deploy.

## Step 3 — Create your account, then lock sign-ups

1. Open `http://<optiplex-ip>:7745` → **Register** → create your account (Vaultwarden).
2. Back in Portainer → Stacks → `homebox` → Editor: change `HBOX_OPTIONS_ALLOW_REGISTRATION=true` to `false` → **Update the stack**. No one else on the network can create an account after this.
3. Want family members to use it too? Use Homebox's built-in invite (Profile → Generate invite link) instead of turning registration back on. Invited people share the same inventory.

## Step 4 — Getting started (suggested structure)

- **Locations** = rooms and storage spots: `House → Garage → Shelf A`, `House → Office → Closet`.
- **Labels/Tags** = cross-cutting groups: `Electronics`, `Tools`, `Warranty`, `Insurance`.
- Start with the expensive stuff (TVs, computers, appliances, tools). Add photos and the receipt, and fill in **Warranty expires**.
- **Labels for bins:** Tools → Label maker → print QR labels, then stick them on storage bins.

On the iPhone, open the site in Safari → Share → **Add to Home Screen**. It then works like an app, and the camera can scan Homebox QR codes.

## Step 5 — Finish

- [ ] Add Homebox to Homepage and Uptime Kuma.
- [ ] Add `~/docker/homebox/data` to the OptiPlex nightly backup script. This one really matters: it holds your receipts and photos.
- [ ] Update the vault: Project Notes, Handoff notes, today's daily note.

## Open questions

- Away-from-home access (e.g. checking a serial number at a store): same answer as the other apps. Use the VPN, don't open a port.
