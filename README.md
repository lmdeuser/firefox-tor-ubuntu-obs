# Firefox & Tor Browser -- Native Ubuntu 26.04 LTS Packages (OBS)

Debian `3.0 (quilt)` packaging specification, openSUSE Build Service (`OBS`)
remote distribution guide, and signed APT client configuration for **Mozilla
Firefox** (`160.0.1-1+obs1`), **Tor Browser** (`16.0.3-1+obs1`), and the unified
**`firefox-tor-suite`** (`1.0.0-1+obs1`) metapackage targeting **Ubuntu 26.04
LTS (`resolute` / `xUbuntu_26.04`, `amd64`)**.

> [!NOTE]
> This is an independent community packaging repository and is not affiliated
> with or endorsed by the Mozilla Foundation, the Tor Project, or Canonical Ltd.

## Table of Contents

- [Overview](#overview)
- [Why This Repository Exists](#why-this-repository-exists)
- [Packages and Version Matrix](#packages-and-version-matrix)
- [Firefox Hardware Acceleration and Codec Matrix](#firefox-hardware-acceleration-and-codec-matrix)
- [Tor Browser Isolation and Hardening](#tor-browser-isolation-and-hardening)
- [System Requirements](#system-requirements)
- [Installation](#installation)
- [Post-Installation Verification](#post-installation-verification)
- [Updating, Rollback, and Uninstallation](#updating-rollback-and-uninstallation)
- [Packaging Architecture](#packaging-architecture)
- [License](#license)

## Overview

Ubuntu 26.04 LTS replaces the native `firefox` `.deb` archive package with a
Snap-redirecting transitional stub and does not provide a native, APT-managed
Tor Browser package integrated with an Ubuntu 26.04 `abi <abi/5.0>` AppArmor
profile and bundled Pluggable Transports.

This repository documents how to install and pin signed `.deb` packages built
from pinned upstream sources on isolated OBS workers and published via
[`home:dwolfF:browsers`](https://build.opensuse.org/project/show/home:dwolfF:browsers):

- **`firefox`**: Native Mozilla Firefox compiled for Ubuntu 26.04 (`amd64`),
  with preconfigured WebRender, DMABUF, Wayland/X11-EGL display integration,
  VA-API and Vulkan hardware decode defaults, FFmpeg 8 (`libavcodec62`),
  OpenH264 (`libopenh264-8`), `dav1d` AV1, JPEG XL, HEVC, WebRTC H.264 HW, and
  `policies.json` delegating browser updates to APT.
- **`tor-browser`**: Native Tor Browser built from pinned upstream Gecko ESR,
  `tor` daemon, Go Pluggable Transports (`lyrebird`, `conjure`), `moat-settings`,
  bundled Noto font coverage, Tor Browser Manual, and NoScript, strictly
  preserving upstream anti-fingerprinting defaults behind an Ubuntu 26.04
  AppArmor policy.
- **`firefox-tor-suite`**: Lockstep metapackage keeping `firefox`,
  `tor-browser`, and shared graphics, codec, and portal recommendations aligned
  across upgrades.

## Why This Repository Exists

| Aspect | Ubuntu Default (`firefox` stub / manual Tor tarball) | Generic Upstream Mozilla APT | OBS Repository (`home:dwolfF:browsers`) |
| --- | --- | --- | --- |
| Delivery mechanism | Snap transitional stub or unpacked `$HOME` tarball | Cross-distro `.deb` without Ubuntu 26.04 FFmpeg 8 or Mesa `26.0.8` alignment | Signed Ubuntu `xUbuntu_26.04` native `.deb` packages |
| Ubuntu 26.04 library ABI | Confined Snap runtime or static archive | Generic defaults; manual codec and GPU driver setup | Linked and declared against `libavcodec62`, `libopenh264-8`, `mesa-libgallium`, `libjxl0.11`, `libdav1d7`, `libvpx12` |
| Firefox HW acceleration & codecs | Varies by confinement and user prefs | Default upstream prefs; legacy or unset codec knobs | Curated `obs-hwaccel-codecs.js` (18 verified Firefox 160 StaticPrefs) plus `/usr/bin/firefox` Wayland/EGL/NVIDIA wrapper |
| Tor Browser isolation | Manual tarball self-updater outside APT | Absent from Mozilla APT | Immutable `/usr/lib/tor-browser/`, `abi <abi/5.0>` AppArmor profile, per-user `.build-id` symlink sync, no self-updater |
| Update discipline | Background Snap refresh or browser auto-updater | Browser updates decoupled from distro ABI | `DisableAppUpdate` in `policies.json` + disabled Tor self-updater; deterministic APT upgrades |
| Build & release safety | Opaque upstream channel | Direct upload | Two-contour OBS workflow (`:staging` build + offline checks before stable promotion) |

## Packages and Version Matrix

| Package | Upstream Base | Package Version | Role |
| --- | --- | --- | --- |
| `firefox` | Mozilla Firefox `160.0.1` | `160.0.1-1+obs1` | Native browser with FFmpeg 8, OpenH264, VA-API/Vulkan/Wayland defaults, and APT update policy |
| `tor-browser` | Tor Browser `16.0.3` (`153.8.0esr`, `tor` `0.4.9.17`, `lyrebird` `0.10.0`, `conjure`, `noscript` `13.6.39.1984`) | `16.0.3-1+obs1` | Hardened Tor Browser bundle with AppArmor `abi/5.0` profile and zero telemetry/VA-API anti-fingerprinting leaks |
| `firefox-tor-suite` | Suite lockstep metapackage | `1.0.0-1+obs1` | Enforces synchronized minimum versions of `firefox` and `tor-browser` plus recommended GPU, codec, and portal packages |

> [!NOTE]
> Staging builds compile an updated `cbindgen` (`0.29.4-1+obs1`) privately
> inside `:staging` because Ubuntu 26.04 `universe` ships `0.29.2` while
> Gecko requires `>= 0.29.4`. It is a build-time dependency only and is never
> published to the stable APT repository.

## Firefox Hardware Acceleration and Codec Matrix

The `firefox` package installs
`/usr/lib/firefox/defaults/pref/obs-hwaccel-codecs.js` containing 18 verified
Firefox 160 `StaticPrefs` (the deprecated `media.ffmpeg.vaapi.enabled` toggle
removed upstream is intentionally omitted):

| Subsystem | Enabled Preferences / Mechanism | Runtime Dependencies / Driver Support |
| --- | --- | --- |
| IPC & Compositor | `dom.ipc.forkserver.enable`, `gfx.webrender.all`, `gfx.canvas.accelerated`, `dom.webgpu.enabled` | `libdrm2`, `libgbm1`, `libegl1`, `libgl1`, `libgles2`, `libvulkan1`, `libpci3`, `mesa-vulkan-drivers` |
| Wayland & DMABUF | `widget.dmabuf.enabled`, `widget.wayland.fractional-scale.enabled`, `widget.wayland.vsync.enabled`; launcher exports `MOZ_ENABLE_WAYLAND=1` on Wayland or `MOZ_X11_EGL=1` on X11 | `pipewire`, `xdg-desktop-portal`, `xdg-desktop-portal-gtk` / `kde` / `gnome` |
| FFmpeg & H.264 / HEVC | `media.ffmpeg.enabled`, `media.ffmpeg.encoder.enabled`, `media.ffmpeg.allow-openh264`, `media.webrtc.hw.h264.enabled`, `media.hevc.enabled` | `libavcodec-extra62` / `libavcodec62`, `libopenh264-8` |
| Hardware Video Decode | `media.hardware-video-decoding.enabled`, `media.hardware-video-decoding-vulkan.enabled` | `libva2`, `libva-drm2`, `libva-wayland2`, `libva-x11-2`, `mesa-libgallium`, `intel-media-va-driver-non-free` / `intel-media-va-driver`, `i965-va-driver`, `nvidia-vaapi-driver`, `va-driver-all` |
| Modern Royalty-Free Media | `media.av1.enabled`, `media.av1.use-dav1d`, `image.jxl.enabled`, `image.avif.sequence.enabled` | `libdav1d7`, `libaom3`, `libvpx12`, `libjxl0.11`, `libwebp7`, `libopus0`, `libvorbis0a` |
| NVIDIA VA-API Compatibility | Launcher detects `/proc/driver/nvidia/version` + `nvidia_drv_video.so` and sets `LIBVA_DRIVER_NAME=nvidia` and `MOZ_DISABLE_RDD_SANDBOX=1` (overridable via env) | `nvidia-vaapi-driver` (never weakens content sandboxing with `--no-sandbox`) |

On Ubuntu 26.04, Mesa `26.0.8` folds `radeonsi_drv_video.so`,
`nouveau_drv_video.so`, and `virtio_gpu_drv_video.so` directly into
`mesa-libgallium` (`mesa-va-drivers` does not exist on Ubuntu 26.04). The
package dependencies declare that split explicitly.

## Tor Browser Isolation and Hardening

Tor Browser packaging prioritizes fingerprint uniformity and host isolation
over desktop codec or GPU tuning:

- **Strict upstream anti-fingerprinting defaults:** No VA-API decode, no WebGPU
  override, no custom hardware codec prefs, and no system font overrides are
  injected. Bundled Noto, CJK, Emoji, and UI fonts remain self-contained inside
  `/usr/lib/tor-browser/`.
- **Immutable system bundle and writable user state:** Program files stay
  read-only under `/usr/lib/tor-browser/`, while `/usr/bin/tor-browser` manages
  writable state under
  `${XDG_DATA_HOME:-$HOME/.local/share}/tor-browser/app/TorBrowser/Data/{Browser,Tor}`
  (`0700`) and atomically reconciles symlinks whenever `.build-id` changes after
  an APT upgrade.
- **Ubuntu 26.04 AppArmor enforcement:** Installs `/etc/apparmor.d/tor-browser`
  targeting `abi <abi/5.0>`.
- **No in-app self-updater races:** Compiled with `--disable-updater` and
  `--disable-base-browser-update` so updates are delivered cleanly through APT.

## System Requirements

| Requirement | Supported Target |
| --- | --- |
| Operating System | Ubuntu 26.04 LTS (`resolute` / `xUbuntu_26.04`) and official flavours |
| Architecture | `amd64` (`x86_64`) |
| Ubuntu Components | `main` and `universe` enabled in `/etc/apt/sources.list.d/ubuntu.sources` |
| Privileges | `sudo` access for configuring `/etc/apt/`, AppArmor, and packages |
| Optional GPU Drivers | `mesa-libgallium`, `mesa-vulkan-drivers`, `intel-media-va-driver` / `intel-media-va-driver-non-free`, `i965-va-driver`, or `nvidia-vaapi-driver` |

## Installation

1. **Install prerequisites and import the OBS repository signing key:**

   ```bash
   sudo apt-get update
   sudo apt-get install -y ca-certificates curl gpg
   sudo install -d -m 0755 /etc/apt/keyrings
   curl -fsSL https://download.opensuse.org/repositories/home:/dwolfF:/browsers/xUbuntu_26.04/Release.key \
     | sudo gpg --dearmor --yes -o /etc/apt/keyrings/home_dwolfF_browsers.gpg
   sudo chmod 0644 /etc/apt/keyrings/home_dwolfF_browsers.gpg
   ```

2. **Verify the signing key fingerprint before trusting packages:**

   ```bash
   gpg --show-keys --with-fingerprint /etc/apt/keyrings/home_dwolfF_browsers.gpg
   ```

   Expected fingerprint (`home:dwolfF OBS Project <home:dwolfF@build.opensuse.org>`):

   ```text
   C106 E46D E757 8257 F4CD  62C9 6AE3 5907 289D 3BDC
   ```

3. **Configure the DEB822 APT source:**

   ```bash
   sudo tee /etc/apt/sources.list.d/home_dwolfF_browsers.sources >/dev/null <<'EOF'
   Types: deb
   URIs: https://download.opensuse.org/repositories/home:/dwolfF:/browsers/xUbuntu_26.04/
   Suites: /
   Architectures: amd64
   Signed-By: /etc/apt/keyrings/home_dwolfF_browsers.gpg
   EOF
   ```

4. **Apply strict APT pinning rules:**

   ```bash
   sudo tee /etc/apt/preferences.d/firefox-tor-ubuntu-obs >/dev/null <<'EOF'
   Explanation: Prefer native OBS Firefox, Tor Browser, and suite packages
   Package: firefox firefox-l10n-* tor-browser firefox-tor-suite
   Pin: origin download.opensuse.org
   Pin-Priority: 1002

   Explanation: Block Ubuntu transitional Snap Firefox stubs and locale wrappers
   Package: firefox firefox-*
   Pin: release o=Ubuntu
   Pin-Priority: -1

   Explanation: Supersede third-party Mozilla APT repository when present
   Package: firefox firefox-l10n-*
   Pin: origin packages.mozilla.org
   Pin-Priority: -1
   EOF
   ```

   > [!IMPORTANT]
   > `Pin-Priority: 1002` allows clean cross-repository migration from higher
   > upstream version numbers (`161.0~build1`), while `-1` prevents Ubuntu Snap
   > stubs and `packages.mozilla.org` overrides from competing with OBS builds.

5. **Retire conflicting Firefox sources (optional):**

   If Ubuntu's Snap Firefox or the third-party `packages.mozilla.org` APT source
   is present on the host, you can clean them up now:

   ```bash
   if command -v snap >/dev/null 2>&1 && snap list firefox >/dev/null 2>&1; then
     sudo snap remove --purge firefox
   fi
   sudo rm -f /etc/apt/sources.list.d/mozilla.list /etc/apt/sources.list.d/mozilla.sources
   ```

   > [!NOTE]
   > If `firefox-l10n-*` from `packages.mozilla.org` was previously installed,
   > APT removes that stub during migration. Additional Firefox UI languages
   > can be installed directly inside Firefox via **Settings -> General ->
   > Language**.

6. **Refresh APT metadata, install `firefox-tor-suite`, and load the Tor Browser AppArmor profile:**

   ```bash
   sudo apt-get update
   sudo apt-get install --allow-downgrades -y firefox-tor-suite
   sudo apparmor_parser -r /etc/apparmor.d/tor-browser
   ```

   If you only want one browser instead of `firefox-tor-suite`, install it directly:

   ```bash
   # Firefox only:
   # sudo apt-get install --allow-downgrades -y firefox

   # Tor Browser only:
   # sudo apt-get install -y tor-browser
   # sudo apparmor_parser -r /etc/apparmor.d/tor-browser
   ```

## Post-Installation Verification

Run these checks after installing the packages:

1. **Confirm APT pinning and OBS package origins (`1002` priority vs `-1` blocked):**

   ```bash
   apt-cache policy firefox tor-browser firefox-tor-suite
   ```

2. **Verify Firefox policies, codec prefs, VA-API, and Vulkan support:**

   ```bash
   test -f /usr/lib/firefox/defaults/pref/obs-hwaccel-codecs.js
   test -f /usr/lib/firefox/distribution/policies.json
   vainfo
   vulkaninfo --summary
   ```

   Inside Firefox, open `about:support` (check **Window Protocol**, **WebRender**
   compositing, and hardware video decode / codec tables) and `about:policies`
   (confirm `DisableAppUpdate` is active).

3. **Verify Tor Browser AppArmor enforcement and isolated user state:**

   ```bash
   sudo aa-status | grep tor-browser
   ```

   After launching `tor-browser` once, verify isolated `0700` state creation:

   ```bash
   ls -ld "${XDG_DATA_HOME:-$HOME/.local/share}/tor-browser/app/TorBrowser/Data"
   ```

## Updating, Rollback, and Uninstallation

### Updating

Both browsers disable internal self-updaters (`policies.json` `DisableAppUpdate`
and Tor's `--disable-updater`) and receive updates strictly through APT:

```bash
sudo apt-get update
sudo apt-get upgrade
```

### Rollback

If an earlier `+obs` build is present in the repository or local APT cache,
list available versions and pin/install a specific release:

```bash
apt-cache madison firefox tor-browser firefox-tor-suite
sudo apt-get install --allow-downgrades -y firefox=<previous-version>
```

### Full Uninstallation

Remove the packages, repository source, pinning rules, and signing key:

```bash
sudo apt-get purge -y firefox-tor-suite firefox tor-browser
sudo apt-get autoremove -y
sudo rm -f /etc/apt/sources.list.d/home_dwolfF_browsers.sources \
           /etc/apt/preferences.d/firefox-tor-ubuntu-obs \
           /etc/apt/keyrings/home_dwolfF_browsers.gpg
sudo apt-get update
```

User profiles under `~/.mozilla/firefox/` and
`${XDG_DATA_HOME:-$HOME/.local/share}/tor-browser/` remain untouched unless
explicitly removed.

## Packaging Architecture

Packages follow a two-contour OBS workflow:

1. **Staging ([`home:dwolfF:browsers:staging`](https://build.opensuse.org/project/show/home:dwolfF:browsers:staging)):**
   Unpublished `xUbuntu_26.04` (`x86_64`) build environment resolving
   dependencies in strict order (`Ubuntu:26.04/update` -> `universe-update` ->
   `standard` -> `universe`) with Ubuntu 26.04 `rustc-1.97`, `cargo-1.97`,
   `clang-22`, `lld-22`, `llvm-22`, bundled `NSS 3.131`, and system `zlib`.
   Compiler cache wrappers are strictly prohibited (`Build-Conflicts: ccache, sccache`, `--without-ccache`). Built `.deb` packages undergo offline
   container validation (`--network none` on `ubuntu:26.04`) before promotion.
2. **Release & Promotion ([`home:dwolfF:browsers`](https://build.opensuse.org/project/show/home:dwolfF:browsers)):**
   Direct building is disabled; verified packages enter only through controlled
   OBS release with `PublishFlags: nofailedpackages`. `firefox-tor-suite` is
   promoted only when compatible `firefox` and `tor-browser` builds are both
   published in stable.

Published OBS links:

- [OBS Project: `home:dwolfF:browsers`](https://build.opensuse.org/project/show/home:dwolfF:browsers)
- [APT Repository Index (`xUbuntu_26.04`)](https://download.opensuse.org/repositories/home:/dwolfF:/browsers/xUbuntu_26.04/)
- [OBS Package: `firefox`](https://build.opensuse.org/package/show/home:dwolfF:browsers/firefox)
- [OBS Package: `tor-browser`](https://build.opensuse.org/package/show/home:dwolfF:browsers/tor-browser)
- [OBS Package: `firefox-tor-suite`](https://build.opensuse.org/package/show/home:dwolfF:browsers/firefox-tor-suite)
- [OBS Staging Project: `home:dwolfF:browsers:staging`](https://build.opensuse.org/project/show/home:dwolfF:browsers:staging)

## License

Packaging metadata, documentation, and launcher/configuration assets in this
repository are licensed under the **Mozilla Public License 2.0 (MPL-2.0)**
([LICENSE](LICENSE)). Upstream Mozilla Firefox and Tor Browser source code
remains subject to their respective upstream licenses.
