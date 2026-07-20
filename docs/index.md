# OpenPolycom

**Open firmware for decommissioned Polycom conference hardware.**

Poly video-conferencing rooms get decommissioned and their hardware usually
ends up as e-waste. OpenPolycom gives two of those devices — the **TC8**
touch panel and the **Trio C60** conference phone — a second life as small
Debian Linux machines: a web kiosk, a dashboard display, a media player, or
a plain headless Debian box.

[Open the Setup Wizard](https://wizard.openpolycom.cc/){ .md-button .md-button--primary }
[Which device do I have?](devices/index.md){ .md-button }

The wizard runs entirely in the browser (Chrome or Edge): plug the device
into a USB port, open the page, and click through. Nothing to install, no
drivers, no command line.

!!! note "Not affiliated with HP or Poly"
    OpenPolycom is a community project. It is not affiliated with, endorsed
    by, or supported by HP Inc., Poly, or Polycom. "Polycom", "Poly", and
    the device names are trademarks of their respective owners, used here
    only to identify the hardware.

## What the firmware is

Both devices are built around the same NXP **i.MX 8M Mini** SoC, so they
share one software stack:

- **A custom U-Boot** for each device. The TC8 ships with secure boot
  fused on, so the project's U-Boot runs as a chainloaded second stage
  behind the untouched factory bootloader; the C60 accepts the project's
  U-Boot directly, loaded over USB by the wizard.
- **Mainline-based Linux** — vanilla kernel plus a small patch set that
  enables each board's display, touch, audio, and networking.
- **A sealed Debian root filesystem** — the OS mounts read-only behind a
  tmpfs overlay, so every reboot comes up pristine; a maintenance mode
  toggles it writable for permanent changes like `apt install`.

Everything the devices run is a plain Debian package from the project's
apt archive, and the role a device plays at boot — kiosk, media player,
developer console — is selected in the wizard.

## Getting started

1. **[Which device do I have?](devices/index.md)** — identify your
   hardware and see what works on it.
2. **[Install](install/index.md)** — how the browser-based install works,
   with per-device quickstarts for the [TC8](install/tc8.md) and the
   [Trio C60](install/c60.md).
3. **[Use](use/access.md)** — getting into an installed device,
   configuration, and the available [applications](use/applications/index.md).
4. **[Build](build/index.md)** — building the firmware from source, and a
   map of which repository does what.

No build is needed just to use a device: the wizard ships the release
artifacts. Building from source is only for development and customization.

## Source

All source lives in the
[Polycom-Open-Firmware](https://github.com/Polycom-Open-Firmware) GitHub
organization. See [Contributing](project/contributing.md) for the repo map
and [License](project/license.md) for licensing.
