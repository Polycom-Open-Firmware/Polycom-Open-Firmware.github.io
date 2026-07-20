# Add an application

Cookbook for taking a new device role from empty directory to a
selectable entry in the [Setup Wizard](https://wizard.openpolycom.cc/).
Background on the model: [Packages and profiles](index.md). The worked
example is a digital-signage role, `poly-app-signage`.

!!! warning "No network at first boot — bake everything in"
    A flashed device routinely comes up with no internet (no DHCP, no
    uplink, air-gapped install). **Everything a role needs must be in the
    baked image.** The archive and the baked `sources.list` are for
    *build-time* installs (in the chroot, on the build host's network)
    and *maintenance-mode* installs (explicit `tc8-rw` with user-provided
    network) — never for first boot. Hard rule: no boot-path service or
    first-boot script may `apt`/`curl`/`wget` anything. A role's
    `Depends:` is satisfied at image build; if the role needs it at
    runtime, it is already on disk. Corollary: no network means no NTP —
    the clock starts from baked/persisted state; don't gate boot on time
    sync.

## 1. Create the metapackage

In the [`packages` repo](https://github.com/Polycom-Open-Firmware/packages):

```sh
mkdir -p poly-app-signage/debian/source
```

`debian/control`:

```
Source: poly-app-signage
Section: metapackages
Priority: optional
Maintainer: OpenPolycom <apt@openpolycom.cc>
Build-Depends: debhelper-compat (= 13)
Standards-Version: 4.6.2

Package: poly-app-signage
Architecture: all
Depends: ${misc:Depends}, poly-app-yourthing, some-debian-pkg
Description: Device application: digital signage
 One-line role description. What it boots into, what it depends on.
```

Plus `debian/changelog` (start at `0.1.0`, suite `stable`),
`debian/rules` (the 3-line dh boilerplate), and `debian/source/format`
(`3.0 (native)`). Copy an existing profile directory as the template.
`Architecture: all` keeps the metapackage board-agnostic; if the role
needs board hardware, restrict it in the wizard's catalog (step 6), not
with per-board packages.

Services and config the role needs live in its *app* packages, not the
metapackage — systemd units via `debian/<pkg>.install` plus
`dh_installsystemd`. Package postinst runs at image build (in the chroot)
or in maintenance mode — never against a sealed root — so normal Debian
packaging just works. An app that ships binaries uses
`Architecture: arm64` and real content instead of a bare `Depends:`;
apps with their own upstream or a long build get their own repo (see
[Chromium](../build/chromium.md) for the worked heavy example).

## 2. Test on a device without publishing

The fastest loop — build the deb locally and install it in maintenance
mode:

```sh
cd poly-app-signage && dpkg-buildpackage -us -uc -b
scp ../poly-app-signage_*.deb root@<device-ip>:/tmp/
# on the device: tc8-rw && reboot; apt install /tmp/poly-…deb; test; tc8-ro && reboot
```

Anything installed on a *sealed* device is ephemeral — it evaporates on
reboot. `tc8-rw`/`tc8-ro` are explained in the
[storage model](../use/storage.md).

## 3. Publish

Push to the `packages` repo — CI builds every package and dispatches the
[apt repo](apt-archive.md), the archive's single writer — or use one of
the manual publish paths described there. Verify from any device in
maintenance mode:

```sh
apt update && apt-cache policy poly-app-signage
```

## 4. Build the image variant

In `poly-firmware-build`:

```sh
sudo ./build.sh --profile=emmc --os-profile=kiosk,signage
# → out/emmc/rootfs-signage.simg
```

Flash `rootfs-signage.simg` to the rootfs partition (plus the usual
boot/dtbo/vbmeta if updating), boot, and confirm the role comes up with
zero manual steps. On the C60, check the size budget with the builder
before publishing — the default budget is 1.6 GiB against the 1.75 GiB
`system_a` ceiling, and the builder refuses oversized images.

## 5. Add config keys (if the role has settings)

Add the keys to `CONFIG-PARTITION.md` in `poly-firmware-build` (the
config-blob contract) and implement them in
`rootfs/etc/tc8-config/apply-config.sh`. Handlers must be
**idempotent** — every new blob re-runs every handler against the
already-configured system. User-facing key semantics belong in the
[configuration reference](../use/configure.md).

## 6. Add the wizard entry

In the `provisioner` repo, add the role to the device's list in
`packages/core/src/profiles/<device>.ts` and give it a `SettingsSection`
for its keys. A role that needs board hardware is restricted here via
`boards:` metadata on the Application entry. Release assets are named
`rootfs-<id>.simg`; plain `rootfs.simg` is the default-profile alias.

## Gotchas

- **Bump `debian/changelog` for every rebuild** — apt serves the highest
  version, so a stale version number silently wins.
- **Sealed rootfs**: `apt install` on a sealed device is ephemeral; use
  the maintenance-mode flow for anything that should stick during
  testing.
- **Key hygiene**: never commit the archive's private key; the public
  keyring is the `poly-archive-keyring` package, baked into the bases.
- **Version stamps**: `TC8_OS_PROFILES` (built variants) in
  `version.env`; `TC8_PROFILE` (this image's role) in `/etc/tc8-version`
  on the device.
