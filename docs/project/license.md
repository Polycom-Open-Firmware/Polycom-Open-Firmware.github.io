# License

OpenPolycom is spread across several repositories in the
[Polycom-Open-Firmware](https://github.com/Polycom-Open-Firmware)
organization; each declares its own license. The bootloader and kernel
work is GPL-2.0 as derivative work of U-Boot and the Linux kernel.

## Per-repository

| Repository | License |
|---|---|
| [poly-firmware-build](https://github.com/Polycom-Open-Firmware/poly-firmware-build) | **GPL-2.0-only**, matching the kernel patches it depends on; ships the GPLv2 text as `LICENSE` |
| [poly-kernel-patches](https://github.com/Polycom-Open-Firmware/poly-kernel-patches) | **GPL-2.0-only** — the patches are derivative work of GPL-2.0-only kernel source |
| [provisioner](https://github.com/Polycom-Open-Firmware/provisioner) | **GPL-2.0-or-later**, declared in the package manifests; ships the GPLv2 text as `LICENSE` |
| [polycom-uboot](https://github.com/Polycom-Open-Firmware/polycom-uboot) | No repository-level license file. The sources are derivative work of U-Boot and carry U-Boot's SPDX identifiers (**GPL-2.0+**, a few files GPL-2.0+ OR MIT) |
| [packages](https://github.com/Polycom-Open-Firmware/packages) | The repository does not declare a license |

Firmware images built from these sources also contain Debian packages;
those retain their own upstream licenses.

## Trademarks and affiliation

OpenPolycom is a community project. It is **not affiliated with, endorsed
by, or supported by HP Inc., Poly, or Polycom**. "Polycom", "Poly", and
the device names (TC8, Trio C60) are trademarks of their respective
owners and are used only to identify the hardware the firmware runs on.
