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
- **Citrix Workspace App 26.04.10.1** (Citrix page label `2604.10`)
- **All required dependencies** (X11, GTK, audio, smart card support)
- **Desktop integration** via `.desktop` file


## Requirements

### Host System

- Linux distribution with Distrobox support
- Distrobox installed
- Either `curl` or `wget` so setup can resolve and download the latest packages


### Citrix Packages

Setup automatically resolves Citrix's latest Linux download page and downloads
missing Debian x86_64 packages. The current resolved filenames are:

- `icaclient-gcc-8_26.04.10.1_amd64.deb`
- `ctxusb-gcc-8_26.04.10.1_amd64.deb`

If neither `curl` nor `wget` is installed, both files must already be present in
the current directory.


## Installation

```bash
git clone https://github.com/trapexit/distrobox-citrix.git

cd distrobox-citrix

./setup-distrobox-citrix
```

### What the Script Does

The `setup-distrobox-citrix` script performs 7 steps:

1. **Resolves Packages**: Reads Citrix's latest Linux page, downloads missing Debian x86_64 packages, and verifies SHA-256 checksums when Citrix provides them
2. **Creates Container**: Creates a Distrobox container named `citrix-26.04.10.1` based on Debian 12
3. **Installs Dependencies**: Installs all required system libraries (GTK, audio, smart card, X11 support)
4. **Installs Citrix**: Installs the Citrix Workspace App packages and disables the client's AAD SSO support, which breaks SAML gateway logons (see Troubleshooting)
5. **Wraps wfica**: Creates a wrapper script that sources environment variables from `~/.ICAClient/wfica.env` and keeps the container's Mesa out of the host's shader cache
6. **Accepts EULA**: Creates the EULA acceptance file and generates the `citrix-workspace` launch script
7. **Creates Desktop File**: Extracts the Citrix icon and installs a `.desktop` file for application menu integration


## Usage

### Launch Methods

**1. Application Menu (Recommended)** After installation, look for
"Citrix Workspace App 26.04.10.1" in your applications menu.

**2. Command Line Script**
```bash
./citrix-workspace
```

**3. Direct Command**
```bash
distrobox-enter citrix-26.04.10.1 -- /opt/Citrix/ICAClient/selfservice
```

### First Launch
1. The SelfService GUI will open
2. Enter your store front URL
3. Log in with your credentials
4. Browse and launch available applications


## Files Created

### Scripts and State
- `setup-distrobox-citrix` - Main setup script
- `uninstall-distrobox-citrix` - Uninstall everything installed by
  setup script
- `citrix-workspace` - Launch script for Citrix SelfService
- `.distrobox-citrix-version` - Installed version used by uninstall


### Desktop File
- `citrix-26.04.10.1.desktop` - Local generated desktop file
- `~/.local/share/applications/citrix-26.04.10.1.desktop` - Menu integration

### Container
- `citrix-26.04.10.1` - Distrobox container with Citrix installed


## Package Resolution

The script fetches Citrix's latest Linux download page at runtime and selects
the exact Debian x86_64 full and USB support package entries. It downloads
missing packages through `curl` or `wget` and verifies checksums published on
the page.

If the page cannot be fetched or parsed, setup falls back to the pinned
`26.04.10.1` filenames and requires those files to be available locally.


## Environment Variables for wfica Sessions

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
- The wrapper also exports `MESA_SHADER_CACHE_DISABLE=true` and
  `MESA_GLSL_CACHE_DISABLE=true` *before* sourcing this file, so it can
  override them. They stop the container's Mesa from writing the shared
  `~/.cache/mesa_shader_cache` of the host; see "EULA was rejected." below
- The launch script and the `.desktop` file export the same two variables for
  the whole `selfservice` process tree


## Maintenance

### Updating Citrix

Re-run setup:

```bash
./setup-distrobox-citrix
```

The script fetches Citrix's latest Linux page and downloads newer Debian x86_64
packages when Citrix publishes them. If the page cannot be reached, it falls
back to the pinned `26.04.10.1` filenames and requires those files locally.


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

### "EULA was rejected." and nothing opens

Symptom: the launch prints `EULA was rejected.` and no SelfService window
appears. `~/.ICAClient/.eula_accepted` exists and re-creating it changes
nothing, which is expected - the message is not about the EULA.

On startup `selfservice` probes the client with

```
$ICAROOT/wfica -eula -tell MinimumTLS,MaximumTLS,SSLCiphers
```

and prints `EULA was rejected.` whenever that helper exits non-zero. It never
looks at the helper's output, so any reason for `wfica` to exit 1 is reported
as a rejected EULA.

On a Wayland host the usual reason is that the X server died while `wfica`
was connecting:

```
journalctl --user | grep -E 'Caught signal|X11 connection broke'
```

```
kwin_wayland_wrapper[...]: (EE) Caught signal 7 (Bus error). Server aborting
kwin_wayland[...]: The X11 connection broke (error 1)
```

`SIGBUS` in Xwayland here is a stale mapping of
`~/.cache/mesa_shader_cache/index`. The container shares `$HOME` with the
host but ships a different Mesa (Debian 12 has 22.3.6) and the cache index
size is compiled into each Mesa build. When the container's Mesa opens the
shared index it truncates it to its own, smaller size, so the running
Xwayland's mapping extends past the end of the file and its next write into
that region raises `SIGBUS`. Xwayland dies, `wfica`'s connection is hung up,
and `selfservice` turns the exit code into the misleading EULA message.

To see which Mesa owns the index and whether anything is still writing it:

```
stat -c %s ~/.cache/mesa_shader_cache/index   # must match the host's Mesa
pgrep -x Xwayland                             # empty = it crashed
```

`setup-distrobox-citrix` prevents this: `citrix-workspace`, the `.desktop`
entry and the `wfica` wrapper all export `MESA_SHADER_CACHE_DISABLE=true` and
`MESA_GLSL_CACHE_DISABLE=true`, so container Mesa never writes the host's
index. That only disables shader caching for Citrix's own processes; the
host's own applications keep theirs.

If you launched Citrix in a way that bypasses all three - for example with
`distrobox-enter citrix-26.04.10.1 -- /opt/Citrix/ICAClient/selfservice` - add
the variables yourself, or give the container its own cache directory with
`MESA_SHADER_CACHE_DIR=/tmp/mesa_shader_cache` (Mesa appends
`mesa_shader_cache` to that path).

### "Your account cannot be added... Http error"

Symptom: SelfService opens, you log in at the identity provider, and then the
client reports `Your account cannot be added using this server address` /
`Http error`. The store itself is fine - a browser and the Windows/macOS
Workspace App log in to the same URL.

`~/.ICAClient/logs/PrimaryAuthManager-*.log` shows what really happened:

```
POST https://<store>/nf/auth/webview/done   -> HTTP 500
CProtocolException: HTTP response status 500
```

The gateway returns a bare `Http/1.1 Internal Server Error <code>` - no
detail, no hint about the credentials. The cause is the client's AAD SSO
support: it stages SSO cookies through the long-lived `AuthManagerDaemon`
before the gateway webview opens, which corrupts the SAML post-back. The
setup script therefore sets `AADSSOEnabled` to `false` in
`/opt/Citrix/ICAClient/config/AuthManConfig.xml`.

Check that it is still off:

```bash
distrobox-enter citrix-26.04.10.1 -- \
  grep -A1 AADSSOEnabled /opt/Citrix/ICAClient/config/AuthManConfig.xml
```

If logons start failing again after a client upgrade, set it back to `false`
and restart the client. If the symptom persists, clear the daemon's stale
cookie state once:

```bash
distrobox-enter citrix-26.04.10.1 -- bash -c \
  'pkill -x selfservice; pkill -f AuthManagerDaemon; rm -rf ~/.ICAClient/.tmp'
```

Citrix's own logs strip URLs and are not very verbose. To see full request
URLs and auth-flow details, switch the client to verbose logging, and switch
it back afterwards - verbose logs record auth tokens:

```bash
distrobox-enter citrix-26.04.10.1 -- bash -c \
  'sudo sed -i "/<key>LoggingMode<\/key>/{n;s|normal|verbose|}" \
     /opt/Citrix/ICAClient/config/AuthManConfig.xml'
```

```bash
distrobox-enter citrix-26.04.10.1 -- bash -c \
  'sudo sed -i "/<key>LoggingMode<\/key>/{n;s|verbose|normal|}" \
     /opt/Citrix/ICAClient/config/AuthManConfig.xml'
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
