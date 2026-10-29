# Module 09 - Backup, Recovery & System Rescue

**LFS207:** Ch. 32 Backup and Recovery Methods · Ch. 34 System Rescue
**Exam domain:** Operations & Deployment (25%) — "Recover from hardware, operating system, or filesystem failures"  
**Theory:** [Module 09 theory notes](../theory/09-backup-rescue/README.md) · **Exercises:** [Module 09 exercises](../exercises/09-backup-rescue/README.md)  

---

## Learning Objectives

- Choose a backup strategy and implement it with standard tools.
- Restore files, filesystems and boot loaders.
- Recover a system that won't boot.

---

## Lesson 1 - Backup Strategy (Ch. 32)

> **Theory:** [Backup Strategy](../theory/09-backup-rescue/01-backup-strategy.md) · **Exercises:** [practice tasks](../exercises/09-backup-rescue/01-backup-strategy.md)

| Type | Copies | Restore needs |
|------|--------|---------------|
| Full | Everything | Last full |
| Incremental | Changes since **last backup of any kind** | Full + every incremental |
| Differential | Changes since **last full** | Full + last differential |

Rule of thumb **3-2-1**: 3 copies, 2 media types, 1 off-site. A backup that has never been restored is not a backup.

What to back up on a server: `/etc`, `/home`, `/var/lib/<app>`, `/srv`, databases (dumps), package list (`dpkg --get-selections` / `rpm -qa`), partition layout (`sfdisk -d /dev/sda`), LVM metadata (`/etc/lvm/backup`).

---

## Lesson 2 - Backup Tools

> **Theory:** [Backup Tools: tar, rsync, dd and Friends](../theory/09-backup-rescue/02-backup-tools.md) · **Exercises:** [practice tasks](../exercises/09-backup-rescue/02-backup-tools.md)

```bash
# tar - full + incremental
tar -czpf /backup/etc-$(date +%F).tar.gz /etc
tar --listed-incremental=/backup/home.snar -czpf /backup/home-full.tar.gz /home
tar --listed-incremental=/backup/home.snar -czpf /backup/home-inc1.tar.gz /home
tar -xzpf /backup/etc-2026-10-20.tar.gz -C /restore etc/ssh/sshd_config   # single file

# rsync - sync/mirror
rsync -aAXv --delete /home/ /backup/home/          # -A ACLs, -X xattrs (SELinux!)
rsync -avz -e ssh /srv/ backup@nas:/backups/srv/
rsync -avn ...                                     # dry run

# dd - raw image (whole disk / partition / MBR)
dd if=/dev/vdb of=/backup/vdb.img bs=4M status=progress
dd if=/dev/sda of=/backup/mbr.bin bs=512 count=1   # MBR (incl. partition table)
dd if=/backup/vdb.img of=/dev/vdb bs=4M

# cpio
find /etc | cpio -ov > /backup/etc.cpio
cpio -idv < /backup/etc.cpio

# Filesystem-level
xfsdump / xfsrestore          # xfs
dump / restore                # ext (legacy)
lvcreate -s ...               # LVM snapshot → consistent backup of a live FS
```

Automate with cron or a systemd timer (Module 04).

---

## Lesson 3 - Boot Process Recap (where it can break)

> **Theory:** [The Boot Process – and Where It Breaks](../theory/09-backup-rescue/03-boot-process.md) · **Exercises:** [practice tasks](../exercises/09-backup-rescue/03-boot-process.md)

```
Firmware (BIOS/UEFI) → GRUB2 → kernel + initramfs → systemd (PID 1) → default.target
```

| Symptom | Likely cause |
|---------|--------------|
| `grub>` / `grub rescue>` prompt | GRUB config or files missing |
| Kernel panic, can't mount root | Bad initramfs, wrong `root=`, missing driver |
| Emergency mode | `/etc/fstab` error, failed filesystem check |
| Boots, can't log in | Lost root password, broken PAM, `/etc/shadow` permissions |

---

## Lesson 4 - System Rescue (Ch. 34)

> **Theory:** [System Rescue and Recovery](../theory/09-backup-rescue/04-system-rescue.md) · **Exercises:** [practice tasks](../exercises/09-backup-rescue/04-system-rescue.md)

### Rescue / emergency targets

At the GRUB menu press `e`, go to the `linux` line, append one of:

| Append | Result |
|--------|--------|
| `systemd.unit=rescue.target` | Single-user, most filesystems mounted, needs root password |
| `systemd.unit=emergency.target` | Minimal, root FS read-only, needs root password |
| `rd.break` (RHEL) | Shell inside initramfs, before root pivot |
| `init=/bin/bash` | Shell as PID 1, no password (root FS read-only) |

Then `Ctrl+X` to boot.

### Reset a lost root password

**RHEL family:**
```bash
# GRUB: append rd.break
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel     # SELinux relabel - DON'T FORGET
exit; exit
```

**Debian/Ubuntu:**
```bash
# GRUB: append init=/bin/bash
mount -o remount,rw /
passwd root
exec /sbin/init          # or: sync; reboot -f
```

### Fix a broken /etc/fstab

```bash
# From emergency mode
mount -o remount,rw /
vi /etc/fstab            # fix or comment the bad line (blkid for correct UUIDs)
findmnt --verify; mount -a
systemctl daemon-reload; reboot
```

### Repair a filesystem

```bash
umount /dev/vg01/lv_data            # never fsck a mounted FS
e2fsck -fy /dev/vg01/lv_data        # ext4
xfs_repair /dev/vg01/lv_data        # xfs (xfs_repair -L as last resort: zeroes the log)
# Root FS: from rescue media, or force at next boot
touch /forcefsck  # (older)  |  fsck.mode=force on the kernel command line
```

### Repair GRUB / initramfs from live/rescue media

```bash
# Boot rescue ISO, find and mount the system
lsblk; vgchange -ay
mount /dev/mapper/vg-root /mnt
mount /dev/sda1 /mnt/boot             # if separate (and /boot/efi on UEFI)
for d in dev proc sys run; do mount --bind /$d /mnt/$d; done
chroot /mnt

# Debian/Ubuntu
grub-install /dev/sda && update-grub
update-initramfs -u -k all
# RHEL family
grub2-install /dev/sda            # BIOS; on UEFI reinstall shim/grub2-efi packages instead
grub2-mkconfig -o /boot/grub2/grub.cfg
dracut -f --regenerate-all
```

### From the `grub>` prompt (manual boot)

```
grub> ls
grub> set root=(hd0,gpt2)
grub> linux /vmlinuz-<version> root=/dev/mapper/vg-root ro
grub> initrd /initramfs-<version>.img
grub> boot
```

### Hardware failures

- Disk errors: `dmesg -T | grep -i error`, `smartctl -a /dev/sda` (smartmontools).
- Failed RAID member: `cat /proc/mdstat`, `mdadm --manage /dev/md0 --fail /dev/sdb1 --remove /dev/sdb1 --add /dev/sdc1`.
- Failed LVM PV: `pvmove` data off a failing disk while it still works.

---

## Lab 09

1. Create a full and an incremental `tar` backup of `/home`; delete a file; restore only that file.
2. Mirror `/etc` to the other VM with rsync over SSH, preserving ACLs and xattrs.
3. Reset the root password on **both** VMs using the GRUB method for each family.
4. Insert a bad line in `/etc/fstab`, reboot, recover.
5. Corrupt GRUB config (rename `grub.cfg`), boot manually from the `grub>` prompt, then regenerate it.
6. Unmount an ext4 LV, run a forced check; repeat on xfs.

## Self-check

- [ ] I can explain incremental vs differential.
- [ ] I can reset a root password on both families without notes (and remember `.autorelabel`).
- [ ] I can recover from a bad fstab in under 3 minutes.
- [ ] I can chroot into a system from rescue media and reinstall GRUB.
