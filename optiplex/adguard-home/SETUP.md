# AdGuard Home on the OptiPlex — Setup Guide (DRAFT)

**Status:** draft, not deployed. Nothing here has been run on any server yet.

## What and why

- **What:** AdGuard Home is a DNS server that blocks ads, trackers and malware domains for every device in the house, including smart TVs and phones, with no app needed on each device. It also gives you a log of what each device is looking up.
- **Why the OptiPlex:** it's light (a few % CPU, ~100 MB RAM) and the OptiPlex is already the always-on box for Homepage and Uptime Kuma.
- **Firewalla overlap:** Firewalla has its own Ad Block. Run only **one** of them, or debugging "why won't this site load" gets confusing. The plan is AdGuard does the blocking and Firewalla's Ad Block is turned off.

## ⚠️ Read this first: DNS is critical

Once the house uses AdGuard for DNS, **if the OptiPlex is off, the internet "breaks"** for everyone: sites won't load, even though the connection is fine. The setup below avoids this by having Firewalla forward to AdGuard (Step 5), so undoing it is one setting in the Firewalla app. Write that setting down before you start.

## Placeholders to fill in

| Placeholder | Where | How to find it |
|---|---|---|
| `<OPTIPLEX_USER>` | docker-compose.yml | your login name on the OptiPlex |
| `<OPTIPLEX_IP>` | Steps 4–5 | Firewalla → Devices → OptiPlex. It **must** be a reserved (fixed) IP |
| Port 53 free | — | `sudo ss -tulpn | grep ':53 '` (see Step 1) |
| Ports 3003, 8083 free | — | `docker ps` must not show them in use |

## Step 1 — OptiPlex: free up port 53

Ubuntu runs its own mini DNS service (`systemd-resolved`) on port 53, which blocks AdGuard from starting. Check:
```bash
sudo ss -tulpn | grep ':53 '
```
If you see `systemd-resolved`, turn off just its port-53 listener (the OptiPlex still gets DNS normally):
```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo nano /etc/systemd/resolved.conf.d/adguardhome.conf
```
Paste:
```
[Resolve]
DNSStubListener=no
```
Then:
```bash
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
sudo ss -tulpn | grep ':53 '      # should now print nothing
ping -c 2 google.com              # the OptiPlex itself should still resolve names
```

## Step 2 — OptiPlex: create folders

```bash
mkdir -p ~/docker/adguardhome/work ~/docker/adguardhome/conf
```

## Step 3 — Deploy in Portainer

Portainer → Stacks → Add stack → name `adguard` → paste `docker-compose.yml` with placeholders filled → Deploy.

## Step 4 — First-time setup wizard

Open `http://<OPTIPLEX_IP>:3003`.

1. **Admin Web Interface:** listen on *All interfaces*, port **80**. Leave it as 80: inside the container it's 80, and the stack maps that to 8083 on the OptiPlex.
2. **DNS server:** *All interfaces*, port **53**.
3. Create the admin login and store it in Vaultwarden.
4. After the wizard finishes, the UI lives at `http://<OPTIPLEX_IP>:8083`. Port 3003 is no longer used.

Settings to set in the UI:

- **Settings → DNS settings → Upstream DNS servers:** replace the default with
  ```
  https://dns.quad9.net/dns-query
  https://cloudflare-dns.com/dns-query
  ```
  and choose **Load-balancing**. This encrypts the house's DNS lookups on the way out.
- **Bootstrap DNS:** `9.9.9.9` and `1.1.1.1`.
- **Filters → DNS blocklists:** the default *AdGuard DNS filter* is a good start. Optionally add *OISD Big* (Add blocklist → Choose from the list). Don't stack 10 lists; more lists means more broken sites.
- **Settings → General → Query logs / Statistics:** keep 7 days each (90 days fills the OptiPlex disk with logs).

Test before switching the house over. From a Mac or iPhone, set DNS manually to `<OPTIPLEX_IP>` for a few minutes and check that sites load and ads drop. Or from any terminal:
```bash
nslookup doubleclick.net <OPTIPLEX_IP>     # should return 0.0.0.0
nslookup apple.com <OPTIPLEX_IP>           # should return a real address
```

## Step 5 — Firewalla: send the house's DNS through AdGuard

1. Firewalla app → **Ad Block: off** (on the whole network). Also turn off **Family Protect** if it's on, since it does the same job.
2. Firewalla app → Network → your LAN → DNS → set the DNS server to `<OPTIPLEX_IP>` only. This tells Firewalla to forward lookups to AdGuard. Devices keep using Firewalla as their DNS, so Firewalla's per-device rules keep working.
3. **Rollback** if anything goes wrong: put the DNS setting back to what it was (note it here: `__________`).

**Trade-off:** in this mode, AdGuard sees every lookup as coming from Firewalla, so you lose per-device stats in AdGuard. If you later want per-device stats, have Firewalla's DHCP hand out `<OPTIPLEX_IP>` to devices directly. That's more visibility but more fragile, and best done once there's a second AdGuard (see Open questions).

**NordVPN group:** devices in Firewalla's NordVPN group may use NordVPN's DNS instead and skip AdGuard. That's expected. Check one VPN'd device after the change.

## Step 6 — Finish

- [ ] Add AdGuard Home to Homepage (it has a built-in AdGuard widget showing queries and blocked %) and Uptime Kuma. In Uptime Kuma, use a **DNS** monitor against `<OPTIPLEX_IP>`, not just HTTP.
- [ ] Add `~/docker/adguardhome/conf` to the OptiPlex nightly backup script.
- [ ] Update the vault: Project Notes, Handoff notes, today's daily note.

## Open questions

- **Redundancy:** to remove the "OptiPlex off = no internet" risk, run a second AdGuard on the Cubi and keep the two in sync with `adguardhome-sync`. That's worth doing once this one has run cleanly for a couple of weeks.
- **Local names:** AdGuard can give servers friendly names (Filters → DNS rewrites), e.g. `plex.home` → Cubi IP, `nas.home` → `10.55.2.65`. Optional, but it pairs nicely with Homepage.
