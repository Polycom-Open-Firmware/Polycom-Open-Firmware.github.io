# Storage model

Two rules cover almost everything about where data lives on an installed
panel:

1. **The OS is sealed.** The root filesystem mounts read-only behind a
   tmpfs overlay. Everything *looks* writable, but changes evaporate on
   reboot — a panel always comes back pristine, and yanking power can't
   corrupt the OS. `tc8-mode` shows the current state.
2. **`/root` is durable.** Root's home lives on a spare 1 GiB eMMC
   partition (`facres`) that nothing else touches: it survives reboots,
   reseals, **and full wipe-and-reinstalls** from the wizard. Keep
   `authorized_keys`, dotfiles, notes, and scratch files there.

!!! note "Scope"
    The sealed-overlay model describes the TC8. The C60 mounts its
    rootfs read-write directly from `system_a` — no overlay, so writes
    are permanent as on ordinary Linux.

## The two modes

| | mount of the rootfs | writes | how you get there |
|---|---|---|---|
| **sealed** (default) | read-only + tmpfs overlay on `/` | ephemeral — evaporate on reboot | every boot, unless armed |
| **maintenance** | direct read-write, no overlay | permanent (dpkg-safe) | `tc8-rw` … `tc8-ro` |

`/root` is persistent in **both** modes.

## Making permanent changes (installing packages)

To change the OS itself — `apt install`, editing baked config — drop
into **maintenance mode**:

```sh
tc8-rw && reboot        # next boot: rootfs direct-rw, no overlay
# … log back in (a banner reminds you writes are now permanent) …
apt update && apt install <package>
tc8-ro && reboot        # reseal; your changes are baked in
```

The flag is sticky across reboots (safe for installs that need a
restart), so remember the `tc8-ro`. `tc8-mode` shows the current and
next-boot mode, and every interactive login warns while unsealed.

!!! warning "Never write to the sealed partition directly"
    Never write to the underlying partition while sealed (for example by
    remounting the overlay's lower layer read-write). Files already
    copied up to the tmpfs shadow the lower layer, and dpkg state would
    tear between them. The reboot flow exists because it is the only
    dpkg-safe way.

Running `apt` in sealed mode is not dangerous — database and payload
both land in the tmpfs and evaporate together — just pointless, and
`apt update` alone can eat around 200 MB of tmpfs.

Besides Debian's archive, every image trusts the **OpenPolycom package
archive** out of the box (`poly-*` packages: device profiles and
first-party apps) — `apt install poly-tc8-profile-kiosk` works in
maintenance mode with no extra setup. Archive conventions:
[the apt archive](../extend/apt-archive.md).

## What persists across what

| data | reboot (sealed) | reseal cycle | reflash / reinstall |
|---|---|---|---|
| `/root` (and `/persist/media`) | persists | persists | persists |
| changes made in maintenance mode | persists | persists | wiped |
| changes made in sealed mode | wiped | wiped | wiped |
| the maintenance-mode flag | persists (sticky) | cleared by `tc8-ro` | persists — run `tc8-ro` |
| device configuration applied from a config blob | persists | persists | wiped — push a blob with the reinstall |

Configuration pushed via the config blob is written to the real
filesystem, so it survives sealed reboots like any baked file — see the
[configuration reference](configure.md).

## Design depth

The boot flow, the initramfs that assembles the overlay, the `facres`
persistence service, and the full failure-mode table live in
[Sealed rootfs](../internals/ro-root.md).
