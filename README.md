# Hosting a Simple Web Server on Raspberry Pi

This repository shows a small, secure-ish workflow to host a static website on a Raspberry Pi using Python. It includes:
- a minimal Python server script (server.py) that serves files from a directory with useful logging,
- a sample `index.html`,
- instructions that avoid unsafe recommendations (no blanket "sudo -s" to run a server).

WARNING: This is for local testing and learning only. Do NOT expose this setup directly to the internet without additional hardening (TLS, authentication, reverse proxy, firewall rules, monitoring).

## Quick overview

- Use `ip` or `hostname -I` to find your Pi's IP address (instead of the deprecated `ifconfig`).
- Run the server as a regular user on a non-privileged port (e.g., 8080).
- If you must use port 80, use a reverse proxy (nginx) or grant a specific binary permission to bind port 80 (see below) — do NOT run general shell sessions as root.
- Use systemd to run the server in the background and restart it automatically.

## Files included
- `server.py` — small Python3 HTTP server with threaded handling and improved logging.
- `index.html` — example web page served by the server.

## Find your Raspberry Pi IP address

Preferred commands:
- Show IPs assigned to all interfaces:
  - ip addr show
- Show the IPs on primary interface:
  - hostname -I
- Show the IP that will be used to reach the internet (useful when multiple networks):
  - ip route get 1.1.1.1 | awk '{print $7; exit}'

The address you use in a browser from another machine on the same LAN will be `http://<raspberry-pi-ip>:8080/` (if using port 8080).

## Running the server (recommended, simplest)

1. Create a directory to serve, e.g.:
   - mkdir -p ~/www
   - cp index.html ~/www/

2. Run the included server as your normal user:
   - python3 server.py --port 8080 --directory ~/www

3. Open a browser on any device in the same network at:
   - http://<raspberry-pi-ip>:8080/

Press Ctrl+C in the server terminal to stop it.

## Why not `sudo -s` + `python -m http.server 80`?

- Port 80 is a privileged port; binding it requires root privileges. Running a shell as root or running arbitrary Python as root increases risk.
- For local testing, use a non-privileged port (8080).
- For production or permanent hosting, use nginx as a reverse proxy and run a dedicated service with minimal privileges.

## Running on port 80 (if you truly need it)

Option A — recommended: use nginx as a reverse proxy that forwards port 80 to the server on 8080.

Option B — grant a single binary the right to bind to port 80 (example, careful and only for specific uses):
- sudo setcap 'cap_net_bind_service=+ep' /usr/bin/python3.10
  - This grants python the capability to bind privileged ports while not making the whole system root. Understand security implications before using.

Option C — run via systemd with `User=www-data` and an appropriate socket/proxy setup.

## Running the server in the background (systemd example)

Create a dedicated directory and copy files:
- sudo mkdir -p /opt/simple-http-server
- sudo cp server.py index.html /opt/simple-http-server/
- sudo chown -R pi:pi /opt/simple-http-server

Create systemd service file `/etc/systemd/system/simple-http-server.service`:
- (see example `simple-http-server.service` below — replace `pi` and paths as needed)

Enable and start:
- sudo systemctl daemon-reload
- sudo systemctl enable --now simple-http-server.service
- sudo journalctl -u simple-http-server.service -f

## Logging and debugging

- The script logs requests to stdout/stderr. When running with systemd, logs are available via `journalctl -u simple-http-server.service`.
- Check network/firewall rules (ufw, iptables) if you can't reach the server.

## Security notes

- This server is minimal and not hardened (no TLS, no authentication, no restriction of served paths).
- Do not use this to serve sensitive files or to expose to the public Internet.
- For public hosting, use a production-grade web server (nginx, Apache) or a managed hosting provider and add TLS (Let's Encrypt).

## Command reference

- ip addr show — show network interfaces and addresses
- hostname -I — print all local IP addresses
- python3 server.py --port 8080 --directory ~/www — run the provided server
- sudo systemctl enable --now simple-http-server.service — run the server as a background service
