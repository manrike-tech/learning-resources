# Exercises – Creating, Mounting and Repairing Filesystems

<!-- nav:start -->
> **Module 05 · Storage** — lesson 3 of 9  
> Read first: [Theory – Creating, Mounting and Repairing Filesystems](../../theory/05-storage/03-filesystems-mounting.md) · [Course lesson (commands)](../../course/05-Storage.md#lesson-3---creating-checking--mounting-ch-2122) · [Module exercises](README.md)
<!-- nav:end -->

> Uses the partitions from the [partitioning exercises](02-partitioning.md): `/dev/vdb1` (1 GiB) and `/dev/vdc1` (2 GiB).

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Exam-style persistent mount ★

> On disk 1, create a filesystem of type ext4 with label `DATA` and mount it on `/data` persistently using its UUID.

**Verify:** `findmnt /data`; `grep /data /etc/fstab` shows `UUID=`; reboot and check again.

<details>
<summary>Solution</summary>

```bash
mkfs.ext4 -L DATA /dev/vdb1
mkdir -p /data
echo "UUID=$(blkid -s UUID -o value /dev/vdb1) /data ext4 defaults 0 2" >> /etc/fstab
findmnt --verify && systemctl daemon-reload && mount -a
```

</details>

---

## Exercise 2 – XFS with options ★★

Create XFS with label `LOGS` on `vdc1` and mount it on `/logs` with `noatime,nodev,nosuid` persistently by label.

<details>
<summary>Solution</summary>

```bash
mkfs.xfs -L LOGS /dev/vdc1
mkdir -p /logs
echo "LABEL=LOGS /logs xfs defaults,noatime,nodev,nosuid 0 0" >> /etc/fstab
systemctl daemon-reload && mount -a && findmnt /logs
```

</details>

---

## Exercise 3 – Target is busy ★

In one shell `cd /data`; in another try `umount /data`. Identify the blocking process with two tools and unmount.

<details>
<summary>Solution</summary>

```bash
fuser -vm /data; lsof +f -- /data
# exit/cd away in the other shell (or fuser -km /data to kill – careful)
umount /data && mount -a
```

</details>

---

## Exercise 4 – Mount options in action ★★

1. Remount `/data` read-only and try to create a file. Remount read-write.
2. Copy `/usr/bin/ls` to `/logs/ls` and try to run it (nosuid/nodev don't stop it). Remount `/logs` with `noexec` and try again.

<details>
<summary>Solution</summary>

```bash
mount -o remount,ro /data; touch /data/x     # Read-only file system
mount -o remount,rw /data
cp /usr/bin/ls /logs/ls; /logs/ls /
mount -o remount,noexec /logs; /logs/ls      # Permission denied
mount -o remount,exec /logs
```

</details>

---

## Exercise 5 – Break fstab safely ★★

1. Add a line with a non-existent UUID for `/broken` **without** `nofail`. Run `findmnt --verify` and `mount -a`. What would happen at boot?
2. Add `nofail` and test again. Then remove the line.

<details>
<summary>Solution</summary>

```bash
mkdir -p /broken
echo "UUID=00000000-0000-0000-0000-000000000000 /broken ext4 defaults 0 2" >> /etc/fstab
findmnt --verify     # reports the missing source
mount -a             # error; at boot: emergency mode after the device timeout
sed -i 's|^UUID=00000000.*|UUID=00000000-0000-0000-0000-000000000000 /broken ext4 defaults,nofail 0 2|' /etc/fstab
sed -i '/\/broken/d' /etc/fstab; systemctl daemon-reload
```

</details>

---

## Exercise 6 – Check and repair ★★

1. Unmount `/data` and run a forced check.
2. Unmount `/logs` and run an XFS check without modifying anything.
3. Find the backup superblock locations of `/dev/vdb1`.

<details>
<summary>Solution</summary>

```bash
umount /data && e2fsck -f /dev/vdb1 && mount /data
umount /logs && xfs_repair -n /dev/vdc1 && mount /logs
dumpe2fs /dev/vdb1 2>/dev/null | grep -i "backup superblock"
```

</details>

---

## Exercise 7 – Tuning ★

1. Change the label of `/dev/vdb1` to `ARCHIVE` and update fstab if needed.
2. Set reserved blocks to 1%. Compare `df -h /data` before and after.
3. Show the current mount count and last check time.

<details>
<summary>Solution</summary>

```bash
e2label /dev/vdb1 ARCHIVE          # UUID unchanged, so a UUID-based fstab still works
df -h /data; tune2fs -m 1 /dev/vdb1; df -h /data
tune2fs -l /dev/vdb1 | grep -E 'Mount count|Last checked'
```

</details>

---

## Challenge – Corrupt and recover ★★★

On a **loop image** (never your real disks): create ext4, write files, unmount, overwrite the primary superblock (`dd if=/dev/zero of=img bs=1024 seek=1 count=1 conv=notrunc`), try to mount, then repair using a backup superblock and verify the files.

<details>
<summary>Solution</summary>

```bash
truncate -s 200M /root/c.img; mkfs.ext4 -q /root/c.img
mkdir -p /mnt/c; mount -o loop /root/c.img /mnt/c; echo important > /mnt/c/file; umount /mnt/c
dd if=/dev/zero of=/root/c.img bs=1024 seek=1 count=1 conv=notrunc
mount -o loop /root/c.img /mnt/c          # fails: wrong fs type, bad superblock
e2fsck -fy -b 8193 /root/c.img || e2fsck -fy -b 32768 /root/c.img
mount -o loop /root/c.img /mnt/c && cat /mnt/c/file
umount /mnt/c; rm /root/c.img
```

(Backup locations depend on block size: 8193 for 1 KiB blocks, 32768 for 4 KiB blocks – `mke2fs -n` shows them.)

</details>

---

<!-- footer:start -->
[← Exercises: Disk Partitioning](02-partitioning.md) · [Module exercises](README.md) · [Exercises: Logical Volume Management (LVM) →](04-lvm.md)
<!-- footer:end -->
