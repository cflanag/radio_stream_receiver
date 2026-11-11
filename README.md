# Radio Stream Receiver (Raspberry Pi 3 B+ + DAC Pro)

Turn a Raspberry Pi into a network web-radio player. The Pi runs a **headless Chromium** showing
<http://nossl.fmstream.org/style.htm>, plays the audio through a **Raspberry Pi DAC Pro** into your stereo,
and you control it from any phone or tablet by opening a web page served by the Pi.

## How it works

```
 Phone / tablet browser ──(http :6080)──► noVNC (websockify) ──► x11vnc ──► Xvfb virtual screen
                                                                             └─ Chromium → fmstream.org page
                                                                                  └─ audio (ALSA) → DAC Pro → stereo
```

Why this design: if you simply opened fmstream.org on your phone, the audio would play *on the phone*.
Here the phone only acts as a remote screen/mouse. Chromium lives on the Pi, so the sound comes out of the stereo.

| Component | Role |
|-----------|------|
| Xvfb | Virtual display (no monitor needed) |
| Chromium | Loads the radio page, plays the streams |
| x11vnc | Shares that virtual display over VNC (password protected) |
| noVNC + websockify | Serves a VNC client as a web page on port 6080 |
| ALSA + `rpi-dacpro` overlay | Audio out through the DAC Pro |
| systemd | Starts everything at boot and restarts it if it dies |

## Requirements

- Raspberry Pi 3 B+ with Raspberry Pi DAC Pro HAT fitted
- microSD card (8 GB+), power supply, network (Ethernet recommended, Wi‑Fi works)
- Stereo connected to the DAC Pro's RCA / 3.5 mm line out
- A computer to flash the card and SSH in

> Tested design for Raspberry Pi OS Lite (Bookworm or newer). Commands assume the default user is `pi`;
> if you chose another username, replace `pi` / `/home/pi` everywhere below.

---

## 1. Flash the OS

1. Install **Raspberry Pi Imager**.
2. Choose **Raspberry Pi OS Lite (64‑bit)** (32‑bit also fine).
3. Click the gear / "Edit settings" and set:
   - Hostname: `radio`
   - Username `pi` and a password
   - Wi‑Fi (if not using Ethernet)
   - Enable **SSH**
4. Write the card, boot the Pi, then connect:

```bash
ssh pi@radio.local
```

## 2. Update the system

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

## 3. Enable the DAC Pro

Edit the boot config (on Bookworm it is `/boot/firmware/config.txt`; on older releases `/boot/config.txt`):

```bash
sudo nano /boot/firmware/config.txt
```

Find the line `dtparam=audio=on` and change it to `dtparam=audio=off`, then add at the bottom:

```ini
dtoverlay=rpi-dacpro
```

If you see a `dtoverlay=vc4-kms-v3d` line, add `,noaudio` to it so HDMI audio doesn't grab the default card:

```ini
dtoverlay=vc4-kms-v3d,noaudio
```

Reboot and verify the card shows up:

```bash
sudo reboot
# after reconnecting:
aplay -l
```

You should see the DAC as card 0. **Note the short name after `card 0:`**: on newer kernels the card is called `Pro` (e.g. `card 0: Pro [RPi DAC Pro]`), on older ones `sndrpihifiberry`. You need that name in the next step. If no DAC card is listed, recheck the overlay line.

## 4. Make the DAC the default ALSA device

```bash
sudo nano /etc/asound.conf
```

Use the card name from `aplay -l` in step 3 (shown here as `Pro`; use `sndrpihifiberry` if that is what yours shows):

```conf
pcm.!default {
    type plug
    slave.pcm "hw:Pro"
}
ctl.!default {
    type hw
    card Pro
}
```

Test with a short tone (Ctrl+C to stop; keep your stereo volume low first):

```bash
speaker-test -c 2 -t sine -f 440
```

Set the output level (the DAC Pro exposes a `Digital` control; 100% is full line level):

```bash
alsamixer          # F6 not needed; press F5 for all controls, adjust "Digital"
sudo alsactl store # remember the setting across reboots
```

## 5. Install the software

```bash
sudo apt install -y xvfb chromium x11vnc novnc websockify openbox
```

(`openbox` is a tiny window manager so Chromium gets a properly sized window.)
If `chromium` isn't found on your release, use `chromium-browser`, and change the path in step 7 accordingly.

## 6. Set a VNC password

This protects the screen from other devices on your network.

```bash
x11vnc -storepasswd
```

Enter a password; it is saved to `/home/pi/.vnc/passwd`.

## 7. Create the services

### 7.1 Virtual display

```bash
sudo tee /etc/systemd/system/radio-xvfb.service >/dev/null <<'EOF'
[Unit]
Description=Radio: virtual display (Xvfb)
After=network.target

[Service]
User=pi
ExecStart=/usr/bin/Xvfb :1 -screen 0 1024x768x24 -nolisten tcp
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

### 7.2 Window manager + Chromium

```bash
sudo tee /etc/systemd/system/radio-browser.service >/dev/null <<'EOF'
[Unit]
Description=Radio: Chromium showing fmstream
After=radio-xvfb.service sound.target
Requires=radio-xvfb.service

[Service]
User=pi
Environment=DISPLAY=:1
ExecStartPre=/bin/sleep 2
ExecStart=/bin/sh -c '/usr/bin/openbox & exec /usr/bin/chromium \
  --no-sandbox --disable-gpu --disable-dev-shm-usage \
  --no-first-run --no-default-browser-check --disable-infobars \
  --disable-session-crashed-bubble --noerrdialogs \
  --autoplay-policy=no-user-gesture-required \
  --allow-running-insecure-content \
  --user-data-dir=/home/pi/.config/radio-chromium \
  --window-position=0,0 --window-size=1024,768 \
  --start-maximized --kiosk \
  "http://nossl.fmstream.org/style.htm"'
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

> `--allow-running-insecure-content` is **required**: without it the page loads but stations (plain `http://` streams) won't play.
> Use the `http://` address exactly as shown: `https://fmstream.org` did not work in testing.
>
> `--no-sandbox` is used because this is a single-purpose appliance on a trusted LAN.
> Do not expose it to the internet.

### 7.3 VNC server (shares the virtual display)

```bash
sudo tee /etc/systemd/system/radio-vnc.service >/dev/null <<'EOF'
[Unit]
Description=Radio: x11vnc
After=radio-xvfb.service
Requires=radio-xvfb.service

[Service]
User=pi
ExecStart=/usr/bin/x11vnc -display :1 -rfbauth /home/pi/.vnc/passwd -rfbport 5900 -forever -shared -noxdamage -localhost
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

`-localhost` means only the web gateway below can reach VNC; nothing connects to port 5900 from outside.

### 7.4 Web gateway (noVNC)

```bash
sudo tee /etc/systemd/system/radio-web.service >/dev/null <<'EOF'
[Unit]
Description=Radio: noVNC web gateway
After=radio-vnc.service
Requires=radio-vnc.service

[Service]
User=pi
ExecStart=/usr/bin/websockify --web=/usr/share/novnc 6080 localhost:5900
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

### 7.5 Convenience redirect page (optional)

Makes `http://radio.local:6080/` open the radio screen straight away, scaled to fit your phone:

```bash
sudo tee /usr/share/novnc/index.html >/dev/null <<'EOF'
<!doctype html><meta charset="utf-8">
<meta http-equiv="refresh" content="0;url=vnc.html?autoconnect=true&resize=scale&reconnect=true">
EOF
```

## 8. Start it

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now radio-xvfb radio-browser radio-vnc radio-web
systemctl status radio-xvfb radio-browser radio-vnc radio-web --no-pager
```

All four should say `active (running)`.

## 9. Use it

From a phone or tablet on the same network, open (power-off page is at `:8080`, see *Safe shutdown*):

```
http://radio.local:6080/
```

(or `http://<pi-ip-address>:6080/`). Enter the VNC password from step 6. You'll see the fmstream page;
tap a station and the sound plays from the stereo. You can close the phone browser — the radio keeps
playing until you stop it. Reconnect any time to change station.

Tips:
- Bookmark it / "Add to Home Screen" on the phone for one-tap access.
- Give the Pi a fixed IP (router DHCP reservation) if `radio.local` doesn't resolve on Android.
- Change volume with the Pi's `Digital` ALSA control (`alsamixer`) or on your stereo.

---

## Safe shutdown (power-off button)

Never just pull the power: the SD card can be corrupted. Use the web button below to shut down cleanly.

A tiny web page on port 8080 with **Shut down** and **Reboot** buttons (each asks to confirm).

```bash
sudo tee /usr/local/bin/radio-power.py >/dev/null <<'EOF'
#!/usr/bin/env python3
"""Minimal power page: GET shows buttons, POST /shutdown or /reboot acts."""
import subprocess
from http.server import BaseHTTPRequestHandler, HTTPServer

PAGE = """<!doctype html><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Radio power</title>
<body style="font-family:sans-serif;text-align:center;margin-top:12vh">
<h2>Radio power</h2>
<p><a href="http://{host}:6080/">&larr; Back to radio</a></p>
<form method="post" action="/shutdown" onsubmit="return confirm('Shut down the radio?')">
<button style="font-size:2em;padding:.6em 1.2em;background:#c0392b;color:#fff;border:0;border-radius:.4em">Shut down</button></form>
<br>
<form method="post" action="/reboot" onsubmit="return confirm('Reboot the radio?')">
<button style="font-size:1.2em;padding:.5em 1em">Reboot</button></form>
</body>"""

DONE = "<!doctype html><meta charset='utf-8'><meta name='viewport' content='width=device-width,initial-scale=1'><body style='font-family:sans-serif;text-align:center;margin-top:20vh'><h2>{}</h2><p>Wait until the green LED stops flashing before removing power.</p>"

class H(BaseHTTPRequestHandler):
    def _send(self, body, code=200):
        b = body.encode()
        self.send_response(code)
        self.send_header("Content-Type", "text/html; charset=utf-8")
        self.send_header("Content-Length", str(len(b)))
        self.end_headers()
        self.wfile.write(b)

    def do_GET(self):
        host = (self.headers.get("Host") or "radio.local").split(":")[0]
        self._send(PAGE.format(host=host))

    def do_POST(self):
        if self.path == "/shutdown":
            self._send(DONE.format("Shutting down..."))
            subprocess.Popen(["/usr/bin/systemctl", "poweroff"])
        elif self.path == "/reboot":
            self._send(DONE.format("Rebooting..."))
            subprocess.Popen(["/usr/bin/systemctl", "reboot"])
        else:
            self._send("Not found", 404)

    def log_message(self, *a):
        pass

HTTPServer(("0.0.0.0", 8080), H).serve_forever()
EOF
sudo chmod +x /usr/local/bin/radio-power.py

sudo tee /etc/systemd/system/radio-power.service >/dev/null <<'EOF'
[Unit]
Description=Radio: web power button
After=network.target

[Service]
ExecStart=/usr/local/bin/radio-power.py
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now radio-power
```

Open `http://radio.local:8080/` on your phone and bookmark / "Add to Home Screen" it next to the radio link.
When the Pi's green activity LED stops flashing it is safe to unplug the power.

> This service runs as root and has **no password**: anyone on your home network who opens that page can
> power the Pi off. Fine for a home LAN; don't expose port 8080 to the internet.

## Troubleshooting

| Symptom | Check |
|---------|-------|
| `aplay -l` doesn't list `sndrpihifiberry` | `dtoverlay=rpi-dacpro` present? HAT seated properly? Reboot. |
| Page loads but silent | `speaker-test` works? Station may be offline; try another. Check `Digital` isn't muted in `alsamixer`. Confirm `/etc/asound.conf` uses the exact card name from `aplay -l`. |
| "Redirected you too many times" (ERR_TOO_MANY_REDIRECTS) | Keep the `http://nossl.fmstream.org/style.htm` address (the `https://` one does not work). Reset the browser profile: `sudo systemctl stop radio-browser && rm -rf ~/.config/radio-chromium && sudo systemctl start radio-browser`. |
| Chromium not starting | `journalctl -u radio-browser -e`. Try `chromium-browser` if `chromium` is missing. |
| Black screen in noVNC | `systemctl status radio-browser`; wait ~10 s after boot. |
| Can't connect on :6080 | `systemctl status radio-web radio-vnc`; same Wi‑Fi/network as the Pi? |
| Stutter / slow | Use Ethernet; close extra tabs; lower Xvfb size to `800x600x24` in `radio-xvfb.service`. |
| Station needs a Flash/HTTPS-only player | Some stations only work with certain codecs; Chromium on Pi supports MP3/AAC/Ogg. |

Useful commands:

```bash
sudo systemctl restart radio-browser     # reload the page / recover a hung browser
journalctl -u radio-browser -f           # live logs
sudo systemctl disable --now radio-web radio-vnc radio-browser radio-xvfb   # turn it all off
```

## Security notes

- The VNC password is the only protection; keep the Pi on a trusted home network and don't port‑forward 6080.
- noVNC traffic is unencrypted HTTP on your LAN, which matches the HTTP-only fmstream page.

## Uninstall

```bash
sudo systemctl disable --now radio-power radio-web radio-vnc radio-browser radio-xvfb
sudo rm /usr/local/bin/radio-power.py
sudo rm /etc/systemd/system/radio-*.service
sudo systemctl daemon-reload
```

## Publishing this guide to GitHub

```bash
cd ~/claude_projects/radio_stream_receiver
git init
git add README.md
git commit -m "Add Raspberry Pi web radio receiver install guide"
git branch -M main
git remote add origin https://github.com/<your-username>/radio_stream_receiver.git
git push -u origin main
```

(Create the empty `radio_stream_receiver` repository on github.com first.)

## License

MIT — do whatever you like with this guide.
