# immortal

I had an old Lenovo laptop with a broken screen sitting around, so I took the display off and turned the bare board into a 24/7 headless Linux home server.

It runs a media setup for my family, stays reachable from anywhere without opening router ports, and sits at around 10–12W idle.

---

## Hardware

* **Machine:** Lenovo IdeaPad 310-15ISK (screen, internal keyboard, and internal Wi-Fi card removed)
* **CPU:** Intel Core i3-6006U
* **RAM:** 8 GB DDR4
* **GPU:** Intel HD 520 (used for Jellyfin hardware transcoding). The dedicated Nvidia GT 920M is turned off in BIOS (UMA-only) since it lacks NVENC anyway and just wastes power.
* **Storage:** 2.5" SATA SSD (originally tried a 3.5" desktop HDD, but laptop SATA ports only supply 5V and can't spin a 12V drive)
* **Network:** External dual-band USB Wi-Fi adapter

---

## Quirks & Fixes (Debian Minimal)

Running a stripped-down laptop without a screen caused a few annoying issues that needed manual fixes:

* **Boot freeze (`nouveau` panic):** Removing the internal screen made the open-source Nvidia driver hang on boot (`fifo: SCHED_ERROR`). Fixed it by adding `nouveau.modeset=0` to `/etc/default/grub`.
* **Airplane mode on reboot:** Disconnecting the laptop keyboard/lid made Linux think hardware airplane mode was stuck on, disabling Wi-Fi on every boot. Fixed by masking `systemd-rfkill` and adding an `ExecStartPre` override to `wpa_supplicant` to unblock and bring the interface up automatically.
* **Mobile hotspot timeouts (MTU):** API calls worked fine on home broadband (BSNL) but timed out on my phone's 5G hotspot. Testing with `ping -M do -s` showed the phone connection capped packets at 1372 bytes while Docker and Wi-Fi were sending 1500. Fixed by clamping both the Wi-Fi card and the Docker bridge to MTU `1280`.
* **DNS overwrites:** `dhcpcd` and Tailscale kept replacing `/etc/resolv.conf` when switching networks. Disabled Tailscale DNS (`--accept-dns=false`) and locked `1.1.1.1` / `8.8.8.8` via `/etc/resolv.conf.head` and `/etc/docker/daemon.json`.

---

## What's Running (Docker)

Everything runs inside Docker Compose using a shared `./data` mount so completed files use Linux hardlinks instead of taking double disk space:

* **Jellyfin:** Streams movies and TV shows (`host` networking + `/dev/dri` passed through for Intel QuickSync).
* **Seerr:** Simple web page for my family to browse and request movies or shows from their phones.
* **Radarr / Sonarr / Prowlarr:** Handles searching, tracking, and organizing files into `/data/movies` and `/data/tv`.
* **qBittorrent + FlareSolverr:** Downloads files and handles Cloudflare checks on trackers.
* **Tailscale:** Private WireGuard network so I can SSH from my Arch PC and my family can connect remotely without port forwarding.

---

## Go Portal (`main.go`)

A small Go server listening on `:4000` so my family only needs to bookmark one link (`http://immortal:4000`). It renders a simple page with buttons for watching or requesting, reads the incoming `r.Host` header dynamically (so it works over both local Wi-Fi and Tailscale IPs), and redirects to the right port. Runs in the background as a compiled binary via `systemd`.

---

## Quick Setup

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>

# Create local data folders (ignored by git)
mkdir -p data/{torrents,movies,tv} data/config/{jellyfin,qbittorrent,radarr,sonarr,prowlarr,seerr}

# Start the stack
docker compose up -d

# Build and run the Go portal
go build -o gateway main.go
./gateway
```
