# Choosing a role

A device runs one **role** (also called an application or OS profile) —
what the panel is *for*: a locked web kiosk, a Kodi media player, a
developer console. The role is picked in the
[Setup Wizard](https://wizard.openpolycom.cc/) at install time and decides
what the device boots into.

## What a role is

A role is a thin Debian **metapackage** (`poly-app-<id>`, with per-board
`poly-<device>-profile-<id>` packages where the role needs board hardware)
whose dependencies pull in the applications and services for that role.
Shipped releases carry a single root filesystem — the default-profile
`rootfs.simg`. The picked role travels as the `PROFILE` key in the
device's config blob, and the OS switches to that role on first boot —
nothing is downloaded by the device at runtime. The blob also carries
the role's settings (kiosk URL, media source, …); key semantics live in
the [configuration reference](../configure.md).

For self-built images, the build can also bake a role directly into the
rootfs: `--os-profile=` produces a per-role `rootfs-<name>` variant.

## Switching roles

To switch, run the wizard again (or push over fastboot) and pick the
new role — a reflash takes a few minutes over fastboot. Switching is
low-cost
because of the storage model:

- **`/root` persists** on its own partition across reboots *and*
  reflashes — SSH keys, dotfiles, and files stored there survive a role
  change.
- **Configuration is re-applied** from the config blob on the next boot,
  so device settings travel with the flash rather than living only on the
  old image.

See the [storage model](../storage.md) and
[re-provisioning](../../install/reinstall.md) pages.

## Available roles

| Role | TC8 | Trio C60 | Page |
|---|---|---|---|
| **Kiosk** — locked fullscreen browser (default) | ✅ | ✅ | [Kiosk](kiosk.md) |
| **Media player** — Kodi media center | ✅ | ✅ | [Media player](media-player.md) |
| **Smart speaker** — voice appliance (placeholder) | — | ✅ | [Smart speaker](smart-speaker.md) |
| **Developer** — console + SSH, no application | ✅ | ✅ | [Developer](developer.md) |

The smart-speaker role needs the C60's microphone array and speaker, so it
is offered only on that board. The build system also knows a special
`bare` profile — the untouched Debian base with no role package — which is
a build-time construct rather than a wizard choice.
