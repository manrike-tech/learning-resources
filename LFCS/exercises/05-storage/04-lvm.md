# Exercises – Logical Volume Management

<!-- nav:start -->
> **Module 05 · Storage** — lesson 4 of 9  
> Read first: [Theory – Logical Volume Management (LVM)](../../theory/05-storage/04-lvm.md) · [Course lesson (commands)](../../course/05-Storage.md#lesson-4---lvm-ch-23) · [Module exercises](README.md)
<!-- nav:end -->

> Start with the two extra disks **empty** (`wipefs -a /dev/vdb /dev/vdc` after unmounting anything on them and removing their fstab lines).

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Build the stack ★

1. Make `vdb` a PV and create VG `vgapp` with it.
2. Create LV `lvweb` of 1 GiB, format XFS, mount persistently at `/var/www`.

**Verify:** `lvs`, `findmnt /var/www`, `grep www /etc/fstab`.

<details>
<summary>Solution</summary>

```bash
pvcreate /dev/vdb
vgcreate vgapp /dev/vdb
lvcreate -n lvweb -L 1G vgapp
mkfs.xfs /dev/vgapp/lvweb
mkdir -p /var/www
echo "/dev/vgapp/lvweb /var/www xfs defaults 0 0" >> /etc/fstab
systemctl daemon-reload && mount -a
```

</details>

---

## Exercise 2 – Inspect ★

Record the VG's extent size, total and free extents, and which PV `lvweb` uses.

<details>
<summary>Solution</summary>

```bash
vgdisplay vgapp | grep -E 'PE Size|Total PE|Free  PE'
lvs -o +devices vgapp
```

</details>

---

## Exercise 3 – Grow online ★

Grow `lvweb` by 500 MiB while mounted, filesystem included, in one command.

**Verify:** `df -h /var/www` shows ~1.5 GiB.

<details>
<summary>Solution</summary>

```bash
lvextend -r -L +500M /dev/vgapp/lvweb
```

</details>

---

## Exercise 4 – Forgetting the filesystem ★★

Run `lvextend -L +200M /dev/vgapp/lvweb` (no `-r`). Check `lvs` and `df`. Fix the mismatch.

<details>
<summary>Solution</summary>

```bash
lvextend -L +200M /dev/vgapp/lvweb
lvs; df -h /var/www          # LV bigger, filesystem not
xfs_growfs /var/www
```

</details>

---

## Exercise 5 – Add a disk ★★

Add `vdc` to `vgapp` and give all remaining free space to `lvweb`.

<details>
<summary>Solution</summary>

```bash
pvcreate /dev/vdc; vgextend vgapp /dev/vdc
lvextend -r -l +100%FREE /dev/vgapp/lvweb
vgs; lvs -o +devices
```

</details>

---

## Exercise 6 – Shrink ext4 ★★

1. Try to shrink `lvweb` back to 2 GiB – is it possible? Why not?
2. Remove `lvweb` (unmount it and delete its fstab line). Create LV `lvdata` of 2 GiB with ext4, mounted on `/data2`, put 100 MiB of data on it, then shrink it to 1 GiB keeping the data.

<details>
<summary>Solution</summary>

1. No: XFS cannot be shrunk.

```bash
umount /var/www; sed -i '\|/var/www|d' /etc/fstab; lvremove -y /dev/vgapp/lvweb
lvcreate -n lvdata -L 2G vgapp; mkfs.ext4 -q /dev/vgapp/lvdata
mkdir -p /data2; mount /dev/vgapp/lvdata /data2
dd if=/dev/urandom of=/data2/blob bs=1M count=100; md5sum /data2/blob > /root/blob.md5
umount /data2
lvreduce -r -L 1G /dev/vgapp/lvdata
mount /dev/vgapp/lvdata /data2; md5sum -c /root/blob.md5
```

</details>

---

## Exercise 7 – Snapshot and restore a file ★★

1. Create a 200 MiB snapshot of `lvdata`.
2. Delete `/data2/blob`.
3. Mount the snapshot read-only and copy the file back. Remove the snapshot.

<details>
<summary>Solution</summary>

```bash
lvcreate -s -n snapdata -L 200M /dev/vgapp/lvdata
rm /data2/blob
mkdir -p /mnt/snap && mount -o ro /dev/vgapp/snapdata /mnt/snap
cp /mnt/snap/blob /data2/; md5sum -c /root/blob.md5
umount /mnt/snap; lvremove -y /dev/vgapp/snapdata
```

</details>

---

## Exercise 8 – Roll back with merge ★★★

Take a snapshot, then delete **all** files in `/data2`, and roll the whole LV back to the snapshot with `lvconvert --merge`.

<details>
<summary>Solution</summary>

```bash
lvcreate -s -n snap2 -L 200M /dev/vgapp/lvdata
rm -rf /data2/*
umount /data2
lvconvert --merge /dev/vgapp/snap2
mount /dev/vgapp/lvdata /data2; ls /data2
```

</details>

---

## Exercise 9 – Evacuate a disk ★★★

Make sure `lvdata` has extents on `vdb`. Move everything off `vdb` while `/data2` stays mounted, then remove `vdb` from the VG and from LVM.

**Verify:** `pvs` no longer lists `vdb` in `vgapp`; `md5sum -c /root/blob.md5` passes.

<details>
<summary>Solution</summary>

```bash
lvs -o +devices vgapp
pvmove /dev/vdb
vgreduce vgapp /dev/vdb
pvremove /dev/vdb
md5sum -c /root/blob.md5
```

</details>

---

## Challenge – Exam-style ★★★

> Create VG `vgdata` with 8 MiB extents from disk 1. Create LV `lvapp` of exactly 50 extents, formatted ext4, mounted persistently at `/app`. Then grow it by 512 MiB. Finally create a 256 MiB swap LV `lvswap` in the same VG and enable it permanently.

<details>
<summary>Solution</summary>

```bash
pvcreate /dev/vdb; vgcreate -s 8M vgdata /dev/vdb
lvcreate -n lvapp -l 50 vgdata              # 400 MiB
mkfs.ext4 /dev/vgdata/lvapp; mkdir -p /app
echo "/dev/vgdata/lvapp /app ext4 defaults 0 2" >> /etc/fstab
systemctl daemon-reload; mount -a
lvextend -r -L +512M /dev/vgdata/lvapp
lvcreate -n lvswap -L 256M vgdata; mkswap /dev/vgdata/lvswap
echo "/dev/vgdata/lvswap none swap defaults 0 0" >> /etc/fstab; swapon -a; swapon --show
```

</details>

---

<!-- footer:start -->
[← Exercises: Creating, Mounting and Repairing Filesystems](03-filesystems-mounting.md) · [Module exercises](README.md) · [Exercises: Swap Space →](05-swap.md)
<!-- footer:end -->
