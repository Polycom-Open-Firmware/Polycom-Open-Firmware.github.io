# Debian rootfs

[`poly-rootfs`](https://github.com/Polycom-Open-Firmware/poly-rootfs)
builds the slim Debian bookworm arm64 rootfs for both Polycom panels
(same i.MX 8M Mini SoC): `--device=tc8` (default) or `--device=c60` picks
the board's profile metapackage.

This repo only produces the rootfs. The kernel comes from
`poly-kernel-patches`; the flashable boot artifacts are assembled by
[`poly-firmware-build`](firmware.md), which consumes this repo as a
submodule and packs the rootfs per target — the sparse `rootfs.simg`
flashed to `userdata` on the TC8, `rootfs.img.zst` for `system_a` on the
C60.

## What it builds

- `out/rootfs.tar.gz` — a minbase Debian bookworm arm64 chroot with the
  kiosk service stack (weston + cog/WPE), Hantro VPU userspace, baked SSH
  host keys, and the `/etc` overlay applied

## Build

```bash
sudo apt-get install debootstrap qemu-user-static binfmt-support \
                     gzip rsync
sudo ./build.sh
```

Output lands in `out/`.

The full form is
`sudo ./build.sh --profile=<name>[,<name>…] --device=<tc8|c60>`
(defaults: profile `kiosk`, device `tc8`): the builder debootstraps one
base, then per profile installs that role's metapackage from the
OpenPolycom archive into an isolated clone and emits
`out/rootfs-<name>.tar.gz`. The special profile `bare` is the untouched
base. The archive keyring and sources.list are baked into the base
(`etc/apt/sources.list.d/poly.list`), so image builds and on-device
maintenance installs resolve `poly-*` packages with no setup. The profile
model itself is covered in [Packages and profiles](../extend/index.md).

## Repo layout

```
build.sh            # host-side: debootstrap, chroot, overlay, tarball
chroot-setup.sh     # runs inside the chroot: apt + cleanup + enable units
package-list.txt    # one Debian package per line (comments OK)
etc/                # files copied verbatim into the rootfs at the same path
usr/                # ditto: kiosk-launch, the tc8-* sbin helpers,
                    #   tc8-bootctl.c, the baked archive keyring
out/                # build output (gitignored)
```

## What's installed

`package-list.txt` (42 packages, one per line with per-package rationale)
covers:

- Boot/init: systemd, systemd-sysv, systemd-resolved, systemd-timesyncd,
  udev, dbus, libnss-systemd, fake-hwclock
- Networking: iproute2, iputils-ping, isc-dhcp-client, openssh-server,
  wpasupplicant
- Wayland kiosk: weston (the kiosk-shell compositor), cog launcher,
  WPE WebKit + libwpe-fdo, cage, xwayland
- GPU + input: seatd, libinput-bin, libegl1, libgles2, mesa-utils
- Audio: alsa-utils
- Hantro VPU: gstreamer1.0-{plugins-base,plugins-good,plugins-bad,libav},
  v4l-utils
- U-Boot env access from Linux: u-boot-tools, libubootenv-tool
- Minimal utils: util-linux, e2fsprogs, psmisc, procps, less,
  curl, ca-certificates, busybox-static, locales

`--no-install-recommends` everywhere; `/usr/share/doc`, `/usr/share/man`,
and non-`en` locales are stripped via `dpkg path-exclude`. The final
rootfs tarball is ~280–320 MiB compressed.

## Baked configuration

Per-build defaults live in `etc/`:

- `etc/default/tc8-kiosk` — `KIOSK_URL`, `KIOSK_ENGINE`, `COG_OPTS`,
  `CHROMIUM_OPTS`
- `etc/tc8-kiosk/weston.ini` — compositor config: kiosk-shell,
  `xwayland=false`, output rotation via `transform=rotate-90`
- `etc/systemd/system/{kiosk,kiosk-vt}.service` — the kiosk-on-tty7
  service (runs `kiosk-launch`, which starts weston with kiosk-shell and
  then the browser selected by `KIOSK_ENGINE`: cog by default, chromium
  when installed) plus an explicit `chvt 7` helper
- `etc/udev/rules.d/{50-drm,70-seat}.rules` — gives the `kiosk` user
  group access to `/dev/dri/*` and seat-tags `/dev/input/event*` so
  libinput finds the Goodix touchscreen
- `etc/environment.d/99-vpu.conf` — biases GStreamer toward the v4l2
  stateless decoders (Hantro G1/G2) over libav

Per-device values are **not** baked: the wizard writes a config blob to
the `cache` partition and `etc/tc8-config/apply-config.sh` applies it at
boot — see the [configuration reference](../use/configure.md).

On the flashed image the rootfs runs sealed (read-only behind a tmpfs
overlay) with a persistent `/root` and a maintenance mode for permanent
changes — that model is described in the
[storage model](../use/storage.md) page.

## SSH host keys

`chroot-setup.sh` runs `ssh-keygen -A` inside the chroot, so every build
gets a fresh, unique set of host keys; keys are never committed. The
baked keys are shared by every panel flashed from one image.
