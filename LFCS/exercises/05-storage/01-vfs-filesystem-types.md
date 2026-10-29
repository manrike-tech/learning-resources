# Exercises – The VFS and Filesystem Types

<!-- nav:start -->
> **Module 05 · Storage** — lesson 1 of 9  
> Read first: [Theory – The VFS and Filesystem Types](../../theory/05-storage/01-vfs-filesystem-types.md) · [Course lesson (commands)](../../course/05-Storage.md#lesson-1---the-vfs--filesystem-types-ch-19) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Draw your storage stack ★

Using `lsblk -f`, `pvs`, `lvs` and `findmnt --real`, write in `/root/stack.txt` the full chain for `/` and `/boot` (disk → partition → LVM? → filesystem type → mount point).

<details>
<summary>Solution</summary>

```bash
lsblk -f > /root/stack.txt
findmnt --real -o TARGET,SOURCE,FSTYPE >> /root/stack.txt
pvs >> /root/stack.txt 2>/dev/null; lvs >> /root/stack.txt 2>/dev/null
```

</details>

---

## Exercise 2 – Supported vs used filesystems ★

1. List the filesystem types the kernel supports and mark which don't need a device.
2. List the distinct filesystem types currently mounted.

<details>
<summary>Solution</summary>

```bash
cat /proc/filesystems
findmnt -n -o FSTYPE | sort -u
```

</details>

---

## Exercise 3 – Identify unknown devices ★★

On an extra disk, create two loop images: one formatted ext4, one XFS. Without mounting them, identify their filesystem types, UUIDs and labels in three different ways.

<details>
<summary>Solution</summary>

```bash
truncate -s 300M /root/a.img /root/b.img
mkfs.ext4 -q -L EXTDISK /root/a.img; mkfs.xfs -q -L XFSDISK /root/b.img
A=$(losetup -f --show /root/a.img); B=$(losetup -f --show /root/b.img)
blkid $A $B
lsblk -f $A $B
file -s $A $B
```

</details>

---

## Exercise 4 – Compare ext4 and XFS ★★

Mount both images (`/mnt/ext`, `/mnt/xfs`). For each, find: block size, total inodes, reserved space (ext4), and whether inode count changes when you create many files.

<details>
<summary>Solution</summary>

```bash
mkdir -p /mnt/ext /mnt/xfs; mount $A /mnt/ext; mount $B /mnt/xfs
stat -f /mnt/ext /mnt/xfs
df -i /mnt/ext /mnt/xfs
tune2fs -l $A | grep -E 'Block size|Inode count|Reserved block count'
xfs_info /mnt/xfs
for d in /mnt/ext /mnt/xfs; do mkdir $d/many; for i in $(seq 5000); do : > $d/many/f$i; done; done
df -i /mnt/ext /mnt/xfs     # ext4 total fixed; XFS total may grow dynamically
```

</details>

---

## Exercise 5 – tmpfs ★

Mount a 100 MiB tmpfs on `/mnt/ram`, copy files into it until full, check `free -h` (shared/buff/cache) before and after, then unmount. Where did the data go?

<details>
<summary>Solution</summary>

```bash
mkdir -p /mnt/ram && mount -t tmpfs -o size=100M tmpfs /mnt/ram
free -h; dd if=/dev/urandom of=/mnt/ram/fill bs=1M count=200; free -h; df -h /mnt/ram
umount /mnt/ram     # data lost: tmpfs lives only in RAM/swap
```

</details>

---

## Challenge – Journal replay ★★★

On the ext4 image: mount it, write a file, then simulate a crash by detaching the loop device while mounted is not safe – instead, use `tune2fs -l` to look at the "Filesystem state" before and after a clean `umount`, and use `dumpe2fs -h` to find the journal size and features. Explain in `/root/journal.txt` what `has_journal` and `needs_recovery` mean.

<details>
<summary>Solution</summary>

```bash
dumpe2fs -h $A 2>/dev/null | grep -E 'features|state|Journal size'
```

`has_journal` means the filesystem has a journal; `needs_recovery` appears while it is mounted (or after a crash) and tells the next mount to replay the journal.

</details>

---

## Cleanup

```bash
umount /mnt/ext /mnt/xfs 2>/dev/null; losetup -d $A $B 2>/dev/null; rm -f /root/a.img /root/b.img
```

---

<!-- footer:start -->
[Module exercises](README.md) · [Exercises: Disk Partitioning →](02-partitioning.md)
<!-- footer:end -->
