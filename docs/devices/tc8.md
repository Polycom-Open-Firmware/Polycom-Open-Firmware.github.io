# Polycom TC8

The TC8 is the 8-inch PoE touch panel that ships as the room controller
for Poly video systems. Under OpenPolycom it becomes a small Debian Linux
machine that boots straight into a fullscreen web kiosk: point it at a
dashboard, a calendar, a camera feed, or any other page. One Ethernet
cable supplies both power (PoE) and network.

## Hardware

- **SoC**: NXP i.MX 8M Mini, quad Cortex-A53, 2 GiB LPDDR4, 16 GB eMMC
- **Display**: 800×1280 MIPI-DSI panel with backlight; Vivante GC600/GC520
  GPU (etnaviv)
- **Touch**: Goodix GT9110 multi-touch controller
- **Audio**: TAS5751M class-D speaker amplifier
- **Networking**: RTL8363NB-VB Ethernet switch behind the SoC's FEC
  controller; gigabit full-duplex, PoE-powered
- **USB**: a data port that the firmware exposes as a composite gadget

## What works

Everything, on mainline-based Linux, shipping as tagged releases:

- Display and backlight with GPU acceleration — the kernel boot crawl and
  systemd status show on the panel, so a failing boot tells you where it
  died
- Multi-touch input (`/dev/input/event0`)
- Audio via the TAS5751M (`tas5751-audio` ALSA card). The volume range is
  hard-capped in the kernel so 100% is safe — full scale browns out the
  panel on loud content
- Ethernet with the panel's **factory MAC address**, recovered from the
  stock bootloader environment — stable DHCP leases out of the box
- Composite USB gadget on the data port: a serial console with a root
  login (`/dev/ttyACM0`), USB Ethernet (panel at `10.55.0.1`, ssh straight
  off the cable), and MTP exposing two "Portable Device" storages for
  drag-and-drop: the persistent `/root` and the
  [media folders](../use/applications/media-player.md)
- A **sealed root filesystem**: the OS mounts read-only behind a tmpfs
  overlay, so reboots always come up pristine; `tc8-rw`/`tc8-ro` toggle a
  maintenance mode for permanent changes
  ([storage model](../use/storage.md))
- **Persistent `/root`**: root's home lives on a spare eMMC partition and
  survives reboots *and* full reinstalls

By default the panel boots into a fullscreen Wayland kiosk; the kiosk URL
and other settings are set from the wizard, no shell needed
([configuration](../use/configure.md)).

## Kernel support

The kernel is vanilla mainline Linux 6.6 plus a small patch set that
enables the board: a device tree for the TC8, a driver for its MIPI-DSI
panel, RTL8363NB-VB support in the Realtek DSA switch driver (including
the switch's 8051 microcode load), a FEC tweak so the switch's conduit
link comes up reliably, and TAS5751M support in the TAS571x audio codec
driver. See [Kernel internals](../internals/kernel.md).

## How it boots

The TC8's BootROM only starts bootloader code signed by Polycom — that
check is burned into the chip (HAB secure boot) and cannot be changed. The
install works with it rather than against it: the factory bootloader runs
first, exactly as shipped, and then chainloads the project's U-Boot
(stage-2) from a spare region of the eMMC. Stage-2 boots Debian packaged
as an Android-format slot image — to the hardware, nothing unusual is
happening, and stock Android can stay in the spare slot as a way back. The
factory bootloader is never overwritten, so a bad flash cannot permanently
brick the panel. Full detail: [TC8 boot chain](../internals/tc8-boot.md).

## Install

First-time unlock needs a serial adapter, hooked up only once;
everything else happens over USB in the browser. Follow the
[TC8 install quickstart](../install/tc8.md).
