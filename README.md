# Cisco Packet Tracer 9.0 and 8.X on openSUSE

This repository documents a manual installation of Cisco Packet Tracer 9.0
and 8.2.2 on openSUSE using the official Debian package distributed by Cisco.

Packet Tracer 9.x is shipped as an AppImage and installs primarily under
`/opt/pt`. While Cisco provides native packages for some Linux distributions,
openSUSE users typically need to handle the installation manually.

Packet Tracer 8.x is shipped as a traditional binary package and also installs
under `/opt/pt`.

If both versions are required, the 8.2.2 directory must be renamed (for example
to `/opt/pt-822`) and the symbolic links must be adjusted accordingly.

The steps below describe the process in a way that makes each part of the
installation understandable and reproducible.

## System Context

- Distribution: openSUSE (tested on KDE Plasma, Tumbleweed)
- Installer source: Cisco Networking Academy (`.deb`)
- Packet Tracer versions: 9.0 and 8.2.2

SELinux was present on the system but running in permissive mode and did not
affect the installation.

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
````

## Extracting the Packet Tracer Installer

```sh
mkdir ~/pt-extract
cd ~/pt-extract

ar x ~/Downloads/CiscoPacketTracer_900_Ubuntu_64bit.deb

## or:

ar x ~/Downloads/CiscoPacketTracer822_amd64_signed.deb
ls
```

At this stage, the relevant archive is `data.tar.*`.

### Extract and install application files

```sh
mkdir data
tar -xf data.tar.* -C data

sudo cp -r data/opt /
```

For Packet Tracer 9.0 and 8.X, the installation consists almost entirely of the
`/opt/pt` directory.

## Installing Both Versions

If installing both versions:

1.  Extract 8.2.2 and rename it:
```sh
sudo mv /opt/pt /opt/pt-822
```
2. Then install 9.0 normally (remains in `/opt/pt`).

Final layout:

```
/opt/pt        → Packet Tracer 9.0
/opt/pt-822    → Packet Tracer 8.2.2
```

## Tumbleweed Compatibility (8.2.2 Only)

Packet Tracer 8.2.2 requires `libxml2.so.2`, which is not provided in recent
Tumbleweed snapshots.

Install a compatible legacy package:

```sh
sudo zypper ar -f \
home_alvistack
https://download.opensuse.org/repositories/home:/alvistack/openSUSE_Tumbleweed/ \

sudo zypper refresh
sudo zypper in libxml2-2
sudo zypper al libxml2-2
```

This installs the required library without replacing the system `libxml2-16`.

## Command-Line Entry Point

For 9.X versions:

Create a symbolic link to make the AppImage accessible from the command line:

```sh
sudo ln -sf /opt/pt/packettracer.AppImage /usr/local/bin/packettracer-900
```

Create a wrapper to force a light theme on KDE and handle Cisco login via Brave
(Flatpak), while isolating the Packet Tracer configuration:

```sh
sudo tee /opt/pt/pt-brave >/dev/null <<'EOF'
#!/bin/sh
set -eu

export XDG_CONFIG_HOME="$HOME/.pt_isolated_config"
export LC_ALL=en_US.UTF-8
export LANG=en_US.UTF-8

TMPDIR="$(mktemp -d)"
cleanup() { rm -rf "$TMPDIR"; }
trap cleanup EXIT INT TERM

cat > "$TMPDIR/xdg-open" <<'EOFX'
#!/bin/sh
unset LD_LIBRARY_PATH QT_PLUGIN_PATH QML2_IMPORT_PATH
unset QT_QPA_PLATFORM_PLUGIN_PATH XDG_DATA_DIRS
exec flatpak run com.brave.Browser "$@"
EOFX
chmod +x "$TMPDIR/xdg-open"

cat > "$TMPDIR/kde-open" <<'EOFX'
#!/bin/sh
unset LD_LIBRARY_PATH QT_PLUGIN_PATH QML2_IMPORT_PATH
unset QT_QPA_PLATFORM_PLUGIN_PATH XDG_DATA_DIRS
exec flatpak run com.brave.Browser "$@"
EOFX
chmod +x "$TMPDIR/kde-open"

exec env PATH="$TMPDIR:$PATH" /usr/local/bin/packettracer-900 "$@"
EOF

sudo chmod +x /opt/pt/pt-brave
```

Create command symlinks:

```sh
sudo ln -sf /opt/pt/pt-brave /usr/local/bin/packettracer-brave
```

For 8.X versions:

Edit the original with a simple launcher:

```sh
sudo tee /opt/pt-822/packettracer >/dev/null <<'EOF'
#!/bin/bash
echo "Starting Packet Tracer 8.2.2"
export LD_LIBRARY_PATH=/opt/pt-822/bin
cd /opt/pt-822/bin || exit 1
./PacketTracer "$@"
EOF

sudo chmod +x /opt/pt-822/packettracer
```

Create a wrapper (light theme / isolated config) for 8.2.2:

```sh
sudo tee /usr/local/bin/packettracer-822-light >/dev/null <<'EOF'
#!/bin/sh
set -eu

export XDG_CONFIG_HOME="$HOME/.pt82_isolated_config"
export LC_ALL=en_US.UTF-8
export LANG=en_US.UTF-8

exec /opt/pt-822/packettracer "$@"
EOF

sudo chmod +x /usr/local/bin/packettracer-822-light
```

Usage:

```sh
packettracer              # Latest version (9.0, original)
packettracer-brave        # Latest version (9.0, Light Theme & Brave Wrapper)
packettracer-822-light    # Legacy version (8.2.2)
```

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
packettracer-brave --pt-activate
packettracer-822-light --pt-activate
```

The command exits after printing a single identifying line. This indicates
successful registration and does not start the graphical interface.

## qtpaths Dependency

During activation and desktop integration, `xdg-mime` invokes `qtpaths`.
On openSUSE, this utility is not installed by default.

```sh
sudo zypper in libqt5-qtpaths
```

## KDE Login Issue

On KDE systems, Packet Tracer attempts to open the Cisco login page using
`kde-open`. Because Packet Tracer is packaged as an AppImage, its runtime
environment can interfere with KDE tools that depend on the system Qt
version.

This typically prevents the browser from opening, leaving the login process
blocked.

## Browser Wrapper Approach

To handle this, a small wrapper script is used. The script temporarily
overrides `xdg-open` and `kde-open` for the lifetime of the Packet Tracer
process and forces URL handling through a known working browser (Brave Flatpak),
while clearing Qt-related environment variables.

The wrapper operates only within the Packet Tracer process tree and does not
modify system-wide configuration.

## Desktop Integration (Optional)

Create separate desktop entries for both versions.

### Packet Tracer 9.0

```sh
sudo tee /usr/share/applications/packettracer-900.desktop >/dev/null <<'EOF'
[Desktop Entry]
Name=Cisco Packet Tracer 9.0
Comment=Network Simulation Tool
Exec=/usr/local/bin/packettracer-brave
Icon=/opt/pt/art/app.png
Terminal=false
Type=Application
Categories=Education;Network;
EOF
```

### Packet Tracer 8.2.2

```sh
sudo tee /usr/share/applications/packettracer-822.desktop >/dev/null <<'EOF'
[Desktop Entry]
Name=Cisco Packet Tracer 8.2.2
Comment=Network Simulation Tool (Legacy)
Exec=/usr/local/bin/packettracer-822-light
Icon=/opt/pt-822/art/app.png
Terminal=false
Type=Application
Categories=Education;Network;
EOF
```

If entries do not appear immediately, log out and back in.

## Removal (Uninstall)

Remove launchers and desktop entries:

```sh
sudo rm -f /usr/local/bin/packettracer /usr/local/bin/packettracer-900
sudo rm -f /usr/local/bin/packettracer-brave /usr/local/bin/packettracer-822-light
sudo rm -f /usr/share/applications/packettracer-900.desktop /usr/share/applications/packettracer-822.desktop
rm -rf ~/.local/.packettracer ~/.pt_isolated_config ~/.pt82_isolated_config
sudo rm -rf /opt/pt /opt/pt-822
```

## Notes

* 9.0 loads slower due to the AppImage runtime.
* 8.2.2 loads faster but requires the legacy `libxml2-2` package.
* Qt WebEngine warnings on Tumbleweed are harmless.
* Both versions can coexist safely when installed in separate directories.
* `packettracer` points to the latest version (9.0).
* The wrapper script can be adapted to other browsers if needed.
* Once authenticated, Packet Tracer per version stores its login state locally and does not require repeating the browser-based login on subsequent launches.
