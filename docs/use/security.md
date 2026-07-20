# Security and hardening

The firmware's defaults favor serviceability on a trusted network over
lockdown. This page lists the sharp edges — what each risk is,
and how to mitigate it. Work through it before putting a device
on an untrusted network.

## Default credentials: `root` / `root`

**Risk.** Every image ships with the same well-known root login, valid
on the serial console, the USB-gadget network, and the wired LAN. An
unchanged panel on a reachable network is an open root shell.

**Mitigation.** Change the password and install an ssh key immediately —
[Getting in](access.md) is the canonical reference and has the
commands. The key lives in `/root` and persists across reinstalls; the
password lives in the OS filesystem, so a `passwd` run during a sealed
boot on the TC8 does not survive a reboot — set it in
[maintenance mode](storage.md) or via the config blob's
`ROOT_PASSWORD`. At fleet scale, set
`ROOT_PASSWORD` and `SSH_AUTHKEY` in the config blob at provisioning
time, or bake different credentials into the image at build time.

## Config blob stored plaintext at rest

**Risk.** The config blob on the `cache` partition is plaintext —
including `WIFI_PASSWORD`, `ROOT_PASSWORD`, and any other secrets it
carries. Any root user on the device can read it. After the blob is
consumed, only its 64-byte header is zeroed; the payload bytes remain on
the partition until overwritten. Encrypting the payload is out of scope
for the blob format.

**Mitigation.** Don't put anything in a blob you wouldn't accept
sitting on the device's disk. The blob travels over local USB fastboot,
not the network, and the trusted-fleet assumption is usually
acceptable — but treat physical access to a provisioned device as
access to every secret its blob carried, and rotate accordingly if a
device leaves your control.

## Fleet-shared SSH host keys

**Risk.** SSH host keys are generated at image build time, so every
panel flashed from one image has the **same** host keys. Host-key
verification then cannot distinguish one panel from another, and a key
extracted from any single device allows impersonating every device from
that image.

**Mitigation.** Keys are fresh per build, so exposure is limited to one
image's fleet. For per-panel keys, regenerate them on each device in
[maintenance mode](storage.md):

```sh
tc8-rw && reboot
rm /etc/ssh/ssh_host_* && ssh-keygen -A
tc8-ro && reboot
```

Regenerated keys live in the rootfs, so a re-flash restores the shared
baked keys — repeat after reinstalling. Keys that survive a re-flash
would have to live on the persistent partition, which the image does not
do.

## Kodi web server on port 8080, no authentication

**Risk.** On the [media player](applications/media-player.md) role,
Kodi's web server is enabled on port 8080 without authentication, for
remote control and screenshots — and it binds to **all** device
interfaces. Anyone on the same network can control playback and take
screenshots of what the panel shows.

**Mitigation.** Keep media-player devices on trusted networks. Kodi's
own settings can add credentials to or disable the web server; Kodi
settings live in the persistent Kodi home, so that change survives
reboots and re-flashes.

## Open serial console

**Risk.** The USB data port always presents a serial console with a
login prompt (and the board's UART is a serial console too). With
unchanged default credentials, anyone with physical access to the port
has a root shell.

**Mitigation.** Changing the credentials (above) closes the walk-up
root shell — the console then requires the real password. Beyond that,
physical access control is the mitigation: the console itself is a
serviceability feature and stays enabled.
