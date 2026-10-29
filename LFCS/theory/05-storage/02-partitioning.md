# Disk Partitioning

<!-- nav:start -->
> **Module 05 · Storage** — lesson 2 of 9  
> [Module index](README.md) · [Course lesson (commands)](../../course/05-Storage.md#lesson-2---partitioning-ch-20) · [Exercises for this lesson](../../exercises/05-storage/02-partitioning.md)
<!-- nav:end -->

Before a new disk can hold filesystems, swap or LVM, it is usually divided into **partitions**. In this lesson you will learn how Linux names disks, the two partition table formats (MBR and GPT), partition types, and how to create, inspect and delete partitions with `fdisk`, `gdisk` and `parted` – interactively and in scripts.

## What you will learn

- Disk and partition device names
- MBR vs GPT: limits, structure and when to use each
- Partition types (Linux filesystem, swap, LVM, EFI)
- Partitioning with `fdisk`, `gdisk` and `parted`
- Making the kernel see a new partition table
- Backing up and restoring a partition table
- Alignment and sizing best practices

---

## Key terms

| Term | Definition |
|------|-----------|
| **Partition** | A contiguous region of a disk treated as an independent block device. |
| **Partition table** | The structure at the start (and for GPT also the end) of a disk describing its partitions. |
| **MBR** | *Master Boot Record* (also called "msdos"/DOS label): the legacy partition table in the first 512-byte sector. |
| **GPT** | *GUID Partition Table*: the modern standard, part of UEFI. |
| **Primary / extended / logical** | MBR only: up to 4 primary partitions, or 3 primary + 1 extended container holding logical partitions. |
| **Partition type** | A code telling the OS what the partition is meant for (Linux, swap, LVM, EFI). |
| **Sector** | The smallest addressable unit of a disk: 512 bytes (logical), often 4096 physical. |
| **Alignment** | Starting partitions on boundaries (usually 1 MiB) that match the physical layout for best performance. |

---

## Device names

| Device | Name | Partitions |
|--------|------|-----------|
| SATA/SAS/USB/SCSI and many virtual disks | `/dev/sda`, `/dev/sdb` … | `/dev/sda1`, `/dev/sda2` |
| virtio disks (KVM) | `/dev/vda`, `/dev/vdb` | `/dev/vda1` |
| NVMe | `/dev/nvme0n1` (controller 0, namespace 1) | `/dev/nvme0n1p1` (note the `p`) |
| MMC/SD | `/dev/mmcblk0` | `/dev/mmcblk0p1` |
| Loop devices | `/dev/loop0` | `/dev/loop0p1` |

> Names are assigned in detection order and **can change** between boots (add a disk and `sdb` may become `sdc`). Always check with `lsblk` before partitioning, and use UUIDs in `/etc/fstab`.

```console
$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda    252:0    0   20G  0 disk
├─vda1 252:1    0    1G  0 part /boot
└─vda2 252:2    0   19G  0 part /
vdb    252:16   0    5G  0 disk
vdc    252:32   0    5G  0 disk
$ sudo fdisk -l /dev/vdb
Disk /dev/vdb: 5 GiB, 5368709120 bytes, 10485760 sectors
Units: sectors of 1 * 512 = 512 bytes
```

`vdb` and `vdc` are empty – our practice disks.

---

## MBR vs GPT

| | MBR (msdos) | GPT |
|--|-------------|-----|
| Location | sector 0 | sector 1 (header + entries), plus **backup copy** at the end of the disk |
| Max disk size | **2 TiB** (with 512-byte sectors) | 8 ZiB |
| Number of partitions | 4 primary, or 3 primary + extended with logicals | 128 by default |
| Integrity | none | CRC32 checksums, backup header |
| Partition IDs | 1-byte type (`83` Linux, `82` swap, `8e` LVM) | 128-bit type GUID; also unique partition GUID (PARTUUID) and a name |
| Boot firmware | BIOS | UEFI (and BIOS with a small `bios_grub` partition) |

**Use GPT** for every new disk unless something old requires MBR.

### MBR layout

```text
Sector 0: [ boot code 446 B | partition table 4 × 16 B | 0x55AA ]
Primary 1 | Primary 2 | Primary 3 | Extended ─┬ Logical 5
                                              ├ Logical 6
                                              └ Logical 7
```

### GPT layout

```text
LBA 0      LBA 1        LBA 2–33              …data…            last 33 LBAs
[protective MBR][GPT header][partition entries][ partitions ][backup entries][backup header]
```

The *protective MBR* makes old tools see the disk as "full" so they don't overwrite it.

---

## Partition types

| Purpose | fdisk (GPT) type name | gdisk code | MBR code |
|---------|----------------------|-----------|----------|
| Linux filesystem | Linux filesystem | `8300` | `83` |
| Swap | Linux swap | `8200` | `82` |
| LVM physical volume | Linux LVM | `8e00` | `8e` |
| Software RAID | Linux RAID | `fd00` | `fd` |
| EFI System Partition | EFI System | `ef00` | `ef` |
| BIOS boot (GRUB on GPT+BIOS) | BIOS boot | `ef02` | – |

The type is only a **label**. Setting "Linux swap" does not create swap – you still run `mkswap`; setting "Linux LVM" doesn't create a PV – you still run `pvcreate`.

---

## fdisk: interactive partitioning

`fdisk` handles MBR and GPT. Nothing is written until you press `w`.

```console
$ sudo fdisk /dev/vdb

Command (m for help): g                     ← create a new empty GPT table (o = MBR)
Created a new GPT disklabel (GUID: 5B9A…).

Command (m for help): n                     ← new partition
Partition number (1-128, default 1): ⏎
First sector (2048-10485726, default 2048): ⏎
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-10485726, default 10483711): +1G
Created a new partition 1 of type 'Linux filesystem' and of size 1 GiB.

Command (m for help): n
Partition number (2-128, default 2): ⏎
First sector (2099200-10485726, default 2099200): ⏎
Last sector, +/-sectors or +/-size{K,M,G,T,P} (…): +512M

Command (m for help): t                     ← change type
Partition number (1,2, default 2): 2
Partition type or alias (type L to list all): swap
Changed type of partition 'Linux filesystem' to 'Linux swap'.

Command (m for help): p                     ← print
Device       Start     End Sectors  Size Type
/dev/vdb1     2048 2099199 2097152    1G Linux filesystem
/dev/vdb2  2099200 3147775 1048576  512M Linux swap

Command (m for help): w                     ← write and exit
The partition table has been altered.
Syncing disks.
```

| Key | Action |
|-----|--------|
| `m` | help |
| `p` | print the table |
| `g` / `o` | new GPT / MBR table (erases all entries) |
| `n` | new partition |
| `d` | delete a partition |
| `t` | change type (`L` lists types) |
| `w` | write and quit |
| `q` | quit **without** saving |

---

## gdisk

`gdisk` is the GPT-specific equivalent with the same command letters and 4-digit type codes (`8300`, `8200`, `8e00`). Useful extras: `sgdisk` for scripting, converting MBR → GPT.

```console
$ sudo sgdisk -n 1:0:+1G -t 1:8300 -c 1:data /dev/vdc     # partition 1, 1 GiB, Linux, name "data"
$ sudo sgdisk -p /dev/vdc
```

---

## parted: scriptable partitioning

`parted` works with sizes and percentages and is easy to use non-interactively. **Careful:** parted writes changes **immediately** – there is no "w" step.

```console
$ sudo parted /dev/vdc --script mklabel gpt
$ sudo parted /dev/vdc --script mkpart data ext4 1MiB 2GiB
$ sudo parted /dev/vdc --script mkpart lvm 2GiB 100%
$ sudo parted /dev/vdc --script set 2 lvm on
$ sudo parted /dev/vdc print
Number  Start   End     Size    File system  Name  Flags
 1      1049kB  2147MB  2146MB               data
 2      2147MB  5369MB  3222MB               lvm   lvm
```

The `ext4` word in `mkpart` only sets the type GUID – it does **not** format the partition. `parted` can also grow a partition: `resizepart 1 3GiB`.

---

## Making the kernel see changes

After writing a table, the kernel usually re-reads it. If the disk is in use, it may keep the old view:

```console
$ sudo partprobe /dev/vdb       # ask the kernel to re-read
$ sudo partx -u /dev/vdb        # alternative
$ lsblk /dev/vdb
$ cat /proc/partitions
```

---

## Backing up and restoring a partition table

```console
$ sudo sfdisk -d /dev/vdb > /root/vdb.layout        # dump (MBR and GPT)
$ sudo sfdisk /dev/vdb < /root/vdb.layout           # restore / clone to another disk
$ sudo sgdisk --backup=/root/vdb.gpt /dev/vdb       # GPT binary backup
$ sudo sgdisk --load-backup=/root/vdb.gpt /dev/vdb
```

`sfdisk` also creates partitions from a script:

```console
$ echo -e 'label: gpt\n,1G,L\n,512M,S\n,,V' | sudo sfdisk /dev/vdc   # 1G Linux, 512M swap, rest LVM
```

### Wiping

```console
$ sudo wipefs -a /dev/vdc           # remove partition table and filesystem signatures
$ sudo sgdisk --zap-all /dev/vdc    # destroy GPT and MBR structures
```

---

## Best practices

- **Check twice which disk you're on** (`lsblk`, size, serial with `lsblk -o NAME,SIZE,SERIAL`). Partitioning the wrong disk destroys data.
- Start the first partition at **1 MiB** (sector 2048) – the default in all modern tools – for alignment.
- Prefer GPT. Use LVM on top for flexibility (resizing partitions later is painful).
- Leave room or use LVM if you might need more filesystems later.

---

## Try it yourself

1. On your first extra disk, create a GPT table with a 1 GiB Linux partition and a 512 MiB swap partition using `fdisk`.
2. On the second extra disk, do the same with `parted --script`; add a third partition with the rest of the disk flagged for LVM.
3. Back up the table of the first disk with `sfdisk -d`, wipe it with `wipefs -a`, and restore it.
4. Create an MBR table with 3 primary and 2 logical partitions on a loop device (`truncate -s 1G /root/disk.img; losetup -fP --show /root/disk.img`).

---

## Common mistakes

- Partitioning the wrong device (names changed, or you typed `vda`).
- Forgetting `w` in fdisk (nothing saved) – or forgetting that parted saves immediately.
- Believing the partition type formats anything.
- Using MBR on a disk larger than 2 TiB.
- Not running `partprobe` and getting "device not found" for the new partition.

---

## Summary

| Task | Command |
|------|---------|
| List disks and partitions | `lsblk`, `fdisk -l`, `parted -l` |
| Interactive (MBR/GPT) | `fdisk /dev/X` (`g`, `n`, `t`, `p`, `w`) |
| GPT-specific | `gdisk`, `sgdisk` |
| Scripted | `parted --script`, `sfdisk`, `sgdisk` |
| Re-read table | `partprobe`, `partx -u` |
| Backup / restore | `sfdisk -d > f` / `sfdisk < f` |
| Wipe | `wipefs -a` |

## Check yourself

1. What is the maximum disk size with MBR?
2. How many partitions can GPT hold by default?
3. You set a partition's type to "Linux swap". Can you `swapon` it immediately?
4. What's the partition device name of the 2nd partition of the first NVMe disk?

<details>
<summary>Answers</summary>

1. 2 TiB (with 512-byte sectors).
2. 128.
3. No – you must run `mkswap` first; the type is only a label.
4. `/dev/nvme0n1p2`.

</details>

## Further reading

- `man 8 fdisk`, `man 8 gdisk`, `man 8 sgdisk`, `man 8 parted`, `man 8 sfdisk`, `man 8 wipefs`
- [ArchWiki: Partitioning](https://wiki.archlinux.org/title/Partitioning) · [fdisk](https://wiki.archlinux.org/title/Fdisk) · [GPT fdisk](https://wiki.archlinux.org/title/GPT_fdisk)
- [Red Hat: Disk partitions (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_storage_devices/disk-partitions_managing-storage-devices)

---

<!-- footer:start -->
[← The VFS and Filesystem Types](01-vfs-filesystem-types.md) · [Module index](README.md) · [Creating, Mounting and Repairing Filesystems →](03-filesystems-mounting.md)  
**Practice now:** [Exercises – Disk Partitioning](../../exercises/05-storage/02-partitioning.md)
<!-- footer:end -->
