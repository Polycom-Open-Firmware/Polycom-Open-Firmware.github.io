# Provisioner architecture

The provisioning wizard is one codebase with two flavors: a
zero-install **web** flavor — open a URL in Chrome or Edge, plug in,
flash — and a **native (Tauri)** flavor for devices the browser
physically cannot reach. Both run the same UI, the same flows, and the
same protocol code; only the bottom layer differs.

The canonical architecture document is
[`ARCHITECTURE.md` in the provisioner repo](https://github.com/Polycom-Open-Firmware/provisioner/blob/main/ARCHITECTURE.md).

## Why two flavors

The web flavor is the default, for devices that cooperate with the
browser: devices that present a WinUSB binding on Windows (TC8
fastboot) or a WebHID-reachable BootROM (the C60's SDP recovery —
[C60 boot chain](c60-boot.md)). No install, no driver ceremony.

The native flavor exists for the genuinely browser-proof cases:

| Blocker | Why the browser loses | Native answer |
|---|---|---|
| Windows driver binding | raw USB needs WinUSB bound to the device, and a browser cannot install a driver — unusable for devices without MS-OS descriptors | bundle a driver-install step and bind WinUSB to any device |
| DFU mode | the protocol works over WebUSB, but only if the DFU interface is WinUSB-bound — the binding is the wall | Rust USB + DFU control transfers |
| SDP / BootROM recovery | WebHID reaches the C60; other devices' HID usages may be blocked, and the Tauri webview itself has no WebHID | pure-Rust SDP loader |
| no fastboot at all (UMS-only, raw block, vendor protocols) | mostly unreachable | whatever the device speaks |
| serial signal control (DTR/RTS, BREAK) | Web Serial lacks full signal control on some platforms | full control via the `serialport` crate |
| offline/air-gapped or batch flashing | network fetch, one device at a time | signed binary, on-disk image library, multi-unit |

Rule of thumb: web where the browser APIs reach the device, native
where they cannot. Same wizard, same UI, two transports.

## The transport seam

The keystone: protocol code never touches `navigator.usb` (or any
platform USB, serial, or HID API) directly. Four layers, each
depending only on the one below through an interface:

```
┌──────────────────────────────────────────────────────────┐
│  UI / Wizard          renders a profile's steps, generic  │
├──────────────────────────────────────────────────────────┤
│  Flow + Profile       resumable step machine per device   │  unlock, reinstall, …
├──────────────────────────────────────────────────────────┤
│  Protocol             what the bytes mean                 │  fastboot, sdp,
│                       (injected a Transport)              │  uboot-console, sparse
├──────────────────────────────────────────────────────────┤
│  Transport            how bytes move       ← the swap point │
│   UsbTransport / SerialTransport / HidTransport            │
└──────────────────────────────────────────────────────────┘
        web adapter    → WebUSB / Web Serial / WebHID
        native adapter → Tauri invoke() → Rust nusb / serialport
```

- **Transport** is the only layer that differs between flavors: a thin
  interface (open/claim, bulk in/out, control transfer,
  list/describe). The web adapter wraps the browser APIs; the native
  adapter forwards over Tauri IPC to a Rust backend.
- **Protocol** modules — fastboot, SDP, sparse, U-Boot console — are
  written once against the transport interface.
- **Flow + Profile** is the wizard logic: an ordered, resumable list
  of steps (identify → preserve identity → flash → verify → reboot),
  each step a protocol operation plus an artifact. A device profile
  picks which protocols and steps apply — a new device is a new
  profile plus any new protocol modules, with no changes to the rest
  of the app. The C60 is exactly that: a profile plus the SDP module.
- **UI** renders a profile's steps generically and streams progress;
  it never imports a transport.

Because only the transport layer is swapped, the two flavors share
roughly 95% of the code.

## Where the bytes come from

The wizard fetches firmware through the release pipeline's same-origin
proxy — release lists and artifact streams alike. That contract, and
why a browser needs it, is covered in
[Releases and delivery](releases.md).
