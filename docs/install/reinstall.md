# Re-provisioning

!!! note "Scope"
    The gesture, slot, and remote-fastboot sections describe the TC8.
    For the C60, see [the note below](#re-provisioning-a-c60).

After the first install, a TC8 never needs the serial cable again. Every
later operation — reinstalling the OS, changing the configuration,
updating the bootloader — starts by getting the panel into **fastboot**
and pointing the [Setup Wizard](https://wizard.openpolycom.cc/) at it.

## The four-finger gesture

On every boot the stage-2 bootloader shows a boot-selector logo with a
short gesture window (3 seconds by default). **Hold four fingers on the
logo** during that window and the panel drops into fastboot on its USB
data port instead of booting the OS. Connect the USB cable, open the
wizard, and re-run whichever flow you need.

The window length is tunable via the `bootsel_win_ms` variable in the
stage-2 environment.

## A/B slots — keeping stock as a way back

The TC8's install reuses the stock Android A/B slot partitions; there is
no repartitioning. The wizard's install flashes slot `a` by default,
which replaces stock entirely — but it can target slot `b` instead,
leaving the stock Android images in slot `a` as a way back. The
bootloader boots whichever slot is marked active (`fastboot set_active`),
so switching between the two is a slot flip away.

Reinstalling is just re-running the wizard's install flow: it rewrites
the chosen slot and the root filesystem and reboots. Root's home directory
survives — `/root` lives on a spare partition that reinstalls never touch
(see [Storage model](../use/storage.md)).

## Entering fastboot remotely

For a panel that is installed somewhere fingers can't reach, the next
boot can be aimed at fastboot from a shell. The bootloader honors a saved
`gesture_sel` variable, and the image ships `/etc/fw_env-stage2.config`
pointing at the stage-2 environment. To make the *next* boot land in
fastboot — once, self-disarming:

```sh
fw_setenv -c /etc/fw_env-stage2.config gesture_sel \
  "setenv gesture_sel bootsel; saveenv; fastboot usb 0"
reboot
```

The panel restores the normal boot flow and parks in fastboot on its USB
data port; flash with the wizard (or plain `fastboot`), then
`fastboot reboot`. The local four-finger path keeps working regardless.

## Config and bootloader updates — no reflash needed

Changing a device's configuration or updating the stage-2 bootloader does
not require reinstalling the OS. Both travel through a spare partition:
the wizard flashes a blob over fastboot, and the running OS applies it on
the next boot — checksum-verified and idempotent, and the factory
bootloader is never touched. Use the wizard's configure and
bootloader-update flows; the underlying contract is described in
[Config partition](../internals/config-partition.md) and the available
settings in the [configuration reference](../use/configure.md).

## Re-provisioning a C60

The C60 re-enters install mode the same way it was first unlocked: set
the BOOT_MODE switches to USB download and re-run the wizard, which loads
U-Boot over USB and continues into the same fastboot install and
configure flows. This path lives in the chip's ROM, so it is always
available. See the [C60 quickstart](c60.md).
