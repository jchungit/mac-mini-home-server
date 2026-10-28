# 🖥️ Mac Mini Home Server — 24/7 Self-Hosted Services Migration

Migrating my home lab services off a gaming desktop and onto an always-on, low-power **Mac mini** running everything in **Docker** containers. The goal: services that stay up when the desktop is off, recover on their own after a power outage, and are easy to update and monitor.

> 🔒 All IP addresses, hostnames, domains, VLAN IDs, and credentials have been intentionally left out of this write-up.

---

## 🎯 Why Migrate?

| Problem (Desktop as Server) | Solution (Mac Mini as Server) |
|---|---|
| Services went down whenever the gaming PC was off or rebooting | Dedicated 24/7 host with low idle power draw |
| Gaming load competed with server workloads | Workloads isolated on their own machine |
| Services installed as Windows apps, each updated differently | Every service in Docker with a compose file — one consistent way to update |
| Server sat on the same network segment as personal devices | Server placed on a dedicated **Homelab VLAN (Virtual Local Area Network)** with a fixed address |

---

## 🧱 Architecture

```
            Internet
               │
     ┌─────────┴──────────┐
     │  UniFi Gateway     │  firewall · region blocking · port forwards 80/443 only
     └─────────┬──────────┘
               │  2.5GbE switch
     ┌─────────┴──────────┐
     │   Mac mini (24/7)  │  Homelab VLAN · fixed IP · Tailscale subnet router + exit node
     │   OrbStack/Docker  │
     ├────────────────────┤
     │ Pi-hole + DoH      │  network-wide DNS (Domain Name System) filtering
     │ Nginx Proxy Mgr    │  reverse proxy + Let's Encrypt TLS
     │ Jellyfin           │  media streaming
     │ Immich             │  self-hosted photo backup (iCloud replacement)
     │ iCloud Photos Sync │  nightly pull of existing iCloud library
     │ Open WebUI         │  multi-model AI chat (cloud models via OpenRouter)
     │ Homepage           │  service dashboard with live widgets
     │ Watchtower         │  scheduled auto-updates + Telegram notifications
     │ Wazuh SIEM         │  see wazuh-soc-dashboard repo
     └────────────────────┘
```

---

## 🛠️ Build Steps

### Phase 1 — Base Setup
- Installed **Homebrew**, **OrbStack** (lightweight Docker runtime for Apple Silicon), and **Tailscale**
- Put container data on a dedicated external SSD so the internal drive stays clean and data is easy to move
- Connected the Mac mini and the desktop through a **2.5GbE managed switch** sharing a single wall jack
- Assigned the Mac mini a fixed address on the **Homelab VLAN**; verified SSH (Secure Shell) access from the desktop across VLANs
- **Unattended recovery:** enabled auto-login and disabled FileVault so the server comes back fully after a power outage without someone typing a password *(a deliberate availability-vs-confidentiality trade-off, documented below)*

### Phase 2 — Service Migration (one at a time)
Each service followed the same pattern: **stand up on the Mac mini → test → cut traffic over → keep the old copy as a rollback.**

| Service | What moved | Cut-over |
|---|---|---|
| **Pi-hole + dnscrypt-proxy** | DNS filtering with DNS-over-HTTPS (DoH) upstream | DHCP (Dynamic Host Configuration Protocol) DNS and Tailscale global DNS pointed to the new host |
| **Nginx Proxy Manager** | Reverse proxy + Let's Encrypt certificate | Port forwards repointed to the Mac mini |
| **Jellyfin** | Media server; library on a 4TB external drive | Proxy host repointed; tested over cellular |
| **iCloud Photos Sync** | Nightly library download | Old desktop copy kept as backup |
| **Immich** | New — automatic iPhone photo backup to my own drive | Mobile app pointed at the server |
| **Open WebUI** | AI chat front end | Moved container + data |
| **Tailscale exit node / subnet router** | Remote access to all home VLANs | Old desktop role switched off |

### Phase 3 — Operations
- **Homepage dashboard** with live widgets for Jellyfin, Immich, Pi-hole and Nginx Proxy Manager — one page to see everything
- **Watchtower** auto-updates only the low-risk containers (Pi-hole, dnscrypt-proxy) nightly, and pushes a **Telegram** message after each run
- Secrets (bot tokens) kept in a `.env` file locked to owner-only permissions (`chmod 600`), never in the compose file

---

## 🧯 Troubleshooting Log

| Symptom | Root Cause | Fix |
|---|---|---|
| YouTube / App Store slow on phone while at home | Phone was routing all traffic through the Tailscale **exit node** even on home Wi-Fi | Exit node set to *None* at home; only used on untrusted networks |
| Two Jellyfin servers showing in the app | Old desktop server still running during migration | Chose the Mac mini as primary; old server stopped and kept as a backup |
| Wazuh dashboard wouldn't start on 443 | Port already taken by the reverse proxy | Moved the dashboard to a different HTTPS port |
| Devices briefly lost ad-blocking | DHCP DNS temporarily set to Auto during testing | Restored Pi-hole as the only DNS server for client VLANs |

---

## ⚖️ Security Trade-offs I Made on Purpose

- **FileVault off + auto-login** → better *availability* (self-recovers after outages), lower *confidentiality* if the device is physically stolen. Mitigated by the machine staying indoors and every admin panel being reachable only over Tailscale.
- **Auto-updates only for low-risk containers** → avoids a surprise breaking change on Wazuh or Immich; those get updated manually.
- **Only 80/443 forwarded**; everything else (dashboards, SSH, admin panels) is reachable only over Tailscale.

---

## 📌 Next Steps
- [ ] Uptime Kuma for down-alerts to the same Telegram bot
- [ ] Backup plan for photos and service data (second local drive + offsite)
- [ ] Move Jellyfin to Tailscale-only and close 80/443 entirely

---

## 🧠 Skills Demonstrated
`Docker` · `Docker Compose` · `macOS server administration` · `Reverse proxy / TLS` · `DNS filtering` · `Tailscale (WireGuard VPN)` · `VLAN segmentation` · `Change management & rollback planning` · `Secrets handling` · `Service migration`
