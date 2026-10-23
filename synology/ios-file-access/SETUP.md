# iPhone / iPad File Access to the Synology — Setup Guide (DRAFT)

**Status:** draft, not done yet. Nothing here has been changed on any device yet.

## What and why

- **What:** browse, open, save and share files on the NAS from iPhone and iPad, at home and away.
- **Why no new server app:** the DS923+ already *is* a NAS. Synology's own apps plus Apple's built-in **Files** app cover this without another container to maintain. A Nextcloud stack would duplicate what the Synology already does, and it would be one more thing to update and back up.

| Want to… | Use |
|---|---|
| Browse/open/save any share from any app (Pages, PDFs, "Save to Files") | iOS **Files** app → SMB (Step 2) |
| Keep certain folders synced/available offline | **Synology Drive** app (Step 3) |
| Back up the iPhone camera roll | **Synology Photos** app (Step 4, optional) |
| Reach all of the above away from home | **VPN** (Step 5) |

## Step 1 — Synology: a personal account and SMB settings

1. Don't use `RayAdmin` on the phone. DSM → Control Panel → User & Group → Create → e.g. `ray` (and one per family member). Give **Read/Write** only on the shares you actually want on the phone (e.g. `home`, `documents`). Give `video` **Read only**, so a mis-swipe in the Files app can't delete movies. Store the passwords in Vaultwarden.
2. DSM → Control Panel → File Services → SMB:
   - [ ] SMB service enabled.
   - [ ] Advanced → Minimum SMB protocol: **SMB2**, Maximum: **SMB3**.
   - [ ] Advanced → enable the **Apple/macOS compatibility** options (named "Enable SMB extensions for macOS" or similar, depending on DSM version). They make the Files app faster and avoid odd filename issues.
3. DSM → Control Panel → User & Group → Advanced → **Enable user home service**. Each person then gets a private `home` folder.

## Step 2 — iPhone/iPad: Files app over SMB

1. Open **Files** → **Browse** → **⋯** (top right) → **Connect to Server**.
2. Server: `smb://10.55.2.65` → Connect → **Registered User** → the account from Step 1.
3. The NAS now appears under **Shared** in the sidebar. Any app's **Share → Save to Files** can save straight to the NAS.

Note: the Files app is good for occasional use but can be slow with folders holding thousands of photos. That's what Step 3–4 are for.

## Step 3 — Synology Drive (sync + offline)

1. DSM → Package Center → install **Synology Drive Server**.
2. Drive Admin Console → Team Folder → enable the shares you want in the app (your personal `home` is enabled automatically).
3. iPhone: App Store → **Synology Drive** → Server: `10.55.2.65`, your account.
4. Star or "make available offline" the folders you want on the phone when there's no connection.

## Step 4 — Synology Photos (optional: camera-roll backup)

DSM → Package Center → **Synology Photos**. iPhone: App Store → **Synology Photos** → sign in → Backup → turn on. This is a second copy alongside iCloud, not a replacement, unless you decide otherwise.

## Step 5 — Away from home: VPN, not port forwarding

Don't forward ports 445, 5000 or 5001 on Firewalla, and preferably don't use QuickConnect either. With a VPN, the phone behaves as if it's home, so Steps 2–4 (and Plex, Frigate, Homebox, n8n) all just work.

Pick **one**:

- **Firewalla VPN server (WireGuard)** — Firewalla app → VPN Server → enable WireGuard → add a profile per phone → scan the QR in the **WireGuard** iOS app. Nothing new to run; managed where the rest of the network already is.
- **Tailscale** — DSM → Package Center → **Tailscale**, plus the Tailscale iOS app. Works even if the home internet changes IP or sits behind CGNAT. To reach other servers through it, set the Synology as a subnet router.

Check after setting it up: on cellular (Wi-Fi off), VPN on → Files app opens the NAS.

**Note for the NordVPN group:** iOS allows only one VPN at a time. On the phone, the home VPN replaces NordVPN while it's on.

## Step 6 — Finish

- [ ] Each family member: own DSM account + Files app connection.
- [ ] Test at home and on cellular through the VPN.
- [ ] Update the vault: Project Notes, Handoff notes, today's daily note.

## Open questions

- If you'd still like a single self-hosted "Dropbox-style" app beyond what Synology Drive does (shared links to people outside the house, calendar/contacts sync, document editing), Nextcloud is the next step up. It could run on the OptiPlex with its storage on the NAS. Worth doing only if Drive falls short after a few weeks.
