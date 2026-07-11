# Resolving "Disk is Mounted" Error During Proxmox Reinstall

**Date:** July 2026
**Category:** Infrastructure / Troubleshooting

## Summary
Documenting how I resolved a persistent "disk mounted" error that blocked 
reinstalling Proxmox VE on Node 1, caused by leftover LVM structures from 
a previous failed install attempt.

## Problem
While reinstalling Proxmox VE on a blank HP EliteDesk node, the installer 
repeatedly refused to proceed, showing: disk/partition '/dev/nvme0n1p2' 
is mounted (500).

## Investigation / What I Tried
- Rebooted the node and retried — same error persisted
- Checked active mounts with `mount | grep nvme0n1` — nothing showed, 
  which was misleading since LVM volumes mount under /dev/mapper/, not 
  the raw device name
- Attempted `umount` on the partition directly — failed
- Checked for an active LVM volume group with `vgs` — found a leftover 
  `pve` volume group from a previous install attempt
- Tried `vgchange -an pve` to deactivate it — failed with "contains a 
  filesystem in use," meaning a logical volume inside it was still mounted
- Located the actual mount under `/dev/mapper/pve-root`, attempted to 
  unmount — got "target is busy"

## Root Cause
A previous, interrupted Proxmox install had left an active LVM volume 
group with a mounted logical volume. Since it wasn't visible under the 
raw device name, standard `umount` checks missed it entirely.

## Resolution
After a full power cycle (not just installer reboot), the LVM activation 
cleared. From there, I confirmed the disk was clean with `lsblk` and 
proceeded with a fresh install successfully.

## Lessons Learned
- LVM volumes don't always show up when grepping the raw device name — 
  check `/dev/mapper/` and `vgs`/`lvs` directly
- A full power cycle can clear stuck LVM/mount states that a soft reboot 
  from an installer environment won't
- Next time, I'll wipe the disk with `wipefs` immediately after a failed 
  install attempt, before walking away, to avoid this on the next boot
