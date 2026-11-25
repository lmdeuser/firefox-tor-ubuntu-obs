# Firefox & Tor Browser OBS APT Suite for Ubuntu 26.04 LTS

> [!NOTE]
> Native Debian (`3.0 (quilt)`) packaging, signed APT repository configuration, and hardware-accelerated desktop integration for **Mozilla Firefox** and **Tor Browser** on **Ubuntu 26.04 LTS (`resolute` / `xUbuntu_26.04`, `amd64`)**, built on the [openSUSE Build Service (OBS)](https://build.opensuse.org/project/show/home:dwolfF:browsers).

This repository publishes the installation guide, APT/DEB822 repository specification, strict APT pinning rules, and verification procedures for consuming `firefox`, `tor-browser`, and the unified `firefox-tor-suite` metapackage from OBS.

---

## Table of Contents

- [Overview](#overview)
- [Why This Repository Exists](#why-this-repository-exists)
- [Packages and Version Matrix](#packages-and-version-matrix)
- [Firefox Hardware Acceleration and Codec Matrix](#firefox-hardware-acceleration-and-codec-matrix)
- [Tor Browser Isolation and Hardening](#tor-browser-isolation-and-hardening)
- [System Requirements](#system-requirements)
- [Installation](#installation)
  - [1. Import the Repository Signing Key](#1-import-the-repository-signing-key)
  - [2. Configure the DEB822 APT Source](#2-configure-the-deb822-apt-source)
  - [3. Apply Strict APT Pinning](#3-apply-strict-apt-pinning)
  - [4. Retire Conflicting Firefox Sources (Recommended)](#4-retire-conflicting-firefox-sources-recommended)
  - [5. Install Packages](#5-install-packages)
- [Verification](#verification)
  - [APT Pinning and Signed Package Origin](#apt-pinning-and-signed-package-origin)
  - [Firefox GPU, VA-API, Vulkan, and Wayland Checks](#firefox-gpu-va-api-vulkan-and-wayland-checks)
  - [Tor Browser AppArmor and Launcher State Checks](#tor-browser-apparmor-and-launcher-state-checks)
- [Updating, Rollback, and Uninstallation](#updating-rollback-and-uninstallation)
- [Packaging Architecture and Build Pipeline](#packaging-architecture-and-build-pipeline)
- [OBS Build Links](#obs-build-links)
- [License](#license)

---

## Overview

Ubuntu 26.04 LTS replaces the native `firefox` `.deb` archive package with a Snap-redirecting transitional stub and does not ship a native, APT-managed Tor Browser package with an Ubuntu 26.04 `abi <abi/5.0>` AppArmor policy and Pluggable Transports bundled from verified upstream sources.

The `home:dwolfF:browsers` OBS repository solves both gaps with signed `.deb` packages compiled from pinned upstream sources:

- **`firefox`**: Native Mozilla Firefox compiled for Ubuntu 26.04 (`amd64`), shipping preconfigured WebRender, DMABUF, Wayland/X11-EGL display integration, VA-API/Vulkan hardware decode policy, FFmpeg 8 (`libavcodec62`), OpenH264, `dav1d` AV1, JPEG XL, HEVC, WebRTC H.264 HW, and `policies.json` to delegate updates exclusively to APT.
- **`tor-browser`**: Native Tor Browser built from pinned upstream Gecko ESR, `tor` daemon, Go Pluggable Transports (`lyrebird`, `conjure`), `moat-settings`, bundled Noto font coverage, Tor Browser Manual, and NoScript, strictly preserving upstream anti-fingerprinting defaults behind an Ubuntu 26.04 AppArmor profile.
- **`firefox-tor-suite`**: Lockstep metapackage keeping `firefox`, `tor-browser`, and shared graphics/codec/portal recommendations aligned across upgrades.

---

## Why This Repository Exists

| Aspect | Ubuntu Default (`firefox` stub / manual Tor tarball) | Third-Party Generic PPA / Mozilla APT | This OBS Repository (`home:dwolfF:browsers`) |
| --- | --- | --- | --- |
| Delivery mechanism | Snap transitional package or unpacked `$HOME` tarball | Generic cross-distro `.deb` without Ubuntu 26.04 FFmpeg 8 / Mesa `26.0.8` alignment | Signed Ubuntu `xUbuntu_26.04` native `.deb` packages |
| Ubuntu 26.04 library ABI | Confined Snap runtime or static upstream archive | Generic dynamic defaults; manual codec/GPU setup | Linked/declared against `libavcodec62`, `libopenh264-8`, `mesa-libgallium`, `libjxl0.11`, `libdav1d7`, `libvpx12` |
| Firefox HW acceleration & codecs | Mixed depending on confinement & user prefs | Default upstream prefs; legacy/unset codec knobs | Curated `obs-hwaccel-codecs.js` (18 verified Firefox 160 StaticPrefs) + `/usr/bin/firefox` Wayland/EGL/NVIDIA wrapper |
| Tor Browser isolation | Manual tarball self-updater outside APT | Typically absent or separate script wrapper | Immutable `/usr/lib/tor-browser/`, `abi <abi/5.0>` AppArmor profile, per-user `.build-id` symlink state sync, no self-updater |
| Update discipline | Background Snap refresh / browser auto-updater | Uncoordinated browser self-updates vs APT | Enterprise policy (`DisableAppUpdate`) + disabled Tor self-updater; deterministic APT upgrades |
| Build & release safety | Opaque upstream channel | Direct upload | Two-contour OBS workflow (`:staging` build + offline checks -> signed stable promotion) |

---

## Packages and Version Matrix

| Package | Upstream Base | Package Version | Role |
| --- | --- | --- | --- |
| `firefox` | Mozilla Firefox `160.0.1` | `160.0.1-1+obs1` | Native browser with FFmpeg 8, OpenH264, VA-API/Vulkan/Wayland defaults, and APT update policy |
| `tor-browser` | Tor Browser `16.0.3` (`153.8.0esr`, `tor` `0.4.9.17`, `lyrebird` `0.10.0`, `conjure`, `noscript` `13.6.39.1984`) | `16.0.3-1+obs1` | Hardened Tor Browser bundle with AppArmor `abi/5.0` profile and zero telemetry/VA-API anti-fingerprinting leaks |
| `firefox-tor-suite` | Suite lockstep metapackage | `1.0.0-1+obs1` | Enforces synchronized minimum versions of `firefox` and `tor-browser` plus recommended GPU/codec/portal drivers |

> [!NOTE]
> Staging builds compile an updated `cbindgen` (`0.29.4-1+obs1`) privately inside the `:staging` contour because Ubuntu 26.04 `universe` ships `0.29.2` while Gecko requires `>= 0.29.4`. It is a build-time dependency only and is never published to the stable APT repository.

---

## Firefox Hardware Acceleration and Codec Matrix

The `firefox` package installs `/usr/lib/firefox/defaults/pref/obs-hwaccel-codecs.js` containing 18 verified Firefox 160 StaticPrefs (the legacy `media.ffmpeg.vaapi.enabled` preference removed upstream is intentionally omitted):

| Subsystem | Enabled Preferences / Mechanism | Runtime Dependencies / Driver Support |
| --- | --- | --- |
| IPC & Compositor | `dom.ipc.forkserver.enable`, `gfx.webrender.all`, `gfx.canvas.accelerated`, `dom.webgpu.enabled` | `libdrm2`, `libgbm1`, `libegl1`, `libgl1`, `libgles2`, `libvulkan1`, `libpci3`, `mesa-vulkan-drivers` |
| Wayland & DMABUF | `widget.dmabuf.enabled`, `widget.wayland.fractional-scale.enabled`, `widget.wayland.vsync.enabled`; wrapper exports `MOZ_ENABLE_WAYLAND=1` on Wayland or `MOZ_X11_EGL=1` on X11 | `pipewire`, `xdg-desktop-portal`, `xdg-desktop-portal-gtk \| kde \| gnome` |
| FFmpeg & H.264 / HEVC | `media.ffmpeg.enabled`, `media.ffmpeg.encoder.enabled`, `media.ffmpeg.allow-openh264`, `media.webrtc.hw.h264.enabled`, `media.hevc.enabled` | `libavcodec-extra62 \| libavcodec62`, `libopenh264-8` |
| Hardware Video Decode | `media.hardware-video-decoding.enabled`, `media.hardware-video-decoding-vulkan.enabled` | `libva2`, `libva-drm2`, `libva-wayland2`, `libva-x11-2`, `mesa-libgallium`, `intel-media-va-driver-non-free \| intel-media-va-driver`, `i965-va-driver`, `nvidia-vaapi-driver`, `va-driver-all` |
| Modern Royalty-Free Media | `media.av1.enabled`, `media.av1.use-dav1d`, `image.jxl.enabled`, `image.avif.sequence.enabled` | `libdav1d7`, `libaom3`, `libvpx12`, `libjxl0.11`, `libwebp7`, `libopus0`, `libvorbis0a` |
| NVIDIA VA-API Compatibility | Wrapper auto-detects `/proc/driver/nvidia/version` + `nvidia_drv_video.so` and exports `LIBVA_DRIVER_NAME=nvidia` and `MOZ_DISABLE_RDD_SANDBOX=1` (overridable via environment) | `nvidia-vaapi-driver` (never disables content sandboxing with `--no-sandbox`) |

On Ubuntu 26.04, Mesa `26.0.8` folds `radeonsi_drv_video.so`, `nouveau_drv_video.so`, and `virtio_gpu_drv_video.so` directly into `mesa-libgallium` (`mesa-va-drivers` does not exist on Ubuntu 26.04). The package dependencies reflect that split natively.

---

## Tor Browser Isolation and Hardening

Tor Browser packaging prioritizes browser fingerprint uniformity and compartmentalization over host desktop codec/GPU integration:

- **Strict upstream anti-fingerprinting defaults:** No VA-API decode, no WebGPU override, no hardware codec preference file, and no host font overrides are injected. Bundled Noto, CJK, Emoji, and UI fonts remain self-contained inside `/usr/lib/tor-browser/`.
- **Immutable system bundle + writable user state:** The browser tree resides read-only under `/usr/lib/tor-browser/`, while `/usr/bin/tor-browser` initializes `${XDG_DATA_HOME:-$HOME/.local/share}/tor-browser/app/TorBrowser/Data/{Browser,Tor}` with `0700` permissions and atomically reconciles symlinks whenever `.build-id` changes after an APT upgrade.
- **Ubuntu 26.04 AppArmor enforcement:** Installs `/etc/apparmor.d/tor-browser` targeting `abi <abi/5.0>`.
- **No self-updater races:** Compiled with `--disable-updater` and `--disable-base-browser-update` so system updates happen exclusively via signed `apt` transactions.

---

## System Requirements

- **OS:** Ubuntu 26.04 LTS (`resolute` / `xUbuntu_26.04`)
- **Architecture:** `amd64` (`x86_64`)
- **Privileges:** `sudo` access for configuring `/etc/apt/` and installing packages
- **Optional GPU drivers:** Ubuntu `mesa-libgallium`, `mesa-vulkan-drivers`, `intel-media-va-driver` / `intel-media-va-driver-non-free`, `i965-va-driver`, or `nvidia-vaapi-driver` matching your GPU vendor

---

## Installation

### 1. Import the Repository Signing Key

Download and verify the ASCII-armored OBS release key into `/etc/apt/keyrings/home_dwolfF_browsers.gpg`:

```bash
sudo install -d -m 0755 /etc/apt/keyrings
curl -fsSL https://download.opensuse.org/repositories/home:/dwolfF:/browsers/xUbuntu_26.04/Release.key \
  | gpg --dearmor \
  | sudo tee /etc/apt/keyrings/home_dwolfF_browsers.gpg >/dev/null
sudo chmod 0644 /etc/apt/keyrings/home_dwolfF_browsers.gpg
```

Verify the key fingerprint before trusting packages:

```bash
gpg --show-keys --with-fingerprint /etc/apt/keyrings/home_dwolfF_browsers.gpg
```

Expected fingerprint:

```text
C106 E46D E757 8257 F4CD  62C9 6AE3 5907 289D 3BDC
```

### 2. Configure the DEB822 APT Source

Create `/etc/apt/sources.list.d/home_dwolfF_browsers.sources`:

```bash
sudo tee /etc/apt/sources.list.d/home_dwolfF_browsers.sources >/dev/null <<'EOF'
Types: deb
URIs: https://download.opensuse.org/repositories/home:/dwolfF:/browsers/xUbuntu_26.04/
Suites: /
Signed-By: /etc/apt/keyrings/home_dwolfF_browsers.gpg
Architectures: amd64
EOF
```

### 3. Apply Strict APT Pinning

Create `/etc/apt/preferences.d/firefox-tor-ubuntu-obs` to pin OBS packages above default archives while blocking Ubuntu's transitional Snap stub and third-party Mozilla APT overrides:

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

### 4. Retire Conflicting Firefox Sources (Recommended)

If your system previously had Ubuntu's Snap Firefox or Mozilla's upstream APT source (`packages.mozilla.org`), clean them up so updates are unambiguous:

```bash
# Optional: remove Ubuntu Snap Firefox if installed
if command -v snap >/dev/null 2>&1 && snap list firefox >/dev/null 2>&1; then
  sudo snap remove --purge firefox
fi

# Refresh APT metadata after applying pinning
sudo apt-get update
```

### 5. Install Packages

Install the full suite (recommended) or individual browsers:

```bash
# Install both Firefox and Tor Browser in lockstep
sudo apt-get install --allow-downgrades firefox-tor-suite

# Or install individually
sudo apt-get install firefox
sudo apt-get install tor-browser
```

> [!NOTE]
> Use `--allow-downgrades` on first migration if a higher-numbered package from `packages.mozilla.org` is already installed on the host.

---

## Verification

### APT Pinning and Signed Package Origin

Confirm candidate versions, `1002` priority from `download.opensuse.org`, and `-1` suppression of Ubuntu / Mozilla origins:

```bash
apt-cache policy firefox tor-browser firefox-tor-suite
```

### Firefox GPU, VA-API, Vulkan, and Wayland Checks

Verify installed policy, wrapper prefs, codec/VA-API libraries, and Vulkan enumeration:

```bash
test -f /usr/lib/firefox/defaults/pref/obs-hwaccel-codecs.js
test -f /usr/lib/firefox/distribution/policies.json
vainfo
vulkaninfo --summary
```

Inside Firefox, open `about:support` and verify:

- **Window Protocol:** `wayland` (on Wayland sessions) or `x11` with EGL compositing
- **Compositing:** `WebRender`
- **Hardware Video Decoding / WebGPU / Codecs:** `H264`, `VP9`, `AV1`, `HEVC` hardware/software capabilities matching installed drivers

### Tor Browser AppArmor and Launcher State Checks

Reload AppArmor profiles if needed and verify `/etc/apparmor.d/tor-browser` is enforced:

```bash
sudo apparmor_parser -r /etc/apparmor.d/tor-browser
sudo aa-status | grep tor-browser
```

Launch `tor-browser` once and confirm isolated state creation under `${XDG_DATA_HOME:-$HOME/.local/share}/tor-browser/app/TorBrowser/Data/`.

---

## Updating, Rollback, and Uninstallation

### Regular Updates

Both browsers disable internal auto-updaters (`policies.json` `DisableAppUpdate` and Tor's `--disable-updater`) and receive updates strictly via standard APT upgrades:

```bash
sudo apt-get update && sudo apt-get upgrade
```

### Uninstalling Packages and Repository Configuration

```bash
# Remove packages
sudo apt-get purge firefox-tor-suite firefox tor-browser
sudo apt-get autoremove

# Remove repository source, pin rules, and GPG key
sudo rm -f /etc/apt/sources.list.d/home_dwolfF_browsers.sources \
           /etc/apt/preferences.d/firefox-tor-ubuntu-obs \
           /etc/apt/keyrings/home_dwolfF_browsers.gpg
sudo apt-get update
```

User profiles under `~/.mozilla/firefox/` and `${XDG_DATA_HOME:-$HOME/.local/share}/tor-browser/` are preserved unless explicitly removed.

---

## Packaging Architecture and Build Pipeline

The packaging workflow is organized into two isolated OBS contours:

1. **Staging Contour (`home:dwolfF:browsers:staging`):**
   - `sourceaccess=disable`, `publish=disable`, building `xUbuntu_26.04` (`x86_64`) against Ubuntu 26.04 updates and universe (`rustc-1.97`, `cargo-1.97`, `clang-22`, `lld-22`, `llvm-22`, `libclang-22-dev`, bundled `NSS 3.131`, system `zlib`).
   - Enforces a strict no-`ccache` policy (`Build-Conflicts: ccache, sccache`, `--without-ccache`, `unexport CCACHE`).
   - Built `.deb` artifacts undergo offline dpkg/layout/dependency/policy verification inside an isolated `--network none` `ubuntu:26.04` container before promotion.
2. **Stable Publishing Contour (`home:dwolfF:browsers`):**
   - Direct building disabled (`build=disable`), publishing enabled (`publish=enable`).
   - Receives verified binaries via manual OBS release only after staging checks pass (`PublishFlags: nofailedpackages`).
   - `firefox-tor-suite` is only promoted when compatible `firefox` and `tor-browser` builds are both present in stable.

---

## OBS Build Links

- **Stable Project & Package Status:** [home:dwolfF:browsers](https://build.opensuse.org/project/show/home:dwolfF:browsers)
- **APT Repository Index (`xUbuntu_26.04`):** [download.opensuse.org/repositories/home:/dwolfF:/browsers/xUbuntu_26.04/](https://download.opensuse.org/repositories/home:/dwolfF:/browsers/xUbuntu_26.04/)
- **Staging Project:** [home:dwolfF:browsers:staging](https://build.opensuse.org/project/show/home:dwolfF:browsers:staging)

---

## License

Packaging metadata, documentation, and launcher/configuration assets are licensed under the **Mozilla Public License 2.0 (MPL-2.0)** unless otherwise noted in [LICENSE](LICENSE). Upstream Mozilla Firefox and Tor Browser source code remains subject to their respective upstream licenses.
