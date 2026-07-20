# Source map

Everything OpenPolycom ships is built from public repositories under the
[Polycom-Open-Firmware](https://github.com/Polycom-Open-Firmware) GitHub
organization. This page maps each repo to what it builds or owns; the rest
of the Build section walks through the individual builders.

| Repo | Builds / owns |
|---|---|
| [`poly-firmware-build`](https://github.com/Polycom-Open-Firmware/poly-firmware-build) | Image composer for both targets: `--target=<tc8\|c60>` builds the kernel, initramfs, `boot.img`, and packs `--os-profile=` rootfs variants (`boota` sparse images for the TC8, `booti` + zstd rootfs for the C60). See [Firmware images](firmware.md). |
| [`poly-rootfs`](https://github.com/Polycom-Open-Firmware/poly-rootfs) | The Debian bookworm arm64 rootfs: debootstrap base + pre-apt essentials (sources.list, keyring, users, fstab), the `--profile=` variant machinery, and `--device=<tc8\|c60>` for the per-board profile package — one repo, both boards. Consumed as a submodule of `poly-firmware-build`. See [Debian rootfs](rootfs.md). |
| [`poly-kernel-patches`](https://github.com/Polycom-Open-Firmware/poly-kernel-patches) | Per-target patch series applied to vanilla Linux 6.6 by the kernel build. Consumed as a submodule of `poly-firmware-build`. |
| [`polycom-uboot`](https://github.com/Polycom-Open-Firmware/polycom-uboot) | The bootloaders: a stage-1 U-Boot replacement for the Trio C60 and a chainloaded stage-2 U-Boot for the TC8. See [U-Boot](uboot.md). |
| [`packages`](https://github.com/Polycom-Open-Firmware/packages) | One `debian/` tree per package: all profile metapackages, `poly-archive-keyring`, `poly-*-base`, and small tools. See [Packages and profiles](../extend/index.md). |
| [`apt`](https://github.com/Polycom-Open-Firmware/apt) | **The only writer** to the published package archive: stateless `publish.sh` (apt-ftparchive + GPG) plus CI; the pool and indexes live only in Cloudflare R2. See [The apt archive](../extend/apt-archive.md). |
| [`provisioner`](https://github.com/Polycom-Open-Firmware/provisioner) | The [Setup Wizard](https://wizard.openpolycom.cc/): profile picker, per-profile settings pages, and browser-based flashing. |
| [`chromium-a53`](https://github.com/Polycom-Open-Firmware/chromium-a53) | Applications with their own upstream or build gravity live in their own repos; their debs enter the archive via the apt repo's publish paths. `chromium-a53` is the worked example — see [Chromium](chromium.md). |

## How they compose

The pipeline runs **packages → apt archive → rootfs → image → wizard**:

1. The `packages` repo (and standalone app repos like `chromium-a53`)
   build plain Debian packages, which the `apt` repo publishes to the
   GPG-signed OpenPolycom archive.
2. `poly-rootfs` debootstraps the Debian base and installs one role
   metapackage per requested profile from that archive, emitting one
   rootfs tarball per profile.
3. `poly-firmware-build` builds the kernel (vanilla 6.6 +
   `poly-kernel-patches`) and composes it with each rootfs into a
   flashable image set per target.
4. The `provisioner` wizard flashes the image variant matching the role
   the user picks — on hardware whose boot chain the `polycom-uboot`
   bootloaders opened up.

Because the metapackages pin nothing, an image build always resolves
against the archive's current state: publish a new package version and the
next image build picks it up with no change to the image repos. The full
package/profile architecture is in
[Packages and profiles](../extend/index.md).
