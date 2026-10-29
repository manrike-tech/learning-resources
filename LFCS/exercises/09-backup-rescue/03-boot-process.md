# Exercises – The Boot Process

<!-- nav:start -->
> **Module 09 · Backup, Recovery & System Rescue** — lesson 3 of 4  
> Read first: [Theory – The Boot Process – and Where It Breaks](../../theory/09-backup-rescue/03-boot-process.md) · [Course lesson (commands)](../../course/09-Backup-Recovery-Rescue.md#lesson-3---boot-process-recap-where-it-can-break) · [Module exercises](README.md)
<!-- nav:end -->

> Take a VM snapshot before each exercise that changes boot configuration.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Map your boot chain ★

For each VM, record: BIOS or UEFI; boot entries (UEFI); bootloader files; GRUB config path; kernel and initramfs files; root device on the command line; default target.

<details>
<summary>Solution</summary>

```bash
[ -d /sys/firmware/efi ] && echo UEFI || echo BIOS
efibootmgr -v 2>/dev/null
ls /boot /boot/efi/EFI/* 2>/dev/null
ls /boot/grub*/grub.cfg; ls /boot/loader/entries/ 2>/dev/null
cat /proc/cmdline; systemctl get-default
```

</details>

---

## Exercise 2 – Watch a boot ★

Boot once with `quiet`/`rhgb`/`splash` removed (GRUB `e`). Then, after boot, find in `journalctl -b`: the kernel version line, the moment the root filesystem was mounted, the switch to the real root, and when `multi-user.target` was reached.

<details>
<summary>Solution</summary>

```bash
journalctl -b | grep -m1 "Linux version"
journalctl -b | grep -iE "EXT4-fs.*mounted|XFS.*Mounting|Mounted /sysroot"
journalctl -b | grep -iE "Switching root|switch_root"
journalctl -b | grep -i "Reached target.*Multi-User"
```

</details>

---

## Exercise 3 – Analyze boot time ★

Show total boot time per stage, the 5 slowest units, and the critical chain. Which unit delayed boot most?

<details>
<summary>Solution</summary>

```bash
systemd-analyze; systemd-analyze blame | head -5; systemd-analyze critical-chain
```

</details>

---

## Exercise 4 – Targets ★★

1. Boot once into `rescue.target`, then once into `emergency.target`. In each, compare `findmnt` output and `systemctl list-units --state=running | wc -l`.
2. Switch a running system to `rescue.target` with `isolate` from the console, then back with `systemctl default`.

---

## Exercise 5 – Break fstab → emergency mode ★★

Add `UUID=11111111-2222-3333-4444-555555555555 /broken xfs defaults 0 0` to `/etc/fstab`, create `/broken`, reboot. Note the exact messages and how long it waits. Fix it (see [System Rescue exercises](04-system-rescue.md)).

---

## Exercise 6 – Older kernel ★

If two kernels are installed, boot the older one once from the GRUB menu ("Advanced options" on Ubuntu). Verify with `uname -r`. Which kernel boots next time?

---

## Challenge – Rebuild the initramfs ★★★

1. Show whether the virtio block (or your disk) driver is in the current initramfs.
2. Rebuild the initramfs for the running kernel, then for all kernels.
3. Explain when an initramfs rebuild is necessary (give three situations).

<details>
<summary>Solution</summary>

```bash
lsinitrd | grep virtio_blk || lsinitramfs /boot/initrd.img-$(uname -r) | grep virtio_blk
dracut -f; dracut -f --regenerate-all          # RHEL
update-initramfs -u; update-initramfs -u -k all # Debian
```

Situations: storage driver/controller change; LVM/RAID/LUKS changes on the root device (crypttab); blacklisting or setting options for modules loaded in early boot; changing the root filesystem type.

</details>

---

<!-- footer:start -->
[← Exercises: Backup Tools: tar, rsync, dd and Friends](02-backup-tools.md) · [Module exercises](README.md) · [Exercises: System Rescue and Recovery →](04-system-rescue.md)
<!-- footer:end -->
