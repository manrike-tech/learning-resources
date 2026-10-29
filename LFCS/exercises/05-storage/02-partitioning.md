# Exercises – Disk Partitioning

<!-- nav:start -->
> **Module 05 · Storage** — lesson 2 of 9  
> Read first: [Theory – Disk Partitioning](../../theory/05-storage/02-partitioning.md) · [Course lesson (commands)](../../course/05-Storage.md#lesson-2---partitioning-ch-20) · [Module exercises](README.md)
<!-- nav:end -->

> Use the extra disks (`/dev/vdb`, `/dev/vdc` or `/dev/sdb`, `/dev/sdc`). Check with `lsblk` **every time** before writing. Examples use `vdb`/`vdc`.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Inspect ★

Record in `/root/disks.txt` the size, model/serial and current partition table type of each disk.

<details>
<summary>Solution</summary>

```bash
lsblk -d -o NAME,SIZE,MODEL,SERIAL,PTTYPE > /root/disks.txt
fdisk -l /dev/vdb /dev/vdc >> /root/disks.txt
```

</details>

---

## Exercise 2 – fdisk with GPT ★★

On `vdb`, with `fdisk`, create a GPT table with:
1. 1 GiB "Linux filesystem",
2. 512 MiB "Linux swap",
3. the rest as "Linux LVM".

**Verify:** `lsblk /dev/vdb` shows three partitions; `fdisk -l /dev/vdb` shows the types.

<details>
<summary>Solution</summary>

`fdisk /dev/vdb` → `g`, `n` ⏎ ⏎ `+1G`, `n` ⏎ ⏎ `+512M`, `t` `2` `swap`, `n` ⏎ ⏎ ⏎, `t` `3` `lvm`, `p`, `w`. Then `partprobe /dev/vdb`.

</details>

---

## Exercise 3 – parted, scripted ★★

On `vdc`, using only `parted --script`, create a GPT table with partition `data` (1 MiB–2 GiB) and partition `lvm` (2 GiB–100%) with the lvm flag.

<details>
<summary>Solution</summary>

```bash
parted /dev/vdc --script mklabel gpt mkpart data ext4 1MiB 2GiB mkpart lvm 2GiB 100% set 2 lvm on
parted /dev/vdc print
```

</details>

---

## Exercise 4 – MBR with logical partitions ★★

On a 1 GiB loop disk, create an MBR table with 3 primary partitions of 100 MiB, an extended partition with the rest, and 2 logical partitions of 100 MiB inside it.

<details>
<summary>Solution</summary>

```bash
truncate -s 1G /root/mbr.img; L=$(losetup -fP --show /root/mbr.img)
printf 'o\nn\np\n1\n\n+100M\nn\np\n2\n\n+100M\nn\np\n3\n\n+100M\nn\ne\n\n\nn\n\n+100M\nn\n\n+100M\nw\n' | fdisk $L
fdisk -l $L      # p1-p3 primary, p4 extended, p5-p6 logical
losetup -d $L
```

</details>

---

## Exercise 5 – Backup, wipe, restore ★★

Back up the partition table of `vdb`, wipe all signatures, confirm `lsblk` shows no partitions, restore it.

<details>
<summary>Solution</summary>

```bash
sfdisk -d /dev/vdb > /root/vdb.layout
wipefs -a /dev/vdb; lsblk /dev/vdb
sfdisk /dev/vdb < /root/vdb.layout; lsblk /dev/vdb
```

</details>

---

## Exercise 6 – sfdisk one-liner ★★★

Recreate the layout of Exercise 2 on a fresh loop disk with a single `sfdisk` command (heredoc), no interactive tools.

<details>
<summary>Solution</summary>

```bash
truncate -s 5G /root/sf.img; L=$(losetup -fP --show /root/sf.img)
sfdisk $L << 'EOF'
label: gpt
,1G,L
,512M,S
,,V
EOF
lsblk $L; losetup -d $L; rm /root/sf.img
```

(`L` Linux filesystem, `S` swap, `V` LVM.)

</details>

---

## Challenge – Grow a partition ★★★

On a loop disk with one 500 MiB ext4 partition containing data, grow the partition to 900 MiB and the filesystem to match, without losing data.

<details>
<summary>Solution</summary>

```bash
truncate -s 1G /root/g.img; L=$(losetup -fP --show /root/g.img)
parted $L --script mklabel gpt mkpart p 1MiB 501MiB
mkfs.ext4 -q ${L}p1; mkdir -p /mnt/g; mount ${L}p1 /mnt/g; echo keep > /mnt/g/f
parted $L --script resizepart 1 901MiB
partprobe $L; resize2fs ${L}p1; df -h /mnt/g; cat /mnt/g/f
umount /mnt/g; losetup -d $L; rm /root/g.img
```

This is why LVM is preferred – growing partitions only works when free space follows the partition.

</details>

---

<!-- footer:start -->
[← Exercises: The VFS and Filesystem Types](01-vfs-filesystem-types.md) · [Module exercises](README.md) · [Exercises: Creating, Mounting and Repairing Filesystems →](03-filesystems-mounting.md)
<!-- footer:end -->
