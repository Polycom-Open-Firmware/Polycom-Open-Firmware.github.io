# U-Boot

[`polycom-uboot`](https://github.com/Polycom-Open-Firmware/polycom-uboot)
builds mainline-based U-Boot for the Polycom i.MX 8M Mini devices:

| Build target | Device | Codename | DRAM |
|---|---|---|---|
| `c60-kepler_proto1` | Trio C60 | kepler_proto1 | 2 GiB LPDDR4 |
| `tc8-chainload-uboot` | TC8 conference tablet | proline_exec | 2 GiB LPDDR4 |

Both boards use the same i.MX 8M Mini Quad SoC and share DDR training
configs, but each has its own board file, device tree, and quirks. **The
two targets are delivered by different mechanisms** because their BootROM
lock state differs.

## Delivery per device

### C60 — HAB open: replace stage-1 directly

The C60's secure-boot (HAB) fuses are open, so its stage-1 bootloader can
be replaced outright:

- **SDP rescue path.** With the chassis BOOT_MODE switches set to
  USB-download (BMOD=00), `uuu` on the host loads this U-Boot into RAM
  over USB Serial Download Protocol. The build controls `bootcmd`, so the
  chain is hands-off: power-on → SDP enumeration → uuu loads → U-Boot
  autoboots Linux from slot A or B.
- **No fastboot trap.** Stock U-Boot enters fastboot mode whenever the
  BootROM reports a USB boot; this build skips that override, so the
  SDP → U-Boot → fastboot loop never traps.

### TC8 — HAB closed: chainload a stage-2 from eMMC boot1

The TC8 ships **HAB-closed** (SRK fuses burned), so the BootROM rejects
any unsigned stage-1 — there is no Polycom key with which to replace it.
HAB gates stage-1 only, so the project runs an unsigned **U-Boot 2024.04
as a chainloaded stage-2**:

- **Stage-1** is the stock signed bootloader in the eMMC `boot0` hardware
  partition. Its CRC-only, unsigned environment `bootcmd` is rewritten to
  chainload stage-2, so the chain persists automatically.
- **Stage-2** is the project's U-Boot 2024.04 in the eMMC `boot1`
  hardware partition (outside the GPT, so the stock GPT and factory
  partitions stay intact). Stage-1 reads it into RAM and jumps to it.
- **Boot method: unlocked NXP FSL Android `boota`.** AVB is forced
  unlocked, so one path boots unsigned Android-format images — stock
  Android in one GPT slot, Linux in the other, switched with
  `fastboot set_active`.
- **Boot UX.** A bootsel logo plus a touch-gesture window: a 4-finger
  gesture drops to fastboot (the Setup Wizard's entry point), 5 fingers
  selects SDP/UUU, no touch boots normally.
- **Provisioning** is the browser WebUSB/WebSerial wizard: *enroll*
  installs stage-2 into boot1, *flashos* flashes the OS slot image.

The full boot-chain narratives are on the
[TC8 boot chain](../internals/tc8-boot.md) and
[C60 boot chain](../internals/c60-boot.md) pages.

## Build

```sh
./scripts/build.sh c60-kepler_proto1
./scripts/build.sh tc8-chainload-uboot
```

## Flashing / loading

The C60 artifact is SDP-loadable — put the board in USB-download mode and
load it with `uuu`:

```sh
./scripts/flash-via-uuu.sh out/c60-kepler_proto1/flash.bin
```

The TC8 stage-2 artifact is `vendored/uboot-imx/u-boot.bin` (no-SPL
U-Boot proper); it is installed into eMMC boot1 by the WebUSB wizard, not
flashed via SDP.

## Repo layout

```
targets/
  c60-kepler_proto1/        Polycom Trio C60
    board/                  lpddr4_timing.c + lpddr4_timing.h (DDR table)
    dts/                    u-boot-side DTS
    uboot-overlay/          files layered over vendored uboot-imx
                            (board logic, defconfig, DTS, spl.c/board.c hooks)
    target.env              build vars
  tc8-chainload-uboot/      Polycom TC8 (proline_exec) — chainloaded stage-2
    board/                  lpddr4_timing.c (DDR shared with C60)
    uboot-overlay/          files layered over vendored uboot-imx
                            (defconfig, DTS, board hooks, fb_fsl boota fixes)
scripts/
  build.sh                  ./scripts/build.sh <target>
  flash-via-uuu.sh          load to a board in SDP mode
vendored/                   nxp-imx/uboot-imx, arm-trusted-firmware,
                            imx-mkimage, firmware-imx blobs (gitignored)
out/                        per-target build artifacts (gitignored)
```

## Deep engineering docs

The full engineering detail stays in the repo:

- TC8: [`UNLOCK_SPEC.md`](https://github.com/Polycom-Open-Firmware/polycom-uboot/blob/main/UNLOCK_SPEC.md)
  — the stage-2 spec (chainload, gesture selector, boota fixes,
  anti-brick);
  [DDR table provenance](https://github.com/Polycom-Open-Firmware/polycom-uboot/blob/main/targets/tc8-chainload-uboot/board/README.md)
- C60: [`BOOT_RECIPES.md`](https://github.com/Polycom-Open-Firmware/polycom-uboot/blob/main/targets/c60-kepler_proto1/BOOT_RECIPES.md)
  — manual boot recipes + memory map;
  [the `c60-boot` dual-boot helper](https://github.com/Polycom-Open-Firmware/polycom-uboot/blob/main/scripts/c60-dualboot/README.md);
  [`PMIC-DISCREPANCIES.md`](https://github.com/Polycom-Open-Firmware/polycom-uboot/blob/main/targets/c60-kepler_proto1/PMIC-DISCREPANCIES.md)
  — BD71847 erratum
