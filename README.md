# Citrix Workspace App on Linux via Distrobox

This repository contains scripts and documentation for running Citrix
Workspace App on Linux using Distrobox containers.

## Why This Setup Exists

### The Problem
Citrix Workspace App on Linux has several issues:

1. **Distribution Compatibility**: Citrix officially supports only
   specific Linux distributions and versions of said
   distributions.
2. **Dependency Conflicts**: Installing Citrix directly on the host
   system can cause conflicts with existing packages.


### The Solution: Distrobox
**Distrobox** provides containerized environments that integrate
seamlessly with the host system:

- **Isolation**: Citrix runs in a dedicated container without
  affecting the host system
- **Compatibility**: Run any distribution regardless of your host
  distro
- **GUI Integration**: Applications can access the host's display
  server (X11/Wayland)
- **Home Directory Access**: The container has access to your home
  directory for configuration and saved connections
- **Easy Cleanup**: Remove the container if you no longer need Citrix


## Setup Overview

The setup creates a Distrobox container with:
- **Debian 12** base image
- **Citrix Workspace App 26.01.0.150**
- **All required dependencies** (X11, GTK, audio, smart card support)
- **Desktop integration** via `.desktop` file


## Requirements

### Host System

- Linux distribution with Distrobox support
- Distrobox installed


### Citrix Packages

You need to manually download the Citrix `.deb` packages:
1. Visit https://www.citrix.com/downloads/workspace-app/linux/workspace-app-for-linux-latest.html
2. Download the **Debian/Ubuntu 64-bit x86_64** packages
3. Save both files to the current directory:
   - `icaclient_26.01.0.150_amd64.deb`
   - `ctxusb_26.01.0.150_amd64.deb`


## Installation

```bash
git clone https://github.com/trapexit/distrobox-citrix.git

cd distrobox-citrix

./setup-distrobox-citrix
```

### What the Script Does

The `setup-distrobox-citrix` script performs 7 steps:

1. **Validates Packages**: Checks that both `.deb` files exist and opens the download page if missing
2. **Creates Container**: Creates a Distrobox container named `citrix-26.01.0.150` based on Debian 12
3. **Installs Dependencies**: Installs all required system libraries (GTK, audio, smart card, X11 support)
4. **Installs Citrix**: Installs the Citrix Workspace App packages
5. **Wraps wfica**: Creates a wrapper script that sources environment variables from `~/.ICAClient/wfica.env`
6. **Accepts EULA**: Creates the EULA acceptance file and generates the `citrix-workspace` launch script
7. **Creates Desktop File**: Extracts the Citrix icon and installs a `.desktop` file for application menu integration


## Usage

### Launch Methods

**1. Application Menu (Recommended)** After installation, look for
"Citrix Workspace App 26.01.0.150" in your applications menu.

**2. Command Line Script**
```bash
./citrix-workspace
```

**3. Direct Command**
```bash
distrobox-enter citrix-26.01.0.150 -- /opt/Citrix/ICAClient/selfservice
```

### First Launch
1. The SelfService GUI will open
2. Enter your store front URL
3. Log in with your credentials
4. Browse and launch available applications


## Files Created

### Scripts
- `setup-distrobox-citrix` - Main setup script
- `uninstall-distrobox-citrix` - Uninstall everything installed by
  setup script
- `citrix-workspace` - Launch script for Citrix SelfService


### Desktop File
- `~/.local/share/applications/citrix-26.01.0.150.desktop` - Menu integration

### Container
- `citrix-26.01.0.150` - Distrobox container with Citrix installed


## Configuration Variables

The script uses variables at the top for easy updates:

```bash
CITRIX_VERSION="26.01.0.150"
ICACLIENT_PKG="icaclient_${CITRIX_VERSION}_amd64.deb"
CTXUSB_PKG="ctxusb_${CITRIX_VERSION}_amd64.deb"
CONTAINER_NAME="citrix-${CITRIX_VERSION}"
```

To update to a new Citrix version:
1. Update `CITRIX_VERSION`
2. Download new `.deb` packages
3. Re-run the setup script


### Environment Variables for wfica Sessions

The setup script configures `wfica` to source environment variables from
`~/.ICAClient/wfica.env` before launching the actual Citrix session. This
allows you to set environment variables that affect the wfica process without
modifying the system.

**Location:** `~/.ICAClient/wfica.env`

**Example wfica.env:**
```bash
# Set timezone for the Citrix session
export TZ=America/New_York
```

**Notes:**
- The file is optional - if it doesn't exist, wfica runs normally
- Variables are only loaded by wfica, not by selfservice or other Citrix binaries
- Changes take effect on the next wfica session launch


## Maintenance

### Updating Citrix
1. Download new `.deb` packages
2. Update `CITRIX_VERSION` in the script
3. Remove old container: `distrobox rm citrix-26.01.0.150`
4. Re-run setup script


### Removing Everything

```bash
./uninstall-distrobox-citrix
```

## Troubleshooting

When in doubt... start from scratch:

```
./uninstall-distrobox-citrix
rm -rf ~/.ICAClient/
```

## Additional Resources

- [Distrobox Documentation](https://github.com/89luca89/distrobox)
- [Citrix Workspace App for Linux](https://www.citrix.com/downloads/workspace-app/linux/)
- [Citrix Support Articles](https://support.citrix.com/)


## Remaining issues

* Accelerated videos cause green boxes (which appear to be corrupted
  segments of video memory) to cover the section of the screen where
  the video is rendering. Neither `LIBGL_DRI3_ENABLE=0` or
  `GDK_BACKEND=x11` helped resolve it. Neither does disabling GPU
  rendering in the client browser.
* The username isn't stored between runs like you find on Windows.
