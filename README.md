# Project 3 – Hosting a Web Server on Raspberry Pi

This project demonstrates how to host a simple web server on a **Raspberry Pi** using Python's built-in HTTP server module, and serve a static HTML page over the local network.

## Overview

Using nothing but the Raspberry Pi's terminal and Python 3 (which ships with Raspberry Pi OS), we can turn the Pi into a lightweight web server capable of serving files like `index.html` to any device on the same network.

## Steps

### 1. Find the Raspberry Pi's IP Address

```bash
ifconfig
```

This command lists all network interfaces on the Pi. Look under `eth0` (wired) or `wlan0` (Wi-Fi) for the `inet` address — this is the IP that other devices will use to reach the Pi's web server.

### 2. Switch to the Root User

```bash
sudo -s
```

This elevates the current session to the `root` user, which is required for certain system-level operations (such as binding to port `80`, which is a privileged port). After running this, the prompt changes from `cyperduo@raspberrypi:~$` to `root@raspberrypi:/home/cyperduo#`.

### 3. Start the Web Server

```bash
python -m http.server 80
```

Breaking this down:
| Component | Purpose |
|---|---|
| **Python** | Runs Python's built-in `http.server` module, which turns the current directory into a web-servable folder — no extra installation needed. |
| **HTTP** | The protocol used to communicate between the Pi (server) and any browser (client) that connects to it. |
| **Port 80 / 8080** | The "door" through which the connection is made. Port `80` is the default HTTP port (so URLs don't need `:80` appended), while `8080` is a common alternative for non-root use. |

Once running, the terminal prints a log of every request the server receives, including the IP of the client, the request type (`GET`), the file requested, and the HTTP status code (e.g., `200` for success, `404` for file not found).

### 4. Create and Edit the Web Page

```bash
touch index.html      # creates a new empty file
ls                     # lists files/folders in the current directory to confirm it was created
nano index.html        # opens the file in the Nano text editor for editing
```

Inside `nano`, you can write your HTML content. To save and exit:

```
Ctrl + X   → prompts to save
Y          → confirm save
Enter      → confirm filename
```

Once saved, refreshing the browser at `http://<raspberry-pi-ip>/index.html` (or just `http://<raspberry-pi-ip>/`) will display the page.

## Command Reference

| Command | Description |
|---|---|
| `ifconfig` | Displays network interface details, including the Pi's IP address |
| `sudo -s` | Switches to the root user for elevated permissions |
| `python -m http.server 80` | Starts a simple HTTP server on port 80, serving the current directory |
| `touch <filename>` | Creates a new empty file |
| `ls` | Lists files and folders in the current directory |
| `nano <filename>` | Opens the file in the Nano editor for editing |
| `Ctrl + X` | Saves and exits the Nano editor |

## Notes

- The server only runs while the terminal session is active; closing it stops the server.
- Serving on port `80` requires root privileges, which is why the user switches to `root` first.
- This setup is intended for local network testing/learning — it is **not** a production-ready or secured web server.
