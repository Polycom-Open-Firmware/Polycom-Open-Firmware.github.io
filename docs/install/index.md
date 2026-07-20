# How installation works

Everything installs through the
**[Setup Wizard](https://wizard.openpolycom.cc/)** — a web page. Plug the
device into a USB port, open the page in a Chromium-based browser, and
click through. Nothing to install on the computer, no drivers, no command
line: the wizard fetches the release artifacts itself and talks to the
device directly from the browser.

What the wizard can do:

- **Unlock** a device and land the open bootloader (first time only)
- **Install or reinstall the OS** — A/B Android slot images flashed over
  fastboot
- **Choose the application** the device runs at boot — kiosk, media
  player, developer console, and more
  ([choosing a role](../use/applications/index.md))
- **Configure** a device with no shell — hostname, kiosk page, Wi-Fi,
  passwords, time zone, certificates
- **Update the bootloader** in the field, with no serial cable

How it reaches each device:

| Device | First-time unlock | How the browser talks to it |
|---|---|---|
| **TC8** touch panel | one-time serial-adapter hookup | WebUSB (fastboot) + Web Serial |
| **Trio C60** | entirely in the browser (BOOT_MODE switches → SDP) | WebHID (SDP) → WebUSB (fastboot) |

## The three phases

### 1. Unlock — once per device

A fresh unit only runs the factory bootloader, so the first step lands the
project's bootloader on it. The two devices get there differently:

- **TC8** — the panel only starts bootloader code signed by Polycom; that
  check is burned into the chip and can't be changed. The install works
  with it rather than against it: the factory bootloader runs first,
  exactly as shipped, then hands off to the project's bootloader, which
  lives in a spare region of the panel's built-in storage. The factory
  bootloader is never overwritten, so a bad flash can't permanently brick
  the panel. Reaching a fresh panel requires hooking up a serial adapter
  **once**; the serial connection is needed only for enrollment. See the
  [TC8 quickstart](tc8.md) and, for the full mechanics, the
  [TC8 boot chain](../internals/tc8-boot.md).
- **Trio C60** — no adapter at all. The C60's chip accepts unsigned
  bootloader code, so the wizard loads the project's bootloader over
  plain USB (see [SDP below](#what-loading-u-boot-over-usb-means-sdp))
  and then persists it, replacing the stock bootloader directly. See the
  [C60 quickstart](c60.md) and the
  [C60 boot chain](../internals/c60-boot.md).

### 2. Flash — install the OS

Pick a release and click flash. The wizard speaks the fastboot protocol
over WebUSB and writes the Android-format slot images (`boot`, `dtbo`,
`vbmeta`) plus the Debian root filesystem as an Android **sparse** image —
the transfer moves roughly the used data, not the full multi-gigabyte
image. On the TC8 the rootfs lands in the `userdata` partition, and the
spare A/B slot can keep stock Android as a way back; on the C60 the rootfs
(`rootfs.simg`) is flashed to `system_a`, replacing the stock system.

### 3. Configure — role and settings

The wizard writes the device's application role and settings — kiosk page,
hostname, Wi-Fi, passwords, time zone, certificates — from a form; no
shell needed. Role packages are baked into the image, so first boot needs
no network. The settings travel as a config blob flashed to a spare
partition and consumed on the next boot: applied, then invalidated — it is
a message, not a store. Details:
[configuration reference](../use/configure.md) and
[config partition](../internals/config-partition.md).

## What "loading U-Boot over USB" means (SDP)

The C60's chip (an NXP i.MX 8M Mini) has a built-in USB recovery mode
called the **Serial Download Protocol (SDP)**. With the BOOT_MODE switches
set to USB download, the chip does not boot from its storage at all — it
sits on the USB port waiting to be handed a program. The wizard hands it
the project's U-Boot, which runs **from RAM**: nothing on the device's
storage changes at this point, and a power cycle returns the device to
whatever is on its flash. Persisting the bootloader is a separate,
explicit step in the wizard's flow. Because SDP lives in the chip's
mask ROM, it can never be lost — it doubles as the C60's
[recovery path](recovery.md).

## Browser and OS requirements

- **A Chromium-based browser**: Chrome or Edge. The wizard drives the
  device through the WebUSB, Web Serial, and WebHID browser APIs — Firefox
  and Safari do not implement them.
- **Any desktop OS the browser runs on.** No drivers to install: the
  devices present descriptors that make Windows bind its generic driver
  automatically, and the browser asks permission with a device-chooser
  prompt before touching anything.
- **A USB cable** from the computer to the device's USB data port.

## The native app (Tauri)

The same wizard also ships as a native desktop application for hosts and
devices the browser can't reach — for example a Windows machine that
refuses to bind a driver. Installers for macOS, Linux, and Windows are
attached to the
[provisioner releases](https://github.com/Polycom-Open-Firmware/provisioner/releases).
The web flavor is the default; reach for the native app only if the
browser path fails.

## After the first install

Getting back into the wizard never needs the serial cable again: a
four-finger tap during the boot window drops an installed device into
fastboot. See [Re-provisioning](reinstall.md), and
[Recovery](recovery.md) if something went wrong.
