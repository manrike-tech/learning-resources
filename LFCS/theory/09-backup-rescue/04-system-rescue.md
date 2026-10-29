# System Rescue and Recovery

<!-- nav:start -->
> **Module 09 · Backup, Recovery & System Rescue** — lesson 4 of 4  
> [Module index](README.md) · [Course lesson (commands)](../../course/09-Backup-Recovery-Rescue.md#lesson-4---system-rescue-ch-34) · [Exercises for this lesson](../../exercises/09-backup-rescue/04-system-rescue.md)
<!-- nav:end -->

When a system won't boot or nobody can log in, you need to get **inside** it anyway: through special boot targets, by interrupting the boot early, or from external rescue media. The LFCS objective is *"Recover from hardware, operating system, or filesystem failures"*. In this lesson you will learn the rescue entry points, how to reset a lost root password on both families, repair `/etc/fstab`, repair filesystems, boot manually from the GRUB prompt, rebuild GRUB and the initramfs from a chroot, and handle failing disks.

## What you will learn

- Entry points: rescue/emergency targets, `rd.break`, `init=/bin/bash`, rescue media
- Resetting the root password (RHEL and Ubuntu) – and why `/.autorelabel` matters
- Fixing a broken `/etc/fstab` from emergency mode
- Repairing filesystems (ext4, XFS) including the root filesystem
- Booting manually from the `grub>` prompt
- Rescue media + `chroot` to reinstall GRUB and rebuild the initramfs
- Hardware failures: SMART, RAID members, failing LVM physical volumes
- Protecting the boot process (GRUB password)

---

## Key terms

| Term | Definition |
|------|-----------|
| **Rescue target** | systemd state with local filesystems mounted and a root shell, no network/services. |
| **Emergency target** | Minimal state: only `/` mounted **read-only**, root shell. |
| **`rd.break`** | dracut parameter: stop in the initramfs shell before switching to the real root (mounted read-only on `/sysroot`). |
| **`init=/bin/bash`** | Tell the kernel to run bash as PID 1 instead of systemd. |
| **Rescue media** | A live/installation ISO or USB used to boot a separate system that can access the broken one. |
| **chroot** | Run commands with a different directory as `/` – used to work "inside" the installed system from rescue media. |
| **Autorelabel** | Full SELinux relabel at next boot, triggered by `/.autorelabel`. |
| **Remount** | Change mount options of a mounted filesystem, e.g. `mount -o remount,rw /`. |

---

## The entry points

All are added to the kernel line in GRUB: at the menu press **`e`**, go to the line starting with `linux`, append the parameter at the **end**, press **`Ctrl+X`**. Changes apply to this boot only.

| Append | You get | Root FS | Needs root password | Typical use |
|--------|---------|---------|---------------------|-------------|
| `systemd.unit=rescue.target` (or `1`, `single`) | single-user root shell | rw, local FS mounted | yes | broken service, network config |
| `systemd.unit=emergency.target` (or `emergency`) | minimal root shell | **ro**, nothing else mounted | yes | broken fstab, FS problems |
| `rd.break` (RHEL/dracut) | initramfs shell, real root at `/sysroot` | **ro** | **no** | reset root password, fix files before boot |
| `init=/bin/bash` (any distro) | bash as PID 1 | **ro** | **no** | reset password when root is locked (Ubuntu) |
| rescue / live ISO | full separate system | yours to mount | no | GRUB/initramfs repair, unbootable systems |

> Tip: remove `rhgb quiet` / `quiet splash` on the same line to see what's happening.

On Ubuntu, the GRUB menu is often hidden: hold **Shift** (BIOS) or press **Esc** (UEFI) during boot. The "Advanced options → recovery mode" entry gives a menu with a root shell option.

> **Security consequence:** whoever can edit GRUB entries at the console can become root. Production systems protect GRUB with a password and/or use disk encryption (see the end of this lesson).

---

## Reset the root password – RHEL family (`rd.break`)

```text
1. GRUB menu → e → append to the linux line:   rd.break
2. Ctrl+X → you get:   switch_root:/#
```

```console
switch_root:/# mount -o remount,rw /sysroot     # the real root is mounted ro on /sysroot
switch_root:/# chroot /sysroot                  # work inside the installed system
sh-5.1# passwd root
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
sh-5.1# touch /.autorelabel                     # IMPORTANT on SELinux systems
sh-5.1# exit                                    # leave the chroot
switch_root:/# exit                             # continue booting (relabel runs, then reboots)
```

**Why `/.autorelabel`?** In the initramfs, SELinux policy isn't loaded yet, so `passwd` writes a new `/etc/shadow` **without the correct SELinux label**. After boot, the login programs aren't allowed to read it – logins fail. `/.autorelabel` makes the system relabel all files at the next boot (takes a few minutes, then reboots once).

Faster alternative to a full relabel: inside the chroot, run `load_policy -i` and then `chcon -t shadow_t /etc/shadow` (or `restorecon -v /etc/shadow`). It avoids the long relabel, but `/.autorelabel` is the method that always works.

---

## Reset the root password – Ubuntu/Debian (`init=/bin/bash`)

```text
1. GRUB menu (hold Shift / press Esc) → e → on the linux line, append:   init=/bin/bash
   (you can also replace "ro" with "rw" to skip the remount step)
2. Ctrl+X → you get a root prompt:   root@(none):/#
```

```console
root@(none):/# mount -o remount,rw /
root@(none):/# passwd root          # or: passwd alice  (a sudo user)
root@(none):/# sync
root@(none):/# exec /sbin/init      # continue a normal boot   (or: reboot -f)
```

- A normal `reboot` doesn't work: systemd isn't running (bash is PID 1). Use `exec /sbin/init` or `reboot -f` (after `sync`).
- If `/etc` or `/usr` are separate filesystems, mount them first (`mount -a` may work after remounting `/` rw).
- Ubuntu has no SELinux; with AppArmor no relabel is needed.

The same `init=/bin/bash` method also works on RHEL (then you **do** need `touch /.autorelabel`).

---

## Fix a broken /etc/fstab

Symptom: boot ends in **emergency mode** ("Give root password for maintenance"), `journalctl -xb` shows a failed `.mount` unit or "Timed out waiting for device".

```console
# Log in with the root password (or boot with init=/bin/bash / rd.break if root is locked)
# mount -o remount,rw /                       # root is read-only in emergency mode
# journalctl -xb | grep -iE 'mount|fstab|timed out' | tail
# systemctl --failed
# cat /etc/fstab
UUID=00000000-dead-beef-0000-000000000000  /data  ext4  defaults  0 2     ← wrong UUID
# blkid                                       # find the real UUIDs
/dev/vdb1: LABEL="DATA" UUID="3f1c2a7e-…" TYPE="ext4"
# vi /etc/fstab                               # fix the UUID, or comment the line out, or add nofail
# findmnt --verify                            # check syntax and devices
# systemctl daemon-reload
# mount -a                                    # everything mounts without errors?
# systemctl default                           # continue to the default target (or reboot)
```

Typical fstab mistakes: wrong UUID/device, typo in the filesystem type or options, missing mount point directory, a removed disk, wrong field count. Prevention: test with `findmnt --verify` + `mount -a` after every edit, `nofail` for non-essential disks.

---

## Repair filesystems

Never repair a mounted read-write filesystem.

### A data filesystem

```console
# umount /data                       # "target is busy" → fuser -vm /data
# e2fsck -f /dev/vdb1                # ext4: interactive
# e2fsck -fy /dev/vdb1               # ext4: answer yes to all repairs
# e2fsck -b 32768 /dev/vdb1          # primary superblock damaged → use a backup (dumpe2fs / mke2fs -n shows locations)
# xfs_repair /dev/vg01/lv_data       # XFS
# xfs_repair -L /dev/vg01/lv_data    # XFS last resort: zero a dirty log (may lose recent changes)
# mount /data; ls /data/lost+found   # orphaned files recovered by e2fsck end up here
```

If `xfs_repair` reports a dirty log, first try to **mount and unmount** the filesystem (this replays the log), then run `xfs_repair` again.

### The root filesystem

The root FS can't be unmounted while running from it. Options:

- Force a check at next boot: kernel parameter `fsck.mode=force` (add `fsck.repair=yes` to auto-fix). On older systems: `touch /forcefsck`.
- From emergency mode, root is read-only – ext4 can be checked read-only-mounted with care, but the safe way is:
- Boot **rescue media**, don't mount the root FS, and run `e2fsck`/`xfs_repair` on it.

A filesystem that the kernel remounted read-only due to errors (`EXT4-fs error … Remounting filesystem read-only` in `dmesg`) needs a check as soon as possible – and the underlying disk should be examined (SMART, below).

---

## Manual boot from the `grub>` prompt

If GRUB shows `grub>` (no config), you can boot by hand:

```text
grub> ls
(hd0) (hd0,gpt1) (hd0,gpt2) (hd0,gpt3) (lvm/rl-root)
grub> ls (hd0,gpt2)/
lost+found/ grub2/ loader/ vmlinuz-5.14.0-503.14.1.el9_5.x86_64 initramfs-5.14.0-503.14.1.el9_5.x86_64.img …
grub> set root=(hd0,gpt2)
grub> linux /vmlinuz-5.14.0-503.14.1.el9_5.x86_64 root=/dev/mapper/rl-root ro rd.lvm.lv=rl/root
grub> initrd /initramfs-5.14.0-503.14.1.el9_5.x86_64.img
grub> boot
```

- `ls` lists disks/partitions; `ls (hd0,gpt2)/` shows files – look for the one containing `vmlinuz-*` (that's `/boot`).
- Tab completion works for file names.
- If `/boot` is not a separate partition, paths are `/boot/vmlinuz-…` on the root partition.
- On Ubuntu the files are `vmlinuz-…` and `initrd.img-…`, root e.g. `root=/dev/mapper/ubuntu--vg-ubuntu--lv`.

Once booted, **regenerate the config** so it doesn't happen again:

```console
# grub2-mkconfig -o /boot/grub2/grub.cfg      # RHEL
# update-grub                                 # Debian/Ubuntu
```

At `grub rescue>`, only a few commands exist: `set prefix=(hd0,gpt2)/grub2`, `set root=(hd0,gpt2)`, `insmod normal`, `normal` – then continue as above. Usually it's quicker to use rescue media.

---

## Repair from rescue media (chroot)

When GRUB is gone, the initramfs is broken, or the system can't boot at all:

1. Boot the installation ISO in **rescue** mode (RHEL: "Troubleshooting → Rescue a … system"; Ubuntu: "Try Ubuntu" live session or a server ISO shell).
2. Find and mount the installed system:

```console
# lsblk -f                                    # identify root, /boot, ESP
# vgchange -ay                                # activate LVM volumes
# mount /dev/mapper/rl-root /mnt              # root
# mount /dev/vda2 /mnt/boot                   # separate /boot, if any
# mount /dev/vda1 /mnt/boot/efi               # ESP on UEFI systems
# for d in dev proc sys run; do mount --rbind /$d /mnt/$d; done
# chroot /mnt                                 # now "inside" the installed system
```

(RHEL rescue mode can do the mounting for you under `/mnt/sysroot`; then `chroot /mnt/sysroot`.)

3. Repair, inside the chroot:

```console
# --- RHEL family ---
# grub2-install /dev/vda                      # BIOS systems: reinstall boot code to the DISK (not a partition)
# dnf reinstall shim-x64 grub2-efi-x64        # UEFI systems: reinstall the EFI binaries instead
# grub2-mkconfig -o /boot/grub2/grub.cfg
# dracut -f --regenerate-all                  # rebuild all initramfs images
# --- Debian/Ubuntu ---
# grub-install /dev/vda                       # BIOS; UEFI: grub-install --target=x86_64-efi --efi-directory=/boot/efi
# update-grub
# update-initramfs -u -k all
# --- both ---
# passwd root; vi /etc/fstab; touch /.autorelabel (SELinux)
# exit; umount -R /mnt; reboot
```

Without the bind mounts of `/dev`, `/proc`, `/sys` and `/run`, tools like `grub-install` and `dracut` fail inside the chroot.

---

## Hardware failures

### Disks

```console
$ dmesg -T | grep -iE 'error|fail|I/O|ata[0-9]|nvme' | tail
[Tue Oct 21 10:42:11 2026] blk_update_request: I/O error, dev sdb, sector 123456 op 0x0:(READ)
$ sudo smartctl -H /dev/sda                     # package smartmontools: overall health
SMART overall-health self-assessment test result: PASSED
$ sudo smartctl -a /dev/sda | grep -iE 'Reallocated|Pending|Uncorrectable|Power_On'
$ sudo smartctl -t short /dev/sda; sleep 120; sudo smartctl -l selftest /dev/sda
$ sudo nvme smart-log /dev/nvme0                # NVMe (package nvme-cli)
```

Rising reallocated/pending sector counts → replace the disk soon, after backing up / moving data.

### Software RAID (md)

```console
$ cat /proc/mdstat
md0 : active raid1 sdc1[1] sdb1[0](F)          ← (F) = failed member
      5237760 blocks super 1.2 [2/1] [_U]       ← [_U] = one of two members missing
$ sudo mdadm --detail /dev/md0
$ sudo mdadm --manage /dev/md0 --fail /dev/sdb1 --remove /dev/sdb1
# replace the disk, partition it like the old one (sfdisk -d /dev/sdc | sfdisk /dev/sdd), then:
$ sudo mdadm --manage /dev/md0 --add /dev/sdd1
$ watch cat /proc/mdstat                       # resync progress
```

### LVM with a failing disk

Move data off the failing PV while it still works, then remove it:

```console
$ sudo pvs                                     # find the failing PV, check free space elsewhere
$ sudo vgextend vg01 /dev/sdd                  # add a healthy replacement first if needed
$ sudo pvmove /dev/sdb                         # move extents away (online)
$ sudo vgreduce vg01 /dev/sdb && sudo pvremove /dev/sdb
```

If a PV is already dead: `vgreduce --removemissing vg01` (LVs that lived on it are lost unless mirrored – restore from backup).

### Memory and CPU

Random crashes, kernel oopses with different traces, file corruption → test RAM (`memtest86+` from the boot menu or ISO). Machine Check Exceptions appear as `mce:` in `dmesg`/`journalctl -k` (`rasdaemon` collects them).

---

## Protecting the boot process

```console
# RHEL: password for editing GRUB entries
$ sudo grub2-setpassword
Enter password:
$ cat /boot/grub2/user.cfg
GRUB2_PASSWORD=grub.pbkdf2.sha512.10000.…
# Debian/Ubuntu: generate a hash and add to /etc/grub.d/40_custom
$ grub-mkpasswd-pbkdf2
# set superusers="admin"
# password_pbkdf2 admin grub.pbkdf2.sha512.10000.…
$ sudo update-grub
```

Combine with a firmware password, Secure Boot and full-disk encryption (LUKS) – otherwise anyone with console access or the disk can bypass these protections with rescue media.

---

## Try it yourself

> Take snapshots first; practice each on both VMs.

1. Boot `rhel1` into `rescue.target` and `emergency.target`; compare what's mounted (`findmnt`).
2. Reset the root password on `rhel1` with `rd.break` (don't forget `/.autorelabel`), and on `deb1` with `init=/bin/bash`.
3. Put a wrong UUID line into `/etc/fstab`, reboot, recover from emergency mode.
4. Unmount an ext4 LV and run a forced check; repeat with XFS (`xfs_repair -n`, then `xfs_repair`).
5. Rename `grub.cfg`, reboot, boot manually from `grub>`, then regenerate the config.
6. Boot the installation ISO in rescue mode, mount the installed system, chroot, rebuild the initramfs and the GRUB config.
7. Set a GRUB password and confirm that pressing `e` now asks for it.

---

## Common mistakes

- Forgetting `mount -o remount,rw` → "Read-only file system" when running `passwd` or editing fstab.
- Forgetting `touch /.autorelabel` after `rd.break` on SELinux systems → logins fail after the reboot.
- Using `reboot` from an `init=/bin/bash` shell (use `exec /sbin/init` or `reboot -f`).
- Running `fsck` on a mounted read-write filesystem.
- `grub2-install /dev/vda1` (a partition) instead of the disk `/dev/vda`, or running `grub2-install` on UEFI RHEL systems (reinstall `shim`/`grub2-efi` packages instead).
- Forgetting the `/dev`, `/proc`, `/sys`, `/run` bind mounts before `chroot`.

---

## Summary

| Problem | Entry point | Key commands |
|---------|------------|--------------|
| Lost root password (RHEL) | `rd.break` | `mount -o remount,rw /sysroot; chroot /sysroot; passwd; touch /.autorelabel` |
| Lost root password (Ubuntu) | `init=/bin/bash` | `mount -o remount,rw /; passwd; exec /sbin/init` |
| Bad `/etc/fstab` | emergency mode | `mount -o remount,rw /; blkid; vi /etc/fstab; findmnt --verify; mount -a` |
| Corrupt data FS | normal boot / rescue | `umount; e2fsck -fy` / `xfs_repair` |
| Corrupt root FS | rescue media or `fsck.mode=force` | `e2fsck -fy` / `xfs_repair` on the unmounted device |
| `grub>` prompt | GRUB CLI | `ls; set root=; linux … root=…; initrd …; boot` → regenerate config |
| No bootloader / broken initramfs | rescue media + chroot | `grub2-install`/`grub-install`, `grub2-mkconfig`/`update-grub`, `dracut -f`/`update-initramfs -u` |
| Failing disk | running system | `dmesg`, `smartctl -a`, `mdadm --fail/--remove/--add`, `pvmove` |

## Check yourself

1. Why is `/.autorelabel` needed after resetting a password via `rd.break`, but not after `passwd` on a running system?
2. In emergency mode, why does `vi /etc/fstab` fail to save, and how do you fix it?
3. Why can't you use `reboot` after `init=/bin/bash`?
4. Which filesystems must be bind-mounted before `chroot /mnt` to reinstall GRUB?
5. A RAID 1 shows `[_U]` in `/proc/mdstat`. What does it mean and what do you do?

<details>
<summary>Answers</summary>

1. In the initramfs SELinux isn't enforcing labels, so the new `/etc/shadow` gets a wrong label; on a running system `passwd` creates it with the correct label.
2. The root filesystem is mounted read-only; run `mount -o remount,rw /`.
3. systemd isn't running (bash is PID 1), so the normal reboot request has nobody to handle it; use `exec /sbin/init` or `sync; reboot -f`.
4. `/dev`, `/proc`, `/sys` and `/run` (plus `/boot` and `/boot/efi` if separate).
5. One of the two mirror members has failed/is missing; mark it failed and remove it with `mdadm`, replace the disk, partition it and `--add` it, then watch the resync.

</details>

## Further reading

- `man 7 dracut.cmdline`, `man 7 bootup`, `man 8 e2fsck`, `man 8 xfs_repair`, `man 8 grub2-install`, `man 8 smartctl`, `man 8 mdadm`
- [Red Hat: Changing and resetting the root password (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_monitoring_and_updating_the_kernel/index)
- [ArchWiki: GRUB – Troubleshooting](https://wiki.archlinux.org/title/GRUB#Troubleshooting) · [chroot](https://wiki.archlinux.org/title/Chroot) · [fsck](https://wiki.archlinux.org/title/Fsck)
- [ArchWiki: RAID](https://wiki.archlinux.org/title/RAID) · [S.M.A.R.T.](https://wiki.archlinux.org/title/S.M.A.R.T.)

---

<!-- footer:start -->
[← The Boot Process – and Where It Breaks](03-boot-process.md) · [Module index](README.md)  
**Practice now:** [Exercises – System Rescue and Recovery](../../exercises/09-backup-rescue/04-system-rescue.md)
<!-- footer:end -->
