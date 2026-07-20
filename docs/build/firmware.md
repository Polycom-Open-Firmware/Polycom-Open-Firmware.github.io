# Firmware images

[`poly-firmware-build`](https://github.com/Polycom-Open-Firmware/poly-firmware-build)
builds the complete sideloaded Linux image from a fresh checkout. **One
repo, two targets** — `--target=` picks the board:

- `--target=tc8` (default) — the TC8 panel: a `boota` slot image
  (`boot.img` + `dtbo.img` + `vbmeta.img`, unsigned AVB) plus a sparse
  `rootfs.simg` flashed to `userdata`.
- `--target=c60` — the Trio C60: a `booti` image set (`boot.img` with the
  DTB in its `second` area, Android-DTBO `dtbo.img`, **signed** vbmeta)
  plus `rootfs.img.zst` for `system_a`.

A target is `targets/<t>/target.env` (board facts: DTB, partitions, boot
geometry) plus `targets/<t>/boot.sh` (the boot-image recipe). The kernel
is `kernel/config.base` merged with `kernel/targets/<t>.frag`, patched
from `kernel-patches/patches/<t>/`, and the rootfs builder gets
`--device=<t>` so the right per-board profile package lands. Everything
below applies to both targets unless it names one; the worked examples use
the TC8.

Two build profiles share the same kernel and rootfs:

- `emmc` — the slotable Android image, booted by `boota`/`booti` and
  installed by the [Setup Wizard](https://wizard.openpolycom.cc/)
- `nfs` — netboot bring-up: kernel over TFTP, rootfs over NFS (see
  [Network boot](../use/netboot.md))

The TC8 slot image carries **unsigned** AVB metadata
(`tools/avbtool … --algorithm NONE`) and boots because the unit is
unlocked — the
[TC8 boot chain](../internals/tc8-boot.md) page has the whole story.

## 1. Install prerequisites

On a fresh Ubuntu 22.04+ host:

```bash
sudo apt update
sudo apt install -y \
    git build-essential bison flex bc kmod \
    gcc-aarch64-linux-gnu \
    qemu-user-static binfmt-support \
    debootstrap rsync \
    e2fsprogs python3 \
    zstd
```

`binfmt-support` registers `/proc/sys/fs/binfmt_misc/qemu-aarch64` so
`debootstrap --second-stage` can run arm64 binaries inside the chroot.
Verify:

```bash
ls /proc/sys/fs/binfmt_misc/qemu-aarch64    # should exist
```

## 2. Clone

```bash
git clone --recurse-submodules https://github.com/Polycom-Open-Firmware/poly-firmware-build.git
cd poly-firmware-build
./bootstrap.sh                              # downloads vanilla linux-6.6
```

`kernel-patches` and `rootfs` are sibling repos under the same org,
pulled as submodules — hence `--recurse-submodules`.

## 3. (Optional) Bake credentials

Default credentials are **`root` / `root`**, working on tty, USB CDC ACM,
and ssh. To change:

```bash
echo 'mySecret' > root_password         # gitignored
# or: export TC8_ROOT_PASSWORD=mySecret
```

To pre-authorize an SSH pubkey for `root`:

```bash
cat ~/.ssh/id_ed25519.pub > authorized_keys     # gitignored
# or: export TC8_SSH_PUBKEY=~/.ssh/id_ed25519.pub
```

SSH host keys are generated at image build time (`ssh-keygen -A` in the
build chroot) and are never committed.

## 4. Build

```bash
sudo ./build.sh --profile=emmc                  # TC8 (--target=tc8 is the default)
sudo ./build.sh --target=c60 --profile=emmc     # Trio C60
```

`sudo` is needed for the chroot bind-mounts and `debootstrap`. The first
invocation builds the rootfs (~10 min), the kernel (~3 min), and the
rootfs image (~30 s); subsequent invocations cache.

TC8 outputs:

```
out/emmc/boot.img               # Android boot.img v0 + AVB hash footer (NONE) -> fastboot flash boot_a
out/emmc/dtbo.img               # DTB in Android DTBO container + AVB footer    -> fastboot flash dtbo_a
out/emmc/vbmeta.img             # AVB vbmeta, hash descriptors boot+dtbo (NONE) -> fastboot flash vbmeta_a
out/emmc/rootfs.simg            # Android sparse rootfs                         -> fastboot flash userdata
out/emmc/Image                  # raw kernel (netboot)
out/emmc/imx8mm-tc8.dtb         # raw device tree (netboot)
out/emmc/rootfs.img             # 6.4 GiB ext4, sized to the userdata partition (sparse on disk; ~2 GiB used)
out/emmc/initramfs.cpio.gz      # busybox ro-root initramfs (emitted by default; embedded in boot.img)
out/emmc/version.env            # TC8_FW_VERSION, build host, etc.
out/emmc/SHA256SUMS
```

The C60 build emits the same set with `imx8mm-kepler-proto1.dtb`, a
signed `vbmeta.img`, and `rootfs.img.zst` in place of `rootfs.simg`.

!!! note "C60 rootfs artifacts: `.img.zst` vs `.simg`"
    `build.sh --target=c60` emits `rootfs.img.zst` — a zstd-compressed
    ext4 image, a development artifact. A released, wizard-installable
    C60 rootfs is `rootfs.simg` (Android sparse format, flashed to
    `system_a`); converting the ext4 image to sparse format is a manual
    release step.

## 5. Install

The production path is the
[Setup Wizard](https://wizard.openpolycom.cc/): get the unit into the
stage-2 fastboot gadget (one-time serial bootstrap on a fresh unit, or the
four-finger gesture once enrolled), then `flashos` writes
`boot_a`/`dtbo_a`/`vbmeta_a` plus the sparse rootfs, sets the active slot,
and reboots. See [How installation works](../install/index.md); the
repo's
[`FLASHING.md`](https://github.com/Polycom-Open-Firmware/poly-firmware-build/blob/main/FLASHING.md)
and
[`NETBOOT.md`](https://github.com/Polycom-Open-Firmware/poly-firmware-build/blob/main/NETBOOT.md)
cover the manual and development paths.

## 6. Iterate

```bash
./build.sh --profile=emmc --skip-rootfs                 # keep rootfs tarball, rebuild kernel + repack
./build.sh --profile=emmc --skip-kernel --skip-rootfs   # only re-pack from existing artifacts
./build.sh --profile=emmc --rootfs-size=4G              # smaller rootfs image (faster fastboot push)
./build.sh --profile=emmc --os-profile=kiosk,bare       # device-role variants (see below)
```

Tweaking `kernel/config.base` or `kernel/targets/<t>.frag` does not
trigger a rootfs rebuild.

## Device-role profiles (`--os-profile=`)

Not to be confused with `--profile=emmc|nfs` (the *build target*): an
**OS profile** is the device's role — what it boots into. Each named
profile is a metapackage installed from the OpenPolycom apt archive into
an isolated copy of the shared Debian base and packed as its own
`rootfs-<name>` image; plain `rootfs.simg`/`rootfs.img` always alias the
default (`kiosk`) so existing tooling and the wizard keep working. The
metapackage model, naming, and the cookbook for adding a role live in
[Packages and profiles](../extend/index.md).

## Size guards and budgets

The builder enforces hard limits instead of emitting images that flash
but never boot:

- **Kernel `Image` < 32 MiB (TC8).** The stock U-Boot 2018.03 that
  chainloads the TC8 caps `bootm` loads at 32 MiB (`BOOTM_LEN`); CI
  fails the release build past that. ~24 MiB is fine. If the Image grows
  past the cap, add SoC families to `kernel/config.base`'s
  `# CONFIG_ARCH_… is not set` block to drop them. (The C60 runs its own
  U-Boot and is not bound by this cap.)
- **TC8 `rootfs.img` ≤ the `userdata` partition** (6,843,006,976 bytes).
  The kernel refuses to mount an ext4 bigger than its partition, so
  `build.sh` errors out rather than emit an oversized image.
- **C60 rootfs ≤ 1.6 GiB by default; `system_a` is the hard ceiling.**
  The C60 rootfs lands in the 1.75 GiB (1,879,048,192-byte) `system_a`
  partition. The builder's default image size is a headroom-safe
  1.6 GiB, and it refuses — never truncates — an image over the ceiling.

## What's forced built-in

The rootfs ships **no `/lib/modules/`**, so every needed driver is
compiled into the kernel (`=y`): the DRM/DSI display stack (etnaviv,
LCDIF, Samsung DSIM + Mixel PHY), PWM backlight, SAI/SDMA audio, the
RTL8365MB DSA switch, the USB CDC ACM console gadget, and the
netboot stack (IP autoconf + NFS root). The full list with rationale is
in the repo's
[`BUILDING.md`](https://github.com/Polycom-Open-Firmware/poly-firmware-build/blob/main/BUILDING.md).
After a config change, verify on the running device via
`/proc/config.gz` — kconfig silently demotes `=y` to `=m` if a hard
dependency is `=m`.
