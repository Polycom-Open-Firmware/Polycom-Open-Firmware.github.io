# Network boot

How to netboot the TC8 panel: U-Boot pulls a kernel and DTB over TFTP,
and the kernel mounts its rootfs over NFSv3. Result: the same fullscreen
Wayland kiosk as the eMMC install, but nothing is written to the
device's flash.

!!! note "Scope: TC8"
    The C60 boots differently (`booti`, `system_a`) — see
    [C60 boot chain](../internals/c60-boot.md).

!!! note "Development path, not the production install"
    The shipped panel boots the slotable Android image with `boota` (see
    [TC8 boot chain](../internals/tc8-boot.md)); netboot is for kernel
    and rootfs bring-up, and for keeping fleets of panels stateless.

The examples below use the placeholder `<server-ip>` for the TFTP+NFS
server. Substitute your own. The panel and the server must be on the
same routable network.

## 1. Prerequisites

- A built `out/nfs/` from `./build.sh --profile=nfs` — see
  [Firmware images](../build/firmware.md)
- A server reachable from the panel that can run a TFTP daemon and an
  NFS server
- Serial console on the panel's UART (115200 8N1) to interrupt U-Boot
  autoboot

## 2. Server setup (one-time)

```bash
sudo apt install tftpd-hpa nfs-kernel-server

# /etc/default/tftpd-hpa
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure --create"

sudo mkdir -p /srv/tftp/tc8 /srv/nfs/tc8

# /etc/exports — adjust the subnet to whatever your panels live on
echo "/srv/nfs/tc8 <subnet>/<mask>(rw,sync,no_subtree_check,no_root_squash)" \
    | sudo tee -a /etc/exports
sudo exportfs -ra
sudo systemctl enable --now tftpd-hpa nfs-kernel-server
```

Edit `profiles/nfs.env` in the build repo so the kernel cmdline points
at the server's IP and exported path. Rebuild with
`./build.sh --profile=nfs --skip-rootfs` after changing.

## 3. Stage the artifacts

```bash
# Kernel + dtb → TFTP root
sudo cp out/nfs/kernel/Image            /srv/tftp/tc8/Image
sudo cp out/nfs/kernel/imx8mm-tc8.dtb   /srv/tftp/tc8/imx8mm-tc8.dtb
# (Optional) the Android boot.img (unsigned AVB) — useful if the
# U-Boot supports `bootm`/`boota` of Android boot images
sudo cp out/nfs/boot.img                /srv/tftp/tc8/boot.img

# Rootfs → NFS export
sudo rm -rf /srv/nfs/tc8/*
sudo tar -xzf rootfs/out/rootfs.tar.gz -C /srv/nfs/tc8/
```

Verify:

```bash
showmount -e localhost      # should list /srv/nfs/tc8
ls /srv/tftp/tc8/           # Image, imx8mm-tc8.dtb (and optionally boot.img)
ls /srv/nfs/tc8/            # bin boot dev etc home lib root usr var ...
```

## 4. Boot the panel

Power-cycle the panel and interrupt U-Boot at the
`Hit any key to stop autoboot` prompt on the serial console. At the `=>`
prompt, substituting `<server-ip>` for the server's IP:

```sh
=> setenv autoload no
=> dhcp
=> setenv serverip <server-ip>
=> setenv bootargs "console=tty0 console=ttymxc1,115200 earlycon=ec_imx6q,0x30890000,115200 keep_bootcon panic=10 root=/dev/nfs nfsroot=<server-ip>:/srv/nfs/tc8,v3,tcp,nolock ip=:::::lan:dhcp rw rootwait fw_devlink=permissive"
=> tftpboot 0x40480000 tc8/Image
=> tftpboot 0x43000000 tc8/imx8mm-tc8.dtb
=> booti 0x40480000 - 0x43000000
```

If the U-Boot supports Android boot images, the Android boot.img
(unsigned AVB) works instead:

```sh
=> tftpboot 0x40480000 tc8/boot.img
=> bootm 0x40480000
```

The kernel then:

1. Runs early DHCP (`ip=:::::lan:dhcp`) to bring `lan` up while
   U-Boot's `ip` is still active
2. Mounts `<server-ip>:/srv/nfs/tc8` as `/`
3. Hands off to systemd → `graphical.target` → `kiosk.service`

## 5. Verify

After a few seconds the panel should be reachable over ssh (default
credentials — see [Getting in](access.md)):

```bash
ssh root@<panel-ip>

mount | grep ' / '
# <server-ip>:/srv/nfs/tc8 on / type nfs (...,vers=3,...,proto=tcp,...)

ls /dev/dri/        # card0 + card1 + renderD128
aplay -l            # tas5751-audio
pgrep -fa cog       # /usr/bin/cog ... <KIOSK_URL>
```

If `dmesg | grep -i nfs` shows `nfsroot=...` resolved and
`VFS: Mounted root (nfs filesystem)`, the netboot was successful. The
same kiosk URL config (`/etc/default/tc8-kiosk` / `KIOSK_URL=…`) applies
as in the eMMC case — see the
[configuration reference](configure.md).

## Persistence

Anything written to `/` persists in the NFS export — the rootfs is a
real shared filesystem, not a tmpfs overlay. To start fresh, re-extract
`rootfs.tar.gz` over `/srv/nfs/tc8/`.

For multiple panels sharing one NFS export, switch to a read-only export
plus an overlay-fs in initramfs (out of scope here).
