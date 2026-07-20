# Config partition

How the provisioning wizard pushes device configuration and stage-2
bootloader updates over fastboot — no serial connection, no bootloader
change. The wizard writes a blob to the stock `cache` GPT partition;
the device applies it on the next boot and invalidates it in place.

This page covers the mechanism. The configuration **keys** and their
semantics live on the [Configuration reference](../use/configure.md).
The byte-level wire format is a contract between the wizard and the
firmware and stays canonical in
[`CONFIG-PARTITION.md` in poly-firmware-build](https://github.com/Polycom-Open-Firmware/poly-firmware-build/blob/main/CONFIG-PARTITION.md).

## Why `cache`

- It is already in the stock Android GPT — 1 GiB of ext4 (formerly
  Android `/cache`), unused by the Debian image.
- `fastboot flash cache <blob>` works with the existing stage-2
  fastboot gadget — no bootloader rebuild, no re-enrollment.
- `cache` is not in the AVB-verified chain (`boot`/`dtbo`/`vbmeta`),
  so nothing needs re-signing.

The mechanism serves both devices: the wizard writes the same blob on
the TC8 and the C60, and the shared `apply-config` reader consumes it
on both.

## Layout, at a summary level

The blob is written to the start of the partition. Two independent
halves, each behind its own 64-byte header of magic + length + sha256:

| Offset | Contents |
|---|---|
| 0 | config header (`TC8CFGv1`), then the config payload — UTF-8 `KEY=value` lines, max 1 MiB |
| 1 MiB | optional bootloader header (`TC8BOOT1`); the stage-2 image starts at the next sector |

A config-only push simply omits everything from 1 MiB on. The device
verifies magic and sha256 before applying either half; a fresh, empty,
or half-written `cache` is ignored and the unit keeps its current
state.

## Consume-once semantics

The blob is a **one-shot message, not a store**:

- A present, valid config blob is applied exactly once — each unique
  blob is applied on the boot after it is flashed, then invalidated in
  place (its header is zeroed) as the applier's last step.
- The filesystem is the only store of device state; there is no
  snapshot or marker layer.
- **Invalidate-last makes the apply atomic**: a power cut mid-apply
  leaves the blob intact, and it re-applies cleanly on the next boot.
- Applying a blob is explicit intent — it overwrites whatever is
  configured on the device, including maintenance-mode edits.

Who applies it depends on the boot path: on sealed (overlay) TC8 boots
the [initramfs](ro-root.md) applies the blob to the real rootfs before
the overlay is assembled; on direct-rw boots and on targets without an
initramfs (the C60), the `tc8-config` boot service does the same
against the live filesystem.

## PROFILE → systemd target

The `PROFILE` key sets the device role by mapping to a systemd default
target; the role's applications are baked packages
(`poly-<device>-profile-<role>`) — nothing is fetched:

| `PROFILE` | Boot target | Behaviour |
|---|---|---|
| `kiosk` | `graphical.target` | fullscreen `kiosk.service` |
| `dev` | `multi-user.target` | tty1 autologin + ssh, no kiosk lock |
| `smart-speaker` (C60) | `multi-user.target` | enables the voice-app service if baked, else console |
| `media-player` | `graphical.target` | kiosk stack with `KIOSK_ENGINE=kodi` (falls back to the browser engine if Kodi is not baked) |
| *(omitted)* | — | role left untouched — a config-only push never resets the role |
| *(unknown value)* | `graphical.target` | treated as `kiosk` |

Everything else — identity, network, credentials, certificates, time,
audio — is in the [key reference](../use/configure.md).

## Bootloader updates through `cache`

The TC8's stage-2 U-Boot lives in the eMMC `boot1` hardware partition
([TC8 boot chain](tc8-boot.md)). A running Debian can rewrite `boot1`;
a fastboot session generally cannot target it cleanly. So the wizard
hands the stage-2 image to the OS through `cache`, and **the OS does
the write** — the wizard never writes `boot1` directly:

1. The wizard appends the `TC8BOOT1` half to the blob
   (`fastboot flash cache` + `fastboot reboot`); its job ends there.
2. At boot, the updater service validates the header — magic, sha256,
   and a stage-2 instruction-signature check — then compares the image
   to what is already in `boot1`.
3. Only if different: flash, then read-back verify.

The unit comes up on the *old* stage-2, flashes `boot1` in the
background, and the *next* power-cycle runs the new one; since the
write is idempotent, it converges on its own. Nothing in this flow can
brick the unit: `boot0` (the stock stage-1) is never touched, so SDP
recovery always works.

Install and reinstall flows include the current stage-2 in the default
blob, so every install lands the matching bootloader as part of the
same `cache` write — one extra fetch, not a separate device
round-trip.

## Security

The blob is plaintext at rest on `cache` — any root user on the device
can read it, including pushed passwords and keys. It travels over
local USB fastboot, not the network. Encrypting the payload to a
device or fleet key is out of scope for this format: nothing belongs
in a blob that would be unacceptable on the device's disk.
