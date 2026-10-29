# Module 05 - Storage

**LFS207:** Ch. 16 (swap) · Ch. 19 Linux Filesystems and the VFS · Ch. 20 Disk Partitioning · Ch. 21 Filesystem Features · Ch. 22 The EXT4 Filesystem · Ch. 23 LVM
**Exam domain:** Storage (20%)  
**Theory:** [Module 05 theory notes](../theory/05-storage/README.md) · **Exercises:** [Module 05 exercises](../exercises/05-storage/README.md)  
**Gaps covered:** NFS, network block devices, automounters, storage performance

---

## Learning Objectives

- Partition disks (MBR/GPT) and create filesystems.
- Mount persistently and troubleshoot `/etc/fstab`.
- Build and resize LVM storage.
- Add and manage swap.
- Use NFS, NBD and autofs.

> Use the **two extra 5 GB disks** on each VM (`/dev/vdb`, `/dev/vdc` or `/dev/sdb`, `/dev/sdc`). Check names with `lsblk`.

---

## Lesson 1 - The VFS & Filesystem Types (Ch. 19)

> **Theory:** [The VFS and Filesystem Types](../theory/05-storage/01-vfs-filesystem-types.md) · **Exercises:** [practice tasks](../exercises/05-storage/01-vfs-filesystem-types.md)

The **Virtual File System** is the kernel layer that gives every filesystem (ext4, xfs, btrfs, nfs, tmpfs, proc) the same interface (`open`, `read`, `write`).

| FS | Notes |
|----|-------|
| ext4 | Default Debian/Ubuntu. Grow **and shrink** (offline). |
| xfs | Default RHEL. Grow only (online), **cannot shrink**. |
| btrfs | Copy-on-write, snapshots, subvolumes (openSUSE default). |
| tmpfs | RAM-backed. |
| vfat | EFI system partition, USB. |

```bash
cat /proc/filesystems; lsblk -f; blkid; findmnt; mount | column -t
```

---

## Lesson 2 - Partitioning (Ch. 20)

> **Theory:** [Disk Partitioning](../theory/05-storage/02-partitioning.md) · **Exercises:** [practice tasks](../exercises/05-storage/02-partitioning.md)

```bash
lsblk
fdisk /dev/vdb      # MBR or GPT, interactive: n, p, t, w
gdisk /dev/vdb      # GPT
parted /dev/vdb --script mklabel gpt mkpart data ext4 1MiB 2GiB
partprobe /dev/vdb  # make the kernel re-read the table
```

Partition type codes (fdisk, GPT): **Linux filesystem**, **Linux swap**, **Linux LVM**.
MBR: max 4 primary partitions, 2 TiB limit; GPT: 128 partitions, huge disks.

---

## Lesson 3 - Creating, Checking & Mounting (Ch. 21–22)

> **Theory:** [Creating, Mounting and Repairing Filesystems](../theory/05-storage/03-filesystems-mounting.md) · **Exercises:** [practice tasks](../exercises/05-storage/03-filesystems-mounting.md)

```bash
mkfs.ext4 -L data /dev/vdb1
mkfs.xfs /dev/vdb2
mkdir -p /data && mount /dev/vdb1 /data
umount /data      # "target is busy" → lsof /data or fuser -vm /data
```

### Persistent mounts - `/etc/fstab`

```
# device                                    mountpoint  type  options          dump fsck
UUID=3f1c...                                /data       ext4  defaults         0    2
/dev/vg01/lv_web                            /var/www    xfs   defaults,noatime 0    0
UUID=9a2b...                                none        swap  defaults         0    0
server1:/exports/share                      /mnt/nfs    nfs   defaults,_netdev 0    0
```

```bash
blkid /dev/vdb1           # get UUID
findmnt --verify          # check fstab syntax
mount -a                  # mount everything in fstab - ALWAYS test before reboot
systemctl daemon-reload   # systemd regenerates mount units from fstab
```

> **Exam trap:** a typo in `/etc/fstab` can make the system boot into emergency mode. Always run `mount -a` (and `findmnt --verify`) after editing. Use `nofail` for non-critical mounts.

### Check & repair (unmounted!)

```bash
fsck.ext4 -f /dev/vdb1;  e2fsck -fy /dev/vdb1
xfs_repair /dev/vdb2
tune2fs -l /dev/vdb1     # ext4 info; tune2fs -L newlabel, -c, -i
dumpe2fs /dev/vdb1 | less
xfs_info /data
```

### Attributes

```bash
chattr +i file; chattr +a logfile; lsattr
```

---

## Lesson 4 - LVM (Ch. 23)

> **Theory:** [Logical Volume Management (LVM)](../theory/05-storage/04-lvm.md) · **Exercises:** [practice tasks](../exercises/05-storage/04-lvm.md)

```
Disks/partitions → PV (pvcreate) → VG (vgcreate) → LV (lvcreate) → filesystem
```

```bash
pvcreate /dev/vdb /dev/vdc
vgcreate vg01 /dev/vdb
vgextend vg01 /dev/vdc
lvcreate -n lv_data -L 2G vg01
lvcreate -n lv_logs -l 100%FREE vg01
mkfs.xfs /dev/vg01/lv_data

pvs; vgs; lvs; pvdisplay; vgdisplay; lvdisplay

# Grow LV AND filesystem in one step
lvextend -r -L +1G /dev/vg01/lv_data
# or separately:
lvextend -L +1G /dev/vg01/lv_data
xfs_growfs /data              # xfs (mounted)
resize2fs /dev/vg01/lv_data   # ext4

# Shrink (ext4 only, unmount first)
umount /data; lvreduce -r -L 1G /dev/vg01/lv_data

# Snapshots
lvcreate -s -n snap_data -L 500M /dev/vg01/lv_data

# Remove a disk from a VG
pvmove /dev/vdb; vgreduce vg01 /dev/vdb; pvremove /dev/vdb

lvremove /dev/vg01/lv_logs; vgremove vg01
```

---

## Lesson 5 - Swap (Ch. 16)

> **Theory:** [Swap Space](../theory/05-storage/05-swap.md) · **Exercises:** [practice tasks](../exercises/05-storage/05-swap.md)

```bash
swapon --show; free -h
# Partition or LV
mkswap /dev/vg01/lv_swap && swapon /dev/vg01/lv_swap
# Swap file
fallocate -l 1G /swapfile     # (use dd on filesystems where fallocate isn't supported for swap)
chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile
echo '/swapfile none swap defaults 0 0' >> /etc/fstab
swapoff /swapfile
sysctl vm.swappiness          # 0-100 (tendency to swap)
```

---

## Lesson 6 - NFS (Gap)

> **Theory:** [NFS: Network File System](../theory/05-storage/06-nfs.md) · **Exercises:** [practice tasks](../exercises/05-storage/06-nfs.md)

**Server:**
```bash
apt install nfs-kernel-server     # Debian
dnf install nfs-utils             # RHEL
mkdir -p /exports/share
```
```
# /etc/exports
/exports/share  192.168.56.0/24(rw,sync,no_root_squash)
/exports/ro     *(ro,sync)
```
```bash
exportfs -arv; exportfs -v
systemctl enable --now nfs-server     # Debian: nfs-kernel-server
firewall-cmd --add-service={nfs,mountd,rpc-bind} --permanent && firewall-cmd --reload
```

**Client:**
```bash
apt install nfs-common   |  dnf install nfs-utils
showmount -e server1
mount -t nfs server1:/exports/share /mnt/nfs
# fstab: server1:/exports/share /mnt/nfs nfs defaults,_netdev 0 0
```

---

## Lesson 7 - Network Block Devices (Gap)

> **Theory:** [Network Block Devices and iSCSI](../theory/05-storage/07-network-block-devices.md) · **Exercises:** [practice tasks](../exercises/05-storage/07-network-block-devices.md)

NBD exports a **block device** (not files) over the network; the client sees `/dev/nbd0`.

**Server:**
```bash
apt install nbd-server
```
```ini
# /etc/nbd-server/config
[generic]
[disk1]
exportname = /dev/vg01/lv_nbd
```
```bash
systemctl restart nbd-server
```

**Client:**
```bash
apt install nbd-client      # RHEL-family: package may be in EPEL
modprobe nbd
nbd-client server1 -N disk1 /dev/nbd0
lsblk /dev/nbd0; mkfs.ext4 /dev/nbd0; mount /dev/nbd0 /mnt/nbd
nbd-client -d /dev/nbd0     # disconnect
```

> Know the concept of **iSCSI** too (target/initiator, `iscsiadm`): same idea, enterprise-grade.

---

## Lesson 8 - Automounters (Gap)

> **Theory:** [Filesystem Automounters](../theory/05-storage/08-automounters.md) · **Exercises:** [practice tasks](../exercises/05-storage/08-automounters.md)

### autofs

```bash
apt install autofs | dnf install autofs
```
```
# /etc/auto.master  (or a file in /etc/auto.master.d/*.autofs)
/mnt/auto   /etc/auto.nfs   --timeout=60
```
```
# /etc/auto.nfs
share   -rw,sync   server1:/exports/share
*       -rw        server1:/exports/home/&      # wildcard: key & substituted
```
```bash
systemctl enable --now autofs
ls /mnt/auto/share       # mounts on access
```

### systemd automount (via fstab)

```
server1:/exports/share /mnt/nfs nfs noauto,x-systemd.automount,x-systemd.idle-timeout=60,_netdev 0 0
```

---

## Lesson 9 - Storage Performance (Gap)

> **Theory:** [Monitoring Storage Performance](../theory/05-storage/09-storage-performance.md) · **Exercises:** [practice tasks](../exercises/05-storage/09-storage-performance.md)

```bash
iostat -xz 1      # %util, await, r/s, w/s per device
iotop -o          # processes doing I/O
vmstat 1          # wa column = CPU waiting on I/O
pidstat -d 1
cat /sys/block/vda/queue/scheduler
```

High `await` + `%util` near 100% → device saturated.

---

## Lab 05

1. On disk 1, create a GPT partition of 1 GiB, format ext4 with label `DATA`, mount at `/data` persistently using UUID.
2. Create VG `vgapp` from disk 2, LV `lvweb` of 1 GiB with xfs mounted at `/var/www`. Extend it by 500 MiB online.
3. Add a 512 MiB swap LV and enable it permanently.
4. Export `/exports/share` from `rhel1`; mount it on `deb1` with autofs at `/mnt/auto/share`.
5. Export an LV via NBD from `deb1`; attach it on the other VM, format and mount.
6. Break `/etc/fstab` on purpose (wrong UUID), reboot, fix it from emergency mode.

## Self-check

- [ ] I can go from raw disk to a mounted, persistent LV in under 5 minutes.
- [ ] I know which filesystems can shrink.
- [ ] I always run `mount -a` after editing fstab.
- [ ] I can configure NFS server + client and autofs.
- [ ] I can attach a network block device.
