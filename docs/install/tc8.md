# TC8 quickstart

!!! note "Scope"
    This page covers the TC8 touch panel. The Trio C60 installs
    differently (no serial adapter at any point) — see the
    [C60 quickstart](c60.md).

A fresh TC8 runs only the stock signed U-Boot, which won't enter fastboot
on its own. Installation therefore starts with a **one-time serial
bootstrap**: catch the stock U-Boot prompt over the panel's UART, force it
into fastboot, then let the [Setup Wizard](https://wizard.openpolycom.cc/)
do the rest (enroll, then flash the OS). This is needed **once per unit** —
an already-enrolled panel drops into fastboot with the four-finger gesture
at the boot selector, no serial required (see
[Re-provisioning](reinstall.md)).

No build is required — the wizard ships the release artifacts. You bring
the panel, a UART probe, and a USB cable.

For the boot-path rationale (chainloaded stage-2, the slot-image model),
see the [TC8 boot chain](../internals/tc8-boot.md).

## What you need

- **A TC8 panel** powered by PoE (or a TC8-compatible PoE injector).
- **A serial UART probe** on the panel's debug header — anything that
  exposes a serial port at 115200 8N1 (an FTDI cable, a CP2102 board,
  even a Pi GPIO-UART). Wire `probe-TX → panel-RX`,
  `probe-RX → panel-TX`, `GND → GND`. Used only to catch U-Boot this
  once.
- **A USB cable** from your computer to the panel's **micro-B data
  port** — the fastboot gadget (and the wizard's WebUSB connection) ride
  this port.
- **Chrome or Edge** on the computer (WebUSB — Firefox and Safari won't
  work).

## Step 1 — Catch U-Boot

The TC8 ships with `bootdelay=0`, so there is no window to tap a key
during boot the normal way. Instead, send Ctrl-C continuously through the
serial probe while the panel powers up. Open a terminal program
(`picocom`, `screen`, `minicom`) on the UART:

```sh
picocom -b 115200 /dev/ttyUSB0
```

Power-cycle the panel (unplug and replug PoE) and **hold down Ctrl-C
continuously** for about 10 seconds. You should land on the U-Boot prompt:

```
u-boot=>
```

## Step 2 — Force fastboot

Plug the panel's micro-B data port into your computer, then from the
U-Boot prompt drop into the fastboot gadget:

```
u-boot=> fastboot 0
```

The panel now enumerates as a fastboot device over USB. That is all the
serial console is needed for — leave the UART connected in case you want
to retry, but everything else happens in the browser.

## Step 3 — Install from the browser

Open the [Setup Wizard](https://wizard.openpolycom.cc/) in Chrome or Edge
and:

1. **Connect device** — pick the TC8 in the browser's device chooser.
2. **Enroll** (one-time) — lands the stage-2 U-Boot in a spare hardware
   region of the eMMC and sets the factory bootloader's boot command to
   chainload it. From now on the panel loads stage-2 on every boot and
   the serial cable is no longer needed.
3. **Flash the OS** — the wizard flashes the slot images
   (`boot_a`/`dtbo_a`/`vbmeta_a`), sparse-flashes the root filesystem to
   `userdata`, marks slot `a` active, and reboots into Debian.

The wizard pulls all artifacts from the release manifest — nothing is
staged by hand.

## What you should see

After the reboot (roughly 30–45 seconds) the panel comes up into the
fullscreen kiosk, and the serial console shows `tc8-kiosk login:`. Log in
with the default credentials — and change them; see
[Getting in](../use/access.md).

## After the first boot

Set a real password and (optionally) install an SSH key:

```sh
ssh root@<panel-ip>     # find the IP from your DHCP server or `ip neigh`
passwd                  # change the root password
mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys   # paste your pubkey, Ctrl-D
```

The panel is also reachable with no network at all, straight over the USB
cable — see [Getting in](../use/access.md). To point the kiosk at your own
page, edit `/etc/default/tc8-kiosk` and `systemctl restart kiosk` — full
options in [Kiosk](../use/applications/kiosk.md).

## Re-provisioning later

The serial cable is never needed again. Enter fastboot with the
**four-finger gesture** at the boot selector, then re-run the install (or
the configure / bootloader-update flows) from the browser. See
[Re-provisioning](reinstall.md).
