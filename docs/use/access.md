# Getting in

The image bakes in several ways to reach a shell on an installed device.

!!! note "Scope"
    The USB gadget examples below describe the TC8's micro-B data port.
    ssh over the wired LAN and the credential handling apply to both
    devices.

## USB composite gadget

Plugging the TC8's micro-B data port into a computer enumerates a
composite USB gadget with three interfaces:

- **Serial console (CDC ACM)** — appears as `/dev/ttyACM0` on Linux and
  as a "USB Serial Device" COM port on Windows. A getty spawns a login
  prompt automatically; open the port in any terminal program and log
  in.
- **USB Ethernet (CDC NCM)** — appears as a `usb0` network interface on
  the host. The panel runs a DHCP server on `10.55.0.1/24` and leases
  the host `10.55.0.2` (a single-lease pool). ssh to `10.55.0.1` the
  moment the link comes up:

  ```sh
  ssh root@10.55.0.1
  ```

- **File transfer (MTP)** — the persistent `/root` appears as a
  "Portable Device" store named "Root home (persistent)", served by
  uMTP-Responder. Drag and drop files in any native file manager; they
  land on the durable partition and survive reboots and reinstalls (see
  [Storage model](storage.md)). On the
  [media player](applications/media-player.md) role a second store,
  "Media", exposes the `/persist/media` library.

## ssh over the LAN

`sshd` listens on port 22 on the wired LAN. The panel uses its factory
MAC address, so a DHCP reservation set once stays valid across
reinstalls.

## Default credentials

!!! warning "Change the default credentials"
    Every image ships with the default login **`root` / `root`** — on
    the serial console, over ssh on the USB gadget, and over ssh on the
    LAN. Change them before putting the device on an untrusted network.

**ssh key** — install a public key:

```sh
mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys   # paste your pubkey, Ctrl-D
```

The key lands in `/root`, which on the TC8 persists across reboots
**and reinstalls** (see [Storage model](storage.md)) — install it once
and it stays.

**Password** — the password lives in `/etc/shadow` inside the OS, not
in `/root`. On the TC8's sealed root filesystem, a `passwd` run during
a normal (sealed) boot evaporates at the next reboot. Two durable ways
to set it:

- **`passwd` in [maintenance mode](storage.md)** — written to the real
  filesystem, so it survives reboots; a reinstall replaces the
  filesystem and resets it.
- **`ROOT_PASSWORD` in the config blob** (see the
  [configuration reference](configure.md)) — applied to the real
  filesystem before sealing, and reinstall flows include a blob, so the
  password is re-applied after a reinstall.

The C60 has no sealed overlay, so a plain `passwd` there persists
across reboots; a reinstall replaces the system, so use the config blob
to keep the password across reinstalls.

At fleet scale, the same can be done without a shell: the config blob's
`ROOT_PASSWORD` and `SSH_AUTHKEY` keys set the password and append an
authorized key at provisioning time (see the
[configuration reference](configure.md)). To bake different credentials
into the image at build time, see [Debian rootfs](../build/rootfs.md).

For the other sharp edges that follow from default credentials and open
consoles, see [Security and hardening](security.md).
