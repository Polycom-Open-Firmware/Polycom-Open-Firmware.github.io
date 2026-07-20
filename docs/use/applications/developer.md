# Developer

`PROFILE=dev` is the "no application" role: the device boots to a plain
Linux console and nothing else runs — no kiosk lock, no fullscreen
client.

## Behavior

With this role selected the device boots to `multi-user.target` with:

- **tty1 autologin** — a root shell on the panel itself;
- **SSH enabled** — the per-board `poly-<device>-profile-dev`
  metapackage pulls `openssh-server`, so remote access is always
  present.

Every image already carries developer access (SSH and a root shell —
see [getting in](../access.md)); this role simply makes the console the
whole boot target instead of hiding it behind a kiosk application.

## What it is for

Using the panel as a generic Debian arm64 box: development, hardware
experiments, or running software the packaged roles do not cover. The
[storage model](../storage.md) applies as on every role — sealed OS,
persistent `/root`.

Role selection semantics (`PROFILE`): [configuration reference](../configure.md).
