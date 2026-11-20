# RADXA ePoster Installation Guide

Step-by-step installation for additional RADXA devices using the tested Debian/X11 desktop setup.

- **Repository:** [Bhavi-1266/eposter](https://github.com/Bhavi-1266/eposter)
- **Last reviewed:** 20 November 2026
- **Repository revision reviewed:** `904eb818e586dc3531f8e0eac5a66c872c7cd9c6`
- **Suggested location in the repository:** `docs/RADXA_EPOSTER_INSTALLATION.md`

> **Installation order matters:** install the desktop-readiness helper and the per-device X11 service override before running `installer.py`. The reviewed repository still assumes `DISPLAY=:0` and waits for the `X0` socket. The override below replaces those assumptions with the actual settings from each device.

This guide includes the startup fix that waits for a usable fullscreen desktop before creating the first mpv window. It also includes the optional still-image aspect-ratio correction. These are separate changes: desktop readiness addresses the incorrect first window after reboot; image fitting controls whether still images stretch to fill the media area.

## Contents

1. [Prepare the device's desktop](#1-prepare-the-devices-desktop)
2. [Install initial packages and clone the repository](#2-install-initial-packages-and-clone-the-repository)
3. [Create the configuration for this device](#3-create-the-configuration-for-this-device)
4. [Install the desktop-readiness helper](#4-install-the-desktop-readiness-helper)
5. [Generate the per-device X11 service settings](#5-generate-the-per-device-x11-service-settings)
6. [Run the full installer](#6-run-the-full-installer)
7. [Verify playback and the management portal](#7-verify-playback-and-the-management-portal)
8. [Reboot and check the first mpv instance](#8-reboot-and-check-the-first-mpv-instance)

## 1. Prepare the device's desktop

Use the same tested RADXA OS and X11 desktop setup as the working device.

Before installing:

- Connect the monitor, keyboard, and network.
- Log into the desktop using the account that will run ePoster, such as `rock`.
- Set the monitor to the intended landscape orientation and the same resolution used on the working device.
- Enable **automatic desktop login** for that account so that rebooting reaches the desktop without someone entering a password. The project installer does not configure automatic login.

**Run the setup commands from a terminal opened on the RADXA desktop, as the normal desktop user.** In particular, generate the X11 settings in Step 5 from this local desktop terminal rather than an SSH or root shell.

Check the session:

```bash
printf 'User: %s\nUID: %s\nSession: %s\nDisplay: %s\nXauthority: %s\n' \
    "$(id -un)" \
    "$(id -u)" \
    "$XDG_SESSION_TYPE" \
    "$DISPLAY" \
    "$XAUTHORITY"
```

`Session` should be:

```text
x11
```

If the desktop is using Wayland, select an X11 session and log in before continuing. The display could be `:0` or `:1`; the commands in Step 5 capture the value reported by that device.

If you change the automatic-login configuration, verify that the intended X11 desktop starts after a reboot before capturing its settings.

## 2. Install initial packages and clone the repository

Install the tools needed before the project installer runs:

```bash
sudo apt-get update

sudo apt-get install -y \
    git \
    python3 \
    nano \
    x11-utils \
    x11-xserver-utils
```

`x11-utils` supplies `xprop`, which the readiness helper uses. The reviewed project installer does not install that package itself.

Clone into the logged-in user's home directory:

```bash
cd "$HOME"

git clone --branch main https://github.com/Bhavi-1266/eposter.git

cd eposter
```

Run `git clone` as the normal desktop user, without `sudo`, so that the checkout belongs to the correct account.

These instructions are for a fresh installation. If `$HOME/eposter` already exists, use that checkout intentionally rather than cloning over it or replacing its device configuration.

## 3. Create the configuration for this device

On the fresh clone:

```bash
cd "$HOME/eposter"

cp config.example.json config.json

chmod 600 config.json

nano config.json
```

Update these fields in the existing JSON structure:

| Setting | What to enter |
| --- | --- |
| Top-level `hardware_id` | The identity assigned to this physical board. |
| `username` | Your chosen portal login username. |
| `password` | Your chosen portal login password; replace the example password. |
| `api.poster_api_url` | The working poster API endpoint for the deployment. |
| `api.poster_token` | The appropriate API token, if required. |
| `api.timezone` | The timezone used by the API for timestamps without a timezone offset. Use the same setting as the working deployment. |
| `display.screen_number` | **The screen number assigned to this board in the API.** |
| `display.rotation_degree` | `0` for upright content on a normal landscape desktop, or the intended value from the working setup. |
| `display.Mode` | `"Time"` for scheduled posters. |
| `display.video_hwdec` | Keep `"auto"` unless the working board uses another verified setting. |
| `display.video_profile` | Keep `"default"` for the ROCK 5C setup covered by this guide. |
| Wi-Fi fields | Enter the required network details if using the application's Wi-Fi connection management. |

**The current code selects scheduled content using `display.screen_number`.** Top-level `hardware_id` is a separate board identity. Changing only `hardware_id` does not change the selected screen schedule.

For example, a board assigned to screen 2 might have this top-level field:

```json
"hardware_id": 2
```

and this field inside its existing `"display"` object:

```json
"screen_number": 2
```

These are illustrative fields, not a complete replacement configuration. The numbers can differ if the deployment assigns physical board identities and screen numbers separately.

Save in Nano with **Ctrl+O**, press **Enter**, then exit with **Ctrl+X**.

Validate the JSON without printing passwords or tokens:

```bash
python3 -m json.tool config.json >/dev/null
```

If this reports a JSON error, correct it before continuing. Keep each board's real credentials and tokens in its ignored `config.json`; this guide contains no deployment secrets.

### Optional: carry over the still-image proportion fix

The reviewed repository stretches still images to fill the media area. If the working board uses the earlier `ImageOps.pad` correction, apply the following change to give the new device the same aspect-preserving behavior:

```bash
cd "$HOME/eposter"

python3 - <<'PY'
from pathlib import Path

path = Path("helper/display_handler.py")
text = path.read_text()

old = "resized = image.resize(size, Image.Resampling.LANCZOS)"
new = (
    'resized = ImageOps.pad(image, size, '
    'method=Image.Resampling.LANCZOS, color="black", '
    'centering=(0.5, 0.5))'
)

if text.count(old) == 1:
    path.write_text(text.replace(old, new, 1))
    print("Applied aspect-preserving image fitting.")
elif new in text:
    print("Image fitting fix is already present.")
else:
    raise SystemExit("Image preparation code has changed; replacement was not applied.")
PY
```

This keeps still-image proportions and adds black margins when the image and screen shapes differ. It does not redesign a portrait poster into a landscape poster.

**Software-update consideration:** this optional change edits a tracked source file. The portal updater requires a clean checkout, so incorporate the rendering change into the repository before relying on that updater across the fleet. Do not discard a working local rendering fix merely to clear the updater's warning. The startup helper and service override in Steps 4 and 5 are stored outside the checkout.

## 4. Install the desktop-readiness helper

The helper checks that:

- The service account can connect to the X11 session.
- A live window manager is present.
- The window manager advertises fullscreen support.
- At least one connected display has an active resolution.

It proceeds when those conditions pass, with a bounded wait if the desktop is still starting.

Paste the **entire block**:

```bash
sudo install -d -m 755 /usr/local/lib/eposter

sudo tee /usr/local/lib/eposter/wait_x11_ready.py >/dev/null <<'PY'
#!/usr/bin/python3
import os
import re
import subprocess
import sys
import time

os.environ["LC_ALL"] = "C"


def query(*args):
    try:
        return subprocess.check_output(
            args, text=True, stderr=subprocess.DEVNULL, timeout=3
        )
    except (OSError, subprocess.CalledProcessError, subprocess.TimeoutExpired):
        return ""


def window_id(text):
    match = re.search(r"window id #\s*(0x[0-9a-fA-F]+)", text)
    return int(match.group(1), 16) if match else 0


def desktop_ready():
    root = query("/usr/bin/xprop", "-root",
                 "_NET_SUPPORTING_WM_CHECK", "_NET_SUPPORTED")
    wm = window_id(root)
    if not wm:
        return False, "X11 session or window manager unavailable"

    child = query("/usr/bin/xprop", "-id", hex(wm),
                  "_NET_SUPPORTING_WM_CHECK")
    if window_id(child) != wm:
        return False, "window manager not ready"

    atoms = set(re.findall(r"_NET_[A-Z0-9_]+", root))
    if not {"_NET_WM_STATE", "_NET_WM_STATE_FULLSCREEN"} <= atoms:
        return False, "window manager has not advertised fullscreen support"

    output = query("/usr/bin/xrandr", "--current")
    active = [line for line in output.splitlines()
              if re.match(r"^\S+\s+connected\b", line)
              and re.search(r"\b[1-9]\d*x[1-9]\d*[+-]\d+[+-]\d+", line)]
    if not active:
        return False, "no active display output"
    return True, "WM=" + hex(wm) + "; " + "; ".join(active)


def main():
    for tool in ("/usr/bin/xprop", "/usr/bin/xrandr"):
        if not os.access(tool, os.X_OK):
            print("ePoster: missing " + tool, file=sys.stderr, flush=True)
            return 1

    print("ePoster: waiting for desktop fullscreen support", flush=True)
    deadline = time.monotonic() + 120
    detail = "desktop unavailable"
    while time.monotonic() < deadline:
        ready, detail = desktop_ready()
        if ready:
            print("ePoster: desktop ready; " + detail, flush=True)
            return 0
        time.sleep(1)
    print("ePoster: startup readiness timed out: " + detail,
          file=sys.stderr, flush=True)
    return 1


if __name__ == "__main__":
    sys.exit(main())
PY

sudo chmod 644 /usr/local/lib/eposter/wait_x11_ready.py
```

## 5. Generate the per-device X11 service settings

This block captures the actual display, Xauthority path, user ID, and checkout path. It replaces the original `X0` startup check with the desktop-readiness helper.

**Paste the entire block in the local desktop terminal:**

```bash
(
set -eu

if [ "$(id -u)" -eq 0 ] || [ "${XDG_SESSION_TYPE:-}" != "x11" ] || [ -n "${SSH_CONNECTION:-}" ]; then
    echo "Run this as the logged-in display user in a local X11 desktop terminal."
    exit 1
fi

cd "$HOME/eposter"
EP_DIR="$(pwd -P)"
EP_UID="$(id -u)"
EP_XAUTH="${XAUTHORITY:-$HOME/.Xauthority}"
: "${DISPLAY:?DISPLAY is empty; open a local desktop terminal}"

if [ ! -r "$EP_XAUTH" ]; then
    echo "Cannot read the X11 authority file: $EP_XAUTH"
    exit 1
fi

env DISPLAY="$DISPLAY" XAUTHORITY="$EP_XAUTH" \
    /usr/bin/python3 /usr/local/lib/eposter/wait_x11_ready.py

sudo install -d -m 755 /etc/systemd/system/eposter-display.service.d

sudo tee /etc/systemd/system/eposter-display.service.d/99-desktop-ready.conf >/dev/null <<EOF
[Service]
Environment="DISPLAY=$DISPLAY"
Environment="XAUTHORITY=$EP_XAUTH"
Environment="XDG_RUNTIME_DIR=/run/user/$EP_UID"
Environment="DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$EP_UID/bus"
TimeoutStartSec=150
ExecStartPre=
ExecStartPre=/usr/bin/test -x "$EP_DIR/venv/bin/python3"
ExecStartPre=/usr/bin/python3 /usr/local/lib/eposter/wait_x11_ready.py
EOF
)
```

You should see a message beginning with:

```text
ePoster: desktop ready;
```

**If this step fails, resolve the error before running the installer.** The block stops before writing the override when its readiness check fails.

The `venv` path in the service configuration does not need to exist yet. The installer creates it before starting the service. The empty `ExecStartPre=` clears the previous pre-start commands; the following lines install the replacement checks.

The override is generated for this device at installation time. If the desktop account, login manager, or X11 display assignment changes later, rerun this step from the new local X11 session.

## 6. Run the full installer

Only after Steps 4 and 5 complete successfully, run:

```bash
cd "$HOME/eposter"

sudo EPOSTER_USER="$(id -un)" python3 installer.py
```

The full installer:

- Installs the remaining system packages, including mpv and FFmpeg.
- Creates and validates a versioned Python virtual environment.
- Activates that environment through the `venv` path.
- Installs and enables the admin and display services.
- Starts the services and checks for a fresh player heartbeat.

Wait for:

```text
[SUCCESS] Services restarted. Configuration and media cache preserved.
```

Use the full installer command for a fresh installation. `--services-only` expects an existing working Python environment. There is no need to manually create a virtual environment or repeat the earlier recovery commands on a healthy fresh install.

If the installer fails, inspect its final error and the service logs before continuing. A failed first activation can roll back the new `venv` path; fix the reported cause and rerun the full installer.

## 7. Verify playback and the management portal

Check both services:

```bash
systemctl is-active eposter-admin.service eposter-display.service
```

Expected output:

```text
active
active
```

Check the display's latest status:

```bash
cd "$HOME/eposter"

python3 -m json.tool .playback-status.json
```

Look for:

- `"ready": true`
- `"error": null`
- The intended `screen_number`
- A `display_size` matching the active display resolution

Also check the physical screen. An active service and a readiness flag do not, by themselves, verify that the image looks correct.

Find the board's network address:

```bash
hostname -I
```

From a computer or phone on the same network, open:

```text
http://<RADXA-IP>/
```

For example, if that board reports `192.168.29.232`, open:

```text
http://192.168.29.232/
```

Sign in using the username and password set in `config.json`. The portal listens on HTTP port 80.

Confirm the screen number, content API settings, rotation, and playback mode. Portal login credentials and `api.timezone` are configured directly in `config.json`; the portal does not provide editors for those settings in the reviewed version.

In `"Time"` mode, a screensaver can be normal when no poster is currently scheduled. An empty API URL also means there is no online poster feed to fetch.

## 8. Reboot and check the first mpv instance

Save any open work, then run:

```bash
sudo reboot
```

Confirm all of the following on each device:

- [ ] The device automatically reaches the intended desktop account.
- [ ] ePoster starts automatically.
- [ ] **The first mpv instance displays correctly without being manually closed.**
- [ ] The board shows content assigned to its screen number when scheduled.
- [ ] The management portal remains accessible.

If a board fails this check, leave the incorrect first instance running while collecting:

```bash
sudo systemctl status \
    eposter-display.service \
    eposter-admin.service \
    --no-pager -l

sudo journalctl -b -u eposter-display.service --no-pager -n 60

mpv --version
```

## What changes between devices?

| Item | Per-device action |
| --- | --- |
| `hardware_id` | Set the board's assigned identity. |
| `display.screen_number` | Set its assigned API screen number. |
| Display, Xauthority, UID | Captured automatically by Step 5 on that board. |
| Rotation and resolution | Match that monitor's mounting and intended output. |
| API endpoint, token, timezone | Use the settings appropriate to the deployment. |
| IP address | Read it from that board with `hostname -I`. |

Repeat the procedure on each new board, including Step 5. Do not copy another board's fixed display number, UID, or Xauthority path without detecting the new board's actual session.
(
set -e

cd "$HOME/eposter"

python3 - <<'PY'
import ast
from pathlib import Path

path = Path("helper/display_handler.py")
text = path.read_text()

old = "resized = image.resize(size, Image.Resampling.LANCZOS)"
new = (
    'resized = ImageOps.pad(image, size, '
    'method=Image.Resampling.LANCZOS, color="black", '
    'centering=(0.5, 0.5))'
)

if text.count(old) == 1:
    updated = text.replace(old, new, 1)
    ast.parse(updated)
    path.write_text(updated)
    print("Applied image fitting without stretching.")
elif "resized = ImageOps.pad(" in text:
    print("Image fitting fix is already present; restarting the display.")
else:
    raise SystemExit(
        "Expected image-resize code was not found. No changes were made."
    )
PY

sudo systemctl restart eposter-display.service
)
Record the software revision for each deployment:

```bash
cd "$HOME/eposter"
git rev-parse --short HEAD
```

If the optional image-fitting patch is applied locally, record that separately: the Git commit ID alone does not describe uncommitted changes.

## Source references

The procedure was checked against these project files and systemd documentation. The repository links below are pinned to the reviewed revision so that later changes to `main` do not silently change the reference material.

- [Installer](https://github.com/Bhavi-1266/eposter/blob/904eb818e586dc3531f8e0eac5a66c872c7cd9c6/installer.py)
- [Display service template](https://github.com/Bhavi-1266/eposter/blob/904eb818e586dc3531f8e0eac5a66c872c7cd9c6/service_files/eposter-display.service.template)
- [Configuration example](https://github.com/Bhavi-1266/eposter/blob/904eb818e586dc3531f8e0eac5a66c872c7cd9c6/config.example.json)
- [Schedule selection](https://github.com/Bhavi-1266/eposter/blob/904eb818e586dc3531f8e0eac5a66c872c7cd9c6/RunThis.py)
- [Image preparation](https://github.com/Bhavi-1266/eposter/blob/904eb818e586dc3531f8e0eac5a66c872c7cd9c6/helper/display_handler.py)
- [Controller and playback status](https://github.com/Bhavi-1266/eposter/blob/904eb818e586dc3531f8e0eac5a66c872c7cd9c6/helper/media_controller.py)
- [Portal and updater documentation](https://github.com/Bhavi-1266/eposter/blob/904eb818e586dc3531f8e0eac5a66c872c7cd9c6/docs/PORTAL.md)
- [systemd service configuration](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)
