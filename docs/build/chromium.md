# Chromium

[`chromium-a53`](https://github.com/Polycom-Open-Firmware/chromium-a53)
is a recipe (not a fork) for building a hardware-accelerated Chromium for
the panels. The TC8 and the Trio C60 share the same i.MX 8M Mini SoC, so
one arm64 binary serves both: Debian's `chromium` **150** source package
from bookworm-security, cross-built amd64 → arm64, tuned for the
Cortex-A53 and the boards' etnaviv GPU + Hantro VPU.

## Why a custom build

- **Debian bookworm-security as the source.** The Debian security team
  rebases chromium onto every upstream stable, so the pinned version
  tracks the current milestone while building against the same bookworm
  toolchain the images use — and the recipe keeps Debian's packaging,
  launcher, and `/etc/chromium.d` flag mechanism for free. It also avoids
  the ~100 GB depot_tools checkout (a ~950 MB orig tarball instead).
- **Cortex-A53 tuning.** The recipe adds `-mcpu=cortex-a53` scheduling
  (armv8-a + crc, no crypto extensions — matching the i.MX 8M Mini
  silicon exactly), pins V8 pointer compression for the 2 GiB RAM budget,
  and pins the V4L2 stateless HEVC delegate for the Hantro G2.
- **Ozone/Wayland + V4L2 at runtime.** The companion `poly-app-chromium`
  package ships the `/etc/chromium.d` flags that run Chromium natively on
  Wayland under weston (`--ozone-platform=wayland`) and switch on
  hardware video decode through the V4L2 stateless API
  (`--enable-features=AcceleratedVideoDecoder`,
  `--ignore-gpu-blocklist`).

On the TC8, Chromium is the kiosk browser engine: `kiosk-launch` starts
it when installed (cog/WPE is the lighter default otherwise — see
[Debian rootfs](rootfs.md)). C60 images exclude Chromium: the C60 rootfs
must fit the 1.6 GiB image budget of its `system_a` partition (see
[Firmware images](firmware.md)).

## Rebuilding

The build runs in a debootstrapped bookworm amd64 chroot using Debian's
native `cross` build profile; the few build-time tools that must execute
run under qemu-user. In outline:

```sh
sudo scripts/setup-chroot.sh <build-dir>/chroot   # once
scripts/fetch-source.sh <build-dir>/chroot        # pinned, sha256-verified source
sudo OUT_DIR=<build-dir>/out scripts/build.sh <build-dir>/chroot
```

Budget: ~12 amd64 cores, ≥ 30 GiB RAM, ≥ 150 GB disk, 3–6 hours to the
debs; interrupted builds resume. The full recipe — GN arguments and their
rationale, the cross-build environment fixes, runtime-flag reference, and
version-rebase procedure — is the
[`chromium-a53` README](https://github.com/Polycom-Open-Firmware/chromium-a53).
The resulting debs enter the OpenPolycom archive via the
[apt repo's publish paths](../extend/apt-archive.md).
