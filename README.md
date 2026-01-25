# Cisco Packet Tracer 9.0 on openSUSE

This repository documents a manual installation of Cisco Packet Tracer 9.0
on openSUSE using the official Debian package distributed by Cisco.

Packet Tracer 9.x is shipped as an AppImage and installs primarily under
`/opt/pt`. While Cisco provides native packages for some Linux distributions,
openSUSE users typically need to handle the installation manually.

The steps below describe the process in a way that makes each part of the
installation understandable and reproducible.

---

## System Context

- Distribution: openSUSE (tested on KDE Plasma)
- Installer source: Cisco Networking Academy (`.deb`)
- Packet Tracer version: 9.0
- Desktop environment: KDE

SELinux was present on the system but running in permissive mode and did not
affect the installation.

---

## Debian Package Structure

A Debian package is an `ar` archive. Extracting it produces three components:

- `debian-binary`
- `control.tar.*` (metadata and maintainer scripts)
- `data.tar.*` (filesystem contents)

The application files themselves are contained in `data.tar.*`. To extract
the package manually, the `ar` tool is required.

### Required tools

```sh
sudo zypper in binutils tar xz
```

---

## Extracting the Packet Tracer Installer

```sh
mkdir ~/pt-extract
cd ~/pt-extract

ar x ~/Downloads/CiscoPacketTracer_900_Ubuntu_64bit.deb
ls
```

At this stage, the relevant archive is `data.tar.*`.

### Extract and install application files

```sh
mkdir data
tar -xf data.tar.* -C data

sudo cp -r data/opt /
```

For Packet Tracer 9.0, the installation consists almost entirely of the
`/opt/pt` directory.

---

## Command-Line Entry Point

A symbolic link is created to make the AppImage accessible from the command
line.

```sh
sudo ln -sf /opt/pt/packettracer.AppImage /usr/local/bin/packettracer
```

---

## Activation and User Configuration

Packet Tracer requires an activation step that writes state information into
the user’s home directory under `~/.local/.packettracer`. This step must be
run as the normal user.

### Prepare user directory

```sh
mkdir -p ~/.local/.packettracer
chmod -R u+rwX ~/.local/.packettracer
```

### Run activation

```sh
/usr/local/bin/packettracer --pt-activate
```

The command exits after printing a single identifying line. This indicates
successful registration and does not start the graphical interface.

---

## qtpaths Dependency

During activation and desktop integration, `xdg-mime` invokes `qtpaths`.
On openSUSE, this utility is not installed by default.

```sh
sudo zypper in libqt5-qtpaths
```

---

## KDE Login Issue

On KDE systems, Packet Tracer attempts to open the Cisco login page using
`kde-open`. Because Packet Tracer is packaged as an AppImage, its runtime
environment can interfere with KDE tools that depend on the system Qt
version.

This typically results in Qt version mismatch errors and prevents the browser
from opening, leaving the login process blocked.

---

## Browser Wrapper Approach

To handle this, a small wrapper script is used. The script temporarily
overrides `xdg-open` and `kde-open` for the lifetime of the Packet Tracer
process and forces URL handling through a known working browser (Brave),
while clearing Qt-related environment variables.

The wrapper operates only within the Packet Tracer process tree and does not
modify system-wide configuration.

---

## Wrapper Script (Brave)

### Create wrapper

```sh
sudo tee /opt/pt/pt-brave >/dev/null <<'EOF'
#!/bin/sh
set -eu

TMPDIR="$(mktemp -d)"
cleanup() { rm -rf "$TMPDIR"; }
trap cleanup EXIT INT TERM

cat > "$TMPDIR/xdg-open" <<'EOFX'
#!/bin/sh
unset LD_LIBRARY_PATH QT_PLUGIN_PATH QML2_IMPORT_PATH
unset QT_QPA_PLATFORM_PLUGIN_PATH XDG_DATA_DIRS
exec brave-browser "$@"
EOFX
chmod +x "$TMPDIR/xdg-open"

cat > "$TMPDIR/kde-open" <<'EOFX'
#!/bin/sh
unset LD_LIBRARY_PATH QT_PLUGIN_PATH QML2_IMPORT_PATH
unset QT_QPA_PLATFORM_PLUGIN_PATH XDG_DATA_DIRS
exec brave-browser "$@"
EOFX
chmod +x "$TMPDIR/kde-open"

exec env PATH="$TMPDIR:$PATH" /usr/local/bin/packettracer "$@"
EOF

sudo chmod +x /opt/pt/pt-brave
```

### Create command symlink

```sh
sudo ln -sf /opt/pt/pt-brave /usr/local/bin/packettracer-brave
```

### Run Packet Tracer

```sh
packettracer-brave
```

---

## Desktop Integration (Optional)

If a desktop entry is present, update its `Exec` line to:

```ini
Exec=/usr/local/bin/packettracer-brave
```

---

## Removal

```sh
sudo rm -rf /opt/pt
sudo rm -f /usr/local/bin/packettracer /usr/local/bin/packettracer-brave
rm -rf ~/.local/.packettracer
```

---

## Notes

Once authenticated, Packet Tracer stores its login state locally and does not
require repeating the browser-based login on subsequent launches.

The wrapper script can be adapted to other browsers if needed.
