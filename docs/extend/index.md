# Packages and profiles

How OpenPolycom turns Debian packages into device roles. The short
version: every piece of software is a plain Debian package in the
[OpenPolycom apt archive](apt-archive.md), a device's role is a thin
metapackage that depends on the right ones, and the image builders
resolve those metapackages at build time — so images never pin versions
and the wizard flashes a fully-baked rootfs per role.

## The metapackage model

Devices run a sealed OS (read-only Debian bookworm arm64 rootfs — see the
[storage model](../use/storage.md)) whose *role* — web kiosk, media
player, … — is an **application**: a metapackage whose `Depends:` pulls
in the apps and services for that role. Profiles are resolved at **flash
time**: the image builders produce one rootfs variant per profile
(`rootfs-<name>.simg`), and the
[Setup Wizard](https://wizard.openpolycom.cc/) flashes the one the user
picks. Nothing is downloaded by the device at runtime.

Per-device settings (kiosk URL, media source, …) are **not** baked into
images either: they arrive as a config blob in the `cache` partition,
applied at the next boot — see the
[configuration reference](../use/configure.md).

## Naming schema

Everything carries the `poly-` prefix:

- **`poly-app-<id>`** — an application (role) metapackage.
  **Board-agnostic** (`Architecture: all` unless it ships binaries). An
  application that needs board hardware is restricted in the wizard's
  catalog (`boards:` metadata on the Application entry), not by per-board
  packages.
- **`poly-<device>-profile-<name>`** — the legacy per-board naming
  (e.g. `poly-tc8-profile-kiosk`, `poly-c60-profile-kiosk`). The rootfs
  builder installs `poly-app-<id>` when it exists and falls back to the
  per-board name.
- **`poly-archive-keyring`**, **`poly-*-base`** — archive trust and base
  plumbing, from the
  [`packages` repo](https://github.com/Polycom-Open-Firmware/packages).

## How images consume profiles

The rootfs builder (`poly-rootfs/build.sh --profile=a,b --device=<tc8|c60>`)
debootstraps the Debian base **once**, then per profile:

1. makes an isolated `rsync` clone of the base,
2. chroot-installs that profile's metapackage from the archive
   (`--device=` selects the per-board fallback name),
3. stamps `TC8_PROFILE` into `/etc/tc8-version`, and
4. emits `rootfs-<name>.tar.gz`.

The image composer
(`poly-firmware-build/build.sh --profile=emmc --os-profile=a,b`) packs
each tarball into `rootfs-<name>.{img,simg}`; plain `rootfs.simg` always
aliases the default profile (`kiosk`) so existing tooling keeps working.
(`--profile=` selects the emmc/nfs *build* target; `--os-profile=` is the
device role.) The special profile `bare` is the untouched base.

The built variant list lands as `TC8_OS_PROFILES` in the image's
`version.env`; each image's own role is `TC8_PROFILE` in
`/etc/tc8-version` on the device.

**C60 size budget:** the hard ceiling is the 1.75 GiB `system_a`
partition; the builder's default budget is **1.6 GiB**, and it refuses —
never truncates — oversized images.

## Thin metapackages: images pin nothing

Profile metapackages are deliberately thin: a `Depends:` list and a
description, no versions, no payload. Services and config belong in the
*app* packages a profile depends on, not in the metapackage. Because
nothing is pinned, every image build resolves against the archive's
current head — publishing a new package version updates the next image
build without touching the image repos, and the same archive serves live
`apt install` in maintenance mode on devices.

## Where things live

| Concern | Where |
|---|---|
| Metapackages, keyring, base plumbing, small tools | [`packages`](https://github.com/Polycom-Open-Firmware/packages) repo |
| Heavy apps with their own upstream or build gravity | Their own repos (e.g. [Chromium](../build/chromium.md)) |
| Publishing and consuming the archive | [The apt archive](apt-archive.md) |
| The rootfs variant machinery | [Debian rootfs](../build/rootfs.md) |
| Image packing per target | [Firmware images](../build/firmware.md) |
| The role picker users see | [Choosing a role](../use/applications/index.md) |

See [Add an application](add-an-app.md).
