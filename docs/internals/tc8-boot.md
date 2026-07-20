# TC8 boot chain

How a TC8 gets from power-on to Debian. The short version: the fused,
signed stock bootloader is left exactly where it is, and everything the
project controls hangs off a single chainload out of it.

The canonical specification — chainload mechanics, anti-brick
guarantees, the gesture selector, netboot, env handling — is
[`UNLOCK_SPEC.md` in the polycom-uboot repo](https://github.com/Polycom-Open-Firmware/polycom-uboot/blob/main/UNLOCK_SPEC.md).
This page is the map.

## Stage-1: HAB-closed, untouchable

The TC8's i.MX 8M Mini has **HAB closed**: Polycom's signing key is
burned into the SoC fuses, and the BootROM only runs a stage-1
bootloader signed with it. That stage-1 — stock U-Boot 2018.03 in the
eMMC `boot0` hardware partition — can neither be replaced nor modified,
and nothing can be signed for it.

This constraint shapes the whole design, and it also provides the
safety net: `boot0` is **never written**, so no flashing mistake can
brick the unit. HAB gates stage-1 only; everything after it is
modifiable. The BootROM's SDP USB-recovery mode (driven by NXP `uuu`)
remains available regardless of anything stored on eMMC.

## Stage-2: the chainload

Instead of replacing stock U-Boot, the boot path **chainloads** an
unsigned stage-2 — a current U-Boot 2024.04 built from the NXP
`uboot-imx` tree — resident in the eMMC `boot1` hardware partition,
outside the user area.

The hook is the stock bootloader's environment: it is rewritable and
protected only by a CRC32, not by HAB. Enrollment sets a single
variable — `bootcmd` — so that stock U-Boot reads stage-2 out of
`boot1` into DRAM, tears down caches, and jumps to it:

```
stock signed stage-1 (boot0)
  └─ persisted env bootcmd: mmc read from boot1 → go
       └─ stage-2 U-Boot (boot1):
            1. gesture selector  — 5 fingers → SDP, 4 → fastboot
            2. DHCP-66 netboot   — TFTP boot if the DHCP server offers it
            3. local boot        — boota, active A/B slot
            4. fallback          — fastboot gadget for provisioning
```

Precedence is gesture > netboot > local slot > fallback: on a
button-less chassis the touch gesture is sampled before anything
network- or env-driven can run, so a bad OS image or a rogue DHCP
server can always be overridden by hand.

## The stock env block

The stock environment lives at eMMC byte offset `0x400000` — an
unpartitioned gap between the signed bootloader region and the first
GPT partition. Besides the chainload `bootcmd`, it carries per-device
factory identity, most importantly `ethaddr` (Polycom OUI `00:e0:db`,
matching the device certificate).

Stage-2 treats that block as shared stock state and handles it
conservatively:

- **It keeps its own environment elsewhere** (offset `0x700000`), so a
  stage-2 `saveenv` can never touch the stock block.
- **The block is never imported or rewritten wholesale.** Importing it
  would adopt the stock `bootcmd` — the chainload itself — and loop the
  boot. Enrollment changes `bootcmd` through the stock loader's own
  `setenv`/`saveenv`, leaving every other variable intact.
- **Single variables are adopted read-only.** At boot, stage-2 scans
  the block for exactly the factory `ethaddr` and adopts that one
  value, so the panel keeps its factory MAC without the stock env ever
  being written.

## Unlocked `boota` and the unsigned-AVB rule

Stage-2 boots the OS with NXP's `boota` — the established Android boot
path — with the AVB lock state forced to *unlocked*. `boota` reads the
active slot from the `misc` bootctrl block and boots that slot's
`boot_<slot>` + `dtbo_<slot>` + `vbmeta_<slot>`; `fastboot set_active`
flips slots.

The slot images are Android-format: `boot.img` (kernel + the
[sealed-rootfs initramfs](ro-root.md) + the Debian cmdline), `dtbo.img`
(the device tree in an Android DTBO container), and `vbmeta.img`. They
carry AVB metadata, but it is **unsigned** (`avbtool --algorithm
NONE`).

!!! note "Unsigned is not the same as absent"
    `boota` rejects an image with **no** vbmeta as `INVALID_METADATA`,
    even when the bootloader is unlocked. The unlock only forgives a
    *missing or mismatched signature* — not absent metadata. The build
    therefore always generates structurally-valid-but-unsigned AVB:
    hash footers on `boot.img` and `dtbo.img`, descriptors bundled into
    `vbmeta.img`. The images boot only because the unit is unlocked.

## On-eMMC layout

Nothing of the project's sits in the user-area GPT — the stock A/B
partition table is reused as-is, with no repartitioning:

| Where | Contents |
|---|---|
| `boot0` (HW partition) | stock signed stage-1. Never written; keeps the unit SDP-recoverable. |
| `boot1` (HW partition) | the stage-2 U-Boot. Written once at enrollment; field-updated through the [`cache` blob](config-partition.md). |
| `0x400000` (user area, unpartitioned gap) | stock env block: chainload `bootcmd`, factory `ethaddr`. |
| `boot_a/b`, `dtbo_a/b`, `vbmeta_a/b` | the slot images. Slot `a` is the default install target; slot `b` can retain stock Android. |
| `userdata` | the Debian rootfs (`root=PARTLABEL=userdata`), sparse-flashed. |
| `cache` | one-shot config and bootloader-update blobs ([details](config-partition.md)). |
| `facres` | persistent `/root`, untouched by reinstall ([sealed rootfs](ro-root.md)). |

Keeping stage-2 in a boot hardware partition means the entire user-area
GPT belongs to the OS: the installer never has to route around a
reserved gap, and a `flashos` re-run can rewrite any slot freely.

## Related pages

- [Sealed rootfs](ro-root.md) — what the initramfs inside `boot.img` does.
- [Config partition](config-partition.md) — post-install config and stage-2 updates, no serial required.
- [C60 boot chain](c60-boot.md) — the HAB-open sibling, which boots differently.
- [Releases and delivery](releases.md) — how the slot images reach the provisioner.
