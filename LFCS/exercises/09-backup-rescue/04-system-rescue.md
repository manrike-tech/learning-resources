# Exercises – System Rescue and Recovery

<!-- nav:start -->
> **Module 09 · Backup, Recovery & System Rescue** — lesson 4 of 4  
> Read first: [Theory – System Rescue and Recovery](../../theory/09-backup-rescue/04-system-rescue.md) · [Course lesson (commands)](../../course/09-Backup-Recovery-Rescue.md#lesson-4---system-rescue-ch-34) · [Module exercises](README.md)
<!-- nav:end -->

> **Take a VM snapshot before every exercise.** Practise each one on both families until you can do it without notes – these are classic exam tasks.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Root password reset, RHEL ★★

On `rhel1`, set an unknown root password (`echo "root:$(openssl rand -hex 16)" | chpasswd`), reboot, and reset it to `Lfcs2026!` using `rd.break`. Verify you can log in as root on the console after the relabel.

<details>
<summary>Solution</summary>

GRUB `e` → append `rd.break` → `Ctrl+X`:

```text
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel
exit
exit
```

The system relabels and reboots once; then log in.

</details>

---

## Exercise 2 – Root password reset, Ubuntu ★★

On `deb1`, reset the password of your admin user (or root) with `init=/bin/bash`.

<details>
<summary>Solution</summary>

GRUB (hold Shift / Esc) → `e` → append `init=/bin/bash` → `Ctrl+X`:

```text
mount -o remount,rw /
passwd ana
sync
exec /sbin/init
```

</details>

---

## Exercise 3 – What if you forget the relabel? ★★★

Repeat Exercise 1 **without** `touch /.autorelabel`. Try to log in. Then boot with `rd.break` again and fix it **without** a full relabel.

<details>
<summary>Solution</summary>

Logins fail (`/etc/shadow` has the wrong label). Fix:

```text
mount -o remount,rw /sysroot
chroot /sysroot
load_policy -i
restorecon -v /etc/shadow      # or: chcon -t shadow_t /etc/shadow
exit; exit
```

</details>

---

## Exercise 4 – Broken fstab ★★

> Insert a bad line in `/etc/fstab`, reboot, recover.

Use a wrong UUID for an existing mount point; after recovery the system must boot normally and the real disk must be mounted.

<details>
<summary>Solution</summary>

In emergency mode (root password):

```bash
mount -o remount,rw /
journalctl -xb | grep -i -E 'mount|timed out' | tail
blkid
vi /etc/fstab          # correct the UUID (or comment out / add nofail)
findmnt --verify && systemctl daemon-reload && mount -a
systemctl default      # or reboot
```

On Ubuntu without a root password, use `init=/bin/bash`, remount rw and edit fstab there.

</details>

---

## Exercise 5 – Repair filesystems ★★

1. Unmount an ext4 LV, corrupt it slightly (`debugfs -w -R "clri <12>" /dev/vgapp/lvdata` clears an inode – lab only!), run `e2fsck -fy` and inspect `lost+found`.
2. Unmount an XFS filesystem and run `xfs_repair -n`, then `xfs_repair`.
3. Force a check of all filesystems (including root) at the next boot with a kernel parameter.

<details>
<summary>Solution</summary>

```bash
umount /data2; e2fsck -fy /dev/vgapp/lvdata; mount /data2; ls /data2/lost+found
umount /logs; xfs_repair -n /dev/vdc1; xfs_repair /dev/vdc1; mount /logs
grubby --update-kernel=DEFAULT --args="fsck.mode=force"     # RHEL; remove it afterwards with --remove-args
```

</details>

---

## Exercise 6 – Boot from the grub> prompt ★★★

> Corrupt GRUB config (rename `grub.cfg`), boot manually from the `grub>` prompt, then regenerate it.

<details>
<summary>Solution</summary>

```bash
mv /boot/grub2/grub.cfg /boot/grub2/grub.cfg.bak    # Ubuntu: /boot/grub/grub.cfg
reboot
```

```text
grub> ls
grub> ls (hd0,gpt2)/                         ← find the partition with vmlinuz-*
grub> set root=(hd0,gpt2)
grub> linux /vmlinuz-<TAB> root=/dev/mapper/rl-root ro rd.lvm.lv=rl/root
grub> initrd /initramfs-<TAB>
grub> boot
```

After boot: `grub2-mkconfig -o /boot/grub2/grub.cfg` (RHEL) or `update-grub` (Ubuntu). (On RHEL UEFI systems the stub `/boot/efi/EFI/rocky/grub.cfg` points to `/boot/grub2/grub.cfg`.)

</details>

---

## Exercise 7 – Rescue media and chroot ★★★

Boot the installation ISO in rescue mode. Without booting the installed system: mount it, chroot into it, rebuild the initramfs for all kernels, regenerate the GRUB config, change the root password, and reboot into it.

<details>
<summary>Solution</summary>

```bash
lsblk -f; vgchange -ay
mount /dev/mapper/rl-root /mnt; mount /dev/vda2 /mnt/boot; mount /dev/vda1 /mnt/boot/efi 2>/dev/null
for d in dev proc sys run; do mount --rbind /$d /mnt/$d; done
chroot /mnt
dracut -f --regenerate-all          # Debian: update-initramfs -u -k all
grub2-mkconfig -o /boot/grub2/grub.cfg   # Debian: update-grub
passwd root; touch /.autorelabel
exit; umount -R /mnt; reboot
```

</details>

---

## Exercise 8 – Software RAID failure ★★★

On a VM with two extra disks: create a RAID 1 `md0` from `vdb1` and `vdc1`, put a filesystem on it, then simulate a failure of `vdb1`, remove it, "replace" it (wipe and re-add) and watch the resync.

<details>
<summary>Solution</summary>

```bash
dnf install -y mdadm || apt install -y mdadm
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/vdb1 /dev/vdc1
mkfs.ext4 /dev/md0; mount /dev/md0 /mnt; echo data > /mnt/file
mdadm --manage /dev/md0 --fail /dev/vdb1; cat /proc/mdstat         # [_U]
mdadm --manage /dev/md0 --remove /dev/vdb1
wipefs -a /dev/vdb1; mdadm --manage /dev/md0 --add /dev/vdb1
watch -n1 cat /proc/mdstat
mdadm --detail --scan >> /etc/mdadm.conf     # Debian: /etc/mdadm/mdadm.conf + update-initramfs -u
```

</details>

---

## Exercise 9 – Disk health ★

Install smartmontools and show the health status and key attributes of your disks (virtual disks may not support SMART – note what you see). Find I/O errors in the kernel log.

<details>
<summary>Solution</summary>

```bash
smartctl -H /dev/sda; smartctl -a /dev/sda | head -40
dmesg -T | grep -iE 'I/O error|ata[0-9].*error|nvme.*error'
```

</details>

---

## Challenge – Protect GRUB ★★

Set a GRUB password on `rhel1` (`grub2-setpassword`) and on `deb1` (`grub-mkpasswd-pbkdf2` + `/etc/grub.d/40_custom`). Verify that editing an entry with `e` now asks for credentials. Explain why this alone doesn't protect against someone with physical access and a USB stick.

<details>
<summary>Solution</summary>

They can boot their own media and mount the disk – you also need a firmware password, Secure Boot and disk encryption (LUKS).

</details>

---

<!-- footer:start -->
[← Exercises: The Boot Process – and Where It Breaks](03-boot-process.md) · [Module exercises](README.md)
<!-- footer:end -->
