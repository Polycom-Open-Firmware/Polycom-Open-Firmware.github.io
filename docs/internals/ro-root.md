# Sealed rootfs

The TC8 rootfs mounts read-only behind a tmpfs overlay: every boot
starts pristine, and pulling power cannot corrupt the OS. This page
covers the mechanism; the operator's view (modes, making permanent
changes) is on [Storage model](../use/storage.md).

!!! note "Scope"
    TC8 only. The C60 mounts its rootfs read-write directly from
    `system_a` — no initramfs, no overlay
    ([C60 boot chain](c60-boot.md)).

## The two modes

| | mount of `userdata` | writes | how you get there |
|---|---|---|---|
| **sealed** (default) | read-only lower + tmpfs overlay on `/` | ephemeral — evaporate on reboot | every boot, unless armed |
| **maintenance** | direct read-write, no overlay | permanent (dpkg-safe) | `tc8-rw --reboot` … `tc8-ro --reboot` |

`/root` is persistent in **both** modes — bind-mounted onto the
`facres` partition, which survives reboots *and* reflashes. Everything
else in sealed mode is ephemeral by design.

## Where the initramfs lives

The overlay is assembled by a minimal initramfs: a ~1 MB gzipped cpio
holding a static busybox and one auditable POSIX-sh script. The
production `boot.img` embeds it as the **Android boot-image ramdisk**
(`mkbootimg --ramdisk`) — the kernel command line never references it,
because Android boot images deliver the ramdisk through the image
header, not the cmdline. At boot, stage-2 `boota` hands the ramdisk to
`booti` as `addr:size` (relocating it past the FDT first, since the v0
header's stock `ramdisk_offset` would overlap the kernel).

The development paths carry no ramdisk: the `nfs` netboot profile and
raw-`booti` boots driven from the U-Boot console use the profile's
standalone `KERNEL_CMDLINE` and pass no ramdisk at all, so they run
without the initramfs and mount the rootfs read-write with no overlay.
The cmdline's `root=PARTLABEL=userdata rw` serves double duty: the
initramfs parses it, and it keeps the kernel's own direct-mount
fallback correct whenever the ramdisk is absent.

## Design properties

- **All of `/` is sealed** — one overlay over the whole rootfs, not
  per-directory tmpfs mounts, so nothing is accidentally left writable.
- **The mode decision happens before any filesystem is mounted
  read-write.** This is what makes maintenance mode dpkg-safe: in
  direct-rw there is no overlay at all, so dpkg's database and its
  payload land on the same filesystem and cannot tear apart.
- **systemd sees an ordinary writable `/`** — no special ordering
  around `systemd-remount-fs`, `tmpfiles`, or `machine-id`.
- The script fails **towards booting** (see failure modes): a broken
  overlay degrades to a direct-rw boot, never to a dead panel.

## Boot flow

```
stage-2 boota ─ AVB(NONE, unlocked) ─ booti Image + initramfs + dtb
  └─ /init (busybox):
       1. mount /proc /sys /dev(devtmpfs)
       2. parse cmdline: root=PARTLABEL=… (default userdata),
          tc8.rootfs=ro|rw override, tc8.overlay_size=… (default 50%)
       3. wait ≤20 s for the rootfs partition to appear
       4. mode: cmdline override, else the flag file at the root of
          the ext4 facres partition (mounted ro, then umounted)
       5a. sealed:   mount userdata RO at /lower, tmpfs at /rw,
                     overlay(lower,upper,work) → /newroot;
                     layers exposed at /mnt/.tc8/{lower,rw}
       5b. maintenance: mount userdata RW at /newroot (no overlay)
       6. move devtmpfs, exec switch_root /newroot /sbin/init
  └─ systemd boots normally on a writable /
       └─ tc8-persist-root.service (before config/ssh/kiosk):
            facres → /persist (auto-mkfs.ext4 on first use),
            /persist/tc8-root → bind → /root,
            /persist/fake-hwclock.data → file-bind → /etc/fake-hwclock.data
```

The maintenance flag (`/persist/.tc8-rootfs-rw`, written by `tc8-rw`)
is **sticky**: it survives reboots — multi-reboot maintenance sessions
work — and even reflashes; `tc8-ro` is an explicit step. The cmdline
override (`tc8.rootfs=ro|rw`) beats the flag. The initramfs only ever
mounts `facres` read-only to read it.

## Persistent /root

`tc8-persist-root.service` runs in both modes, so `/root` behaves
identically either way: `facres` (1 GiB, untouched by the wizard's
`flashos`) mounts read-write at `/persist`, is seeded once from the
baked `/root`, and bind-mounts onto `/root`. Wizard-pushed SSH keys
persist because the config applier runs after it. The saved
fake-hwclock timestamp lives there too — otherwise every sealed boot
would start at image-build time and TLS would fail until NTP.

## Interactions

- **Config blob** — consumed *pre-seal* by the initramfs: a staged
  blob is applied to the real rootfs (mounted rw for seconds) and
  invalidated in place before the overlay is assembled, so the applied
  state is simply in the filesystem. The runtime config service gates
  itself out on overlay boots and serves direct-rw boots and
  no-initramfs targets (the C60). Details:
  [Config partition](config-partition.md).
- **`/data` (kiosk cache, MTP export)** — `/data` is the `userdata`
  partition mounted a second time. In sealed mode that rw mount fails
  (the superblock is held read-only by the overlay lower); the kiosk
  unit tolerates the failure and creates the directories on the
  overlay instead, so kiosk cache and MTP "Panel storage" are
  ephemeral in sealed mode, persistent in maintenance mode.
- **Audio** — `alsa-restore` reads the baked safe-volume state from
  the read-only lower every boot; runtime mixer changes evaporate.
  Volumes are re-assertable per boot via the config blob's `VOLUME_*`
  keys.
- **SSH** — host keys are baked at build time (lower layer), stable
  across reboots.
- **DHCP** — leases live on the upper: a fresh DHCPDISCOVER every
  boot, by design.
- **Journald** — `Storage=volatile` plus a tmpfs `/var/log`; no
  persistent journal is created anywhere.

## Failure modes

| Failure | Behaviour |
|---|---|
| overlay/tmpfs mount fails (e.g. a kernel without `OVERLAY_FS=y`) | init logs the failure and **falls back to direct-rw** — the panel still boots |
| ramdisk absent from `boot.img` | kernel falls back to the cmdline `root=…` rw direct-mount — boots direct-rw, no overlay |
| `/init` crashes / rootfs partition never appears | rescue busybox shell on console; `exit` reboots (`panic=10` guards the no-console case) |
| `facres` missing | no flag possible → always sealed; `/root` non-persistent; `tc8-rw` refuses with a clear error (the cmdline override still works) |
| `facres` corrupt / not ext4 | sealed boot proceeds; `facres` is **reformatted** (its content is expendable by design) and persistence resumes empty |
| tmpfs upper fills (default 50% of RAM, `tc8.overlay_size=` to tune) | writes get `ENOSPC`; the lower layer is unharmed; reboot clears |
| maintenance flag forgotten | sticky by design; the login banner and `tc8-mode` surface it — run `tc8-ro` |
| power loss during maintenance apt | same risk as any rw Linux — the ext4 journal replays; worst case, reflash `userdata`. The sealed default makes this window rare |

The full design document, including implementation notes on the
initramfs builder and the kernel-config requirement, is
[`docs/RO-ROOT.md` in poly-firmware-build](https://github.com/Polycom-Open-Firmware/poly-firmware-build/blob/main/docs/RO-ROOT.md).
