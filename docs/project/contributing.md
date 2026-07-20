# Contributing

All source lives in the
[Polycom-Open-Firmware](https://github.com/Polycom-Open-Firmware) GitHub
organization. Issues and pull requests are welcome in whichever repository
owns the affected component.

## Which repo does what

| Repository | Role |
|---|---|
| [poly-firmware-build](https://github.com/Polycom-Open-Firmware/poly-firmware-build) | Image composer for both targets: builds the kernel, initramfs, and boot images, and packs the rootfs variants |
| [polycom-uboot](https://github.com/Polycom-Open-Firmware/polycom-uboot) | U-Boot ports for both devices: the C60 board port and the TC8 chainloaded stage-2 |
| [poly-kernel-patches](https://github.com/Polycom-Open-Firmware/poly-kernel-patches) | The mainline-kernel patch set (board device trees, panel, switch, audio drivers), consumed as a submodule of the image composer |
| [poly-rootfs](https://github.com/Polycom-Open-Firmware/poly-rootfs) | Debian rootfs builder: debootstrap base, essentials, and per-role/per-board variant machinery |
| [packages](https://github.com/Polycom-Open-Firmware/packages) | Source monorepo for `poly-*` Debian packages: profile and application metapackages, keyring, small tools |
| [apt](https://github.com/Polycom-Open-Firmware/apt) | The package archive's only writer: publishing pipeline and CI for the hosted apt repository |
| [provisioner](https://github.com/Polycom-Open-Firmware/provisioner) | The browser Setup Wizard: unlock, install, application picker, and device configuration |
| [chromium-a53](https://github.com/Polycom-Open-Firmware/chromium-a53), [poly-kodi](https://github.com/Polycom-Open-Firmware/poly-kodi) | Applications with their own upstream or build gravity live in their own repos; their packages enter the archive via the apt repo's publish paths |

For a deeper map of how the pieces fit together — and how to build
everything from source — see the [source map](../build/index.md).

## Where to start

- **Bugs and feature requests** — open an issue on the repo above that
  owns the component. If unsure, `poly-firmware-build` for anything about
  the installed OS, `provisioner` for anything about the wizard.
- **New applications and device roles** — the package/profile cookbook
  lives in the `packages` repo; see
  [Add an application](../extend/add-an-app.md).
- **Code contributions** — standard GitHub flow: fork, branch, pull
  request against the relevant repo.
