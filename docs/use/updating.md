# Updating

What to do when a new firmware release ships. Two things can update
independently: the **OS image** (kernel + rootfs, re-flashed via the
wizard) and the **stage-2 bootloader** (staged through the `cache`
partition and flashed by the running OS).

!!! note "Scope"
    The mechanics below describe the TC8. The C60 re-flashes through the
    provisioner's C60 flow; config blobs apply to both devices.

## Updating the OS: re-flash via the wizard

An OS update is a re-run of the provisioner's **flashos** step with the
new release: it rewrites the boot images and the rootfs of the active
slot. See [Re-provisioning](../install/reinstall.md).

What survives a re-flash:

- **`/root`** — it lives on the `facres` partition, which the flash
  never touches. Your ssh keys, dotfiles and files persist. See
  [Storage model](storage.md).
- **A staged config blob** — the `cache` partition is not part of the OS
  flash. The wizard's reinstall flow flashes a config blob by default,
  and it is applied on the first boot of the new OS, so the unit comes
  back configured. See the [configuration reference](configure.md).

What does not survive:

- **Changes made in maintenance mode** (`apt install`, edited baked
  config) — they live in the rootfs the flash replaces. Re-apply them,
  or bake them into the image.
- **Configuration applied from an earlier blob** — the applied state
  lives in the replaced filesystem; the blob pushed with the reinstall
  restores it.

### Entering fastboot without touching the panel

The local path into fastboot is the four-finger gesture on the logo
during the boot-selector window. For a remote unit, the bootloader
honors a saved `gesture_sel` variable, and the image ships
`/etc/fw_env-stage2.config` pointing at the stage-2 environment. To make
the *next* boot land in fastboot — once, self-disarming:

```sh
fw_setenv -c /etc/fw_env-stage2.config gesture_sel \
  "setenv gesture_sel bootsel; saveenv; fastboot usb 0"
reboot
```

The panel restores the normal boot flow and parks in fastboot on its USB
data port; flash with the wizard, then `fastboot reboot`.

## Updating the bootloader (stage-2)

The stage-2 U-Boot lives in the eMMC `boot1` hardware partition. The
wizard never writes it directly: it stages the new image in the `cache`
partition (alongside or instead of a config blob), and the running OS
flashes it.

The sequence, and why the update converges after the next reboot:

1. The wizard's job finishes at `fastboot flash cache` +
   `fastboot reboot`.
2. The unit comes up on the **old** stage-2. At boot,
   `tc8-update-bootloader.service` validates the staged image
   (sha256 + stage-2 signature check), compares it to `boot1`, and
   flashes only if different — with a read-back verify.
3. The **next** power-cycle runs the new stage-2.

One extra reboot makes it current in one visible step; otherwise it
converges on its own — the write is idempotent, and a re-flash of the
same blob is a no-op.

### Failure behavior

Nothing in this flow can brick the unit:

- `boot0` (the stock, signed stage-1) is **never touched**, so SDP/`uuu`
  recovery always works.
- The OS verifies the sha256 before writing and reads back after.
- A corrupt or half-written blob is ignored — the unit keeps its current
  bootloader.

The staged-blob layout and the updater's exact checks are in
[Config partition](../internals/config-partition.md); recovery paths are
in [Recovery](../install/recovery.md).

## Netbooted fleets

A netbooted panel has no on-flash OS to update: replace the kernel/DTB
in the TFTP root and re-extract the new rootfs over the NFS export. See
[Network boot](netboot.md).
