# Which device do I have?

OpenPolycom targets two devices from the Poly conference-room family. Both
are built on the NXP i.MX 8M Mini SoC, so they run the same kernel tree and
the same Debian stack; only the board facts (device tree, boot recipe,
partition layout) differ.

## Identifying the hardware

**[Polycom TC8](tc8.md)** — the small 8-inch touch panel that ships as the
room controller for Poly video systems. Rectangular tablet form factor on a
stand, a single Ethernet jack that supplies both power (PoE) and network,
and a USB-C data port. The model name is printed on the label on the back.

**[Polycom Trio C60](c60.md)** — the round conference phone from the same
family: a touchscreen in the middle of a three-legged speakerphone body,
with a microphone array, a speaker, Wi-Fi/Bluetooth, and HDMI input.

## Capability and status

| | **TC8** | **Trio C60** |
|---|---|---|
| SoC | i.MX 8M Mini (quad Cortex-A53) | i.MX 8M Mini (quad Cortex-A53) |
| RAM | 2 GiB LPDDR4 | 2 GiB LPDDR4 |
| Display + touch | Works | Works |
| Audio | Speaker (class-D amp) | Speaker + microphone array |
| Networking | Gigabit Ethernet (PoE) | Wi-Fi / Bluetooth |
| First-time unlock | One-time serial-adapter hookup, driven by the browser | All-browser: USB boot mode + WebHID, no adapter |
| Boot model | Factory bootloader chainloads the project's stage-2; A/B slot images | Project U-Boot boots Linux from `boot_a` |
| Where the OS lands | `userdata` partition (stock Android can stay in the spare slot) | `system_a` partition (replaces the stock system) |
| Support status | Supported; installable from tagged releases via the Setup Wizard | Experimental; C60 images appear in releases that include them |

## Status

**TC8 support is the more mature path.** The full stack — display, touch,
audio, networking — runs on mainline Linux and ships as tagged releases
that the [Setup Wizard](https://wizard.openpolycom.cc/) installs directly.

**Trio C60 support is experimental.** Hardware support covers display,
touch, audio in and out, LEDs, and Wi-Fi/BT, and the build system
produces the complete C60 image set; releases that include C60 images
list them in the release notes. Note also that the C60's smart-speaker
role boots to a console: no voice application package exists (see the
[Trio C60 page](c60.md)).

## Next steps

- [TC8 details](tc8.md) → [TC8 install quickstart](../install/tc8.md)
- [Trio C60 details](c60.md) → [C60 install quickstart](../install/c60.md)
