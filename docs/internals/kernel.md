# Kernel

Both devices run mainline **Linux 6.6** — a vanilla upstream tree plus
a per-device patch series that applies cleanly with `git am`. There is
no vendor kernel anywhere in the stack: the patches bring up each
board's hardware on stable mainline, and the build composes a shared
base config with a per-target fragment.

The series lives in
[`kernel-patches` in poly-firmware-build](https://github.com/Polycom-Open-Firmware/poly-firmware-build/blob/main/kernel-patches/README.md),
one directory per device; the per-patch changelog stays there. This
page is the summary of what each group enables.

!!! note "Size constraint"
    The TC8 kernel `Image` must stay under 32 MiB — the stock U-Boot
    2018.03 `BOOTM_LEN` cap ([TC8 boot chain](tc8-boot.md)). Release CI
    enforces it. The C60 boots through its own U-Boot and is not bound
    by this.

## Shared platform work

The two boards are sisters — the same i.MX 8M Mini SoC and much of the
same peripheral silicon — so the core enablement patches serve both:

- **RTL8363NB-VB switch support** (`net/dsa/realtek`). Both boards
  wire the SoC's FEC1 RGMII link to a Realtek RTL8363NB-VB 3-port
  gigabit switch. The chip is an electrical sibling of the supported
  RTL8365MB-VC but needs a substantially different bring-up: a
  register replay as a precondition for PHY access, an 8051 microcode
  upload via `request_firmware()`, and a PHY patch table sequenced
  after it. The result is one user port (`lan`) plus a CPU port under
  mainline DSA.
- **FEC fixed-phy DSA conduit** (`net/fec`). The MAC-to-switch link
  has no PHY — it is a fixed 1G/full RGMII connection. The patch
  registers a fixed-phy on the fly when no PHY device is on the bus,
  and avoids the FEC hard-reset path, which silently kills MDIO on
  these boards.
- **TAS5751M amplifier support** (`ASoC/tas571x`). The speaker amp on
  both boards. Register-compatible with the TAS5717 but requiring a
  board-specific coefficient sequence; the generic path mishandles it
  in ways that hard-hang the SoC on non-silent output.

## TC8 series

On top of the shared work, the TC8 series adds:

- **Board device tree** — `imx8mm-tc8.dts`: quad Cortex-A53, eMMC,
  the FEC/DSA network path, TAS5751M on I²S, Goodix GT9110
  multi-touch, and the panel, with all pin-mux and clocks.
- **LCC MIPI-DSI panel driver** (`drm/panel`) — the TC8's 800×1280
  4-lane panel, forked from the upstream Raydium RM68200 driver.
- **Panel-orientation plumbing** (`drm/mxsfb`) — propagates the
  panel's orientation to the DRM connector so the portrait glass
  presents correctly on a landscape device.
- **Brownout-safe volume cap** (`ASoC/tas571x`) — caps output volume
  so peak audio cannot brown out the panel's supply.
- **USB gadget mode** (DT) — pins `usbotg1` to peripheral mode with no
  role switch, for the USB gadget access path.

## C60 series

The C60 series shares the switch, FEC, and TAS5751M patches, and adds
its own board bring-up:

- **Board device tree** — `imx8mm-kepler-proto1.dts`, sister to the
  TC8's.
- **Microphone array** (`ASoC/tlv320adc3xxx`, four patches) — three
  TLV320ADC3101 mic ADCs share one TDM bus: `set_tdm_slot` support so
  each chip drives its assigned slots, a `-secondary` compatible (plus
  DT binding) for the clock-consumer chips, and PLL-from-BCLK support
  for a board that routes no MCLK to the ADCs.
- **Audio machine driver** — `poly,c60-audio`: one sound card exposing
  both directions of SAI1 (speaker out via TAS5751M, mic array in) as
  two DAI links sharing the single SAI.
- **LED controller** (`leds/lp5569`) — the light-bar / mute-ring is
  three TI LP5569 9-channel controllers; mainline gained this driver
  only after 6.6, so the series carries a version that builds against
  this tree.
- **Presence sensor** (`Input/digipyro`) — the Excelitas PYD1588
  pyroelectric (PIR) sensor, bit-banged over two GPIOs and used as a
  wake-on-motion interrupt.
- **Panel variant** (`drm/panel/rm67191`) — the C60's
  RM67191-command-compatible 720×1280 LCD differs from the 1080×1920
  AMOLED the mainline driver targets; the patch retargets the driver
  for the C60 glass.

## How the patches are applied

`build.sh` fetches vanilla `v6.6` and applies
`kernel-patches/patches/<target>/` with a reset-then-apply scheme, so
re-runs are idempotent and the tree under build is always exactly
base-plus-series. The kernel config is `kernel/config.base` merged
with `kernel/targets/<target>.frag`.
