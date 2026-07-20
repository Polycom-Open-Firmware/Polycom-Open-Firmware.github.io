# Recovery

!!! note "Scope"
    The ladder below describes the TC8. The C60's recovery story is
    simpler — see [the note at the end](#recovering-a-c60).

The TC8's install is designed so that the factory stage-1 bootloader is
**never overwritten** — it lives in a storage region no project flow
writes. Whatever goes wrong above it, the panel keeps a working first
boot stage, so a bad flash can't permanently brick the unit.

Work down this ladder from the least severe symptom that matches.

## 1. The OS won't boot (bad slot)

The panel powers up, the boot logo appears, but Debian never comes up —
typically after an interrupted or failed OS flash.

**Fix:** enter fastboot with the four-finger gesture at the boot selector
(see [Re-provisioning](reinstall.md)) and re-run the wizard's install to
rewrite the active slot. If the other A/B slot holds a known-good system
(for example stock Android kept as a way back), switching the active slot
is an alternative.

## 2. Broken stage-2 bootloader

A stage-2 bootloader update went wrong.

**Fix:** re-stage it through the wizard's bootloader-update flow. The
update travels via a spare partition: the running OS reflashes the
stage-2 storage region and verifies the write before switching over, so
the update path is safe to simply run again. See
[Re-provisioning](reinstall.md#config-and-bootloader-updates-no-reflash-needed).

## 3. Damaged partition table

The eMMC's partition table (GPT) has been wiped or corrupted — the
device enumerates in fastboot but partition operations fail.

**Fix:** nothing manual. The wizard detects a missing or broken table
(its partition-size queries fail) and repairs it from a restore image
shipped with each release — over USB fastboot, no serial, no brick risk.
The restore image rewrites **only** the partition table: the bootloader
environment and all partition contents are untouched. After the repair
the normal install proceeds. Note that the configure flow deliberately
refuses to run while the table is damaged — repair first.

## 4. No signs of life (SDP rescue)

The panel does not boot and does not present fastboot. Because no project
flow ever writes the factory stage-1, this is very rare — it points at a
damaged factory bootloader region or a hardware fault.

**Last resort:** the i.MX chip's ROM-resident USB recovery mode (SDP)
still answers, and NXP's `uuu` tool can load a bootloader into RAM over
USB. This path is outside the wizard's scope and requires familiarity
with the NXP recovery tooling.

## Recovering a C60

The C60's normal unlock path **is** its rescue path: the SDP USB-download
mode lives in the chip's ROM and is selected with the BOOT_MODE switches,
so no matter what is on the eMMC, the wizard can always load U-Boot into
RAM again and reflash from there. Follow the
[C60 quickstart](c60.md) from the top.

No partition-table restore image ships for the C60 — a wiped C60
partition table needs manual repartitioning.
