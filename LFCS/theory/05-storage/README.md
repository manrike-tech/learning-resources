# Module 05 – Storage: Theory

> [All theory](../00-Theory-Index.md) · [Course module (commands & lab)](../../course/05-Storage.md) · [Module exercises](../../exercises/05-storage/README.md)

Read the lessons in order. Each one explains the concepts with definitions and worked examples, then links to its exercises.

| # | Lesson | About | Practice |
|---|--------|----------------|----------|
| 1 | [The VFS and Filesystem Types](01-vfs-filesystem-types.md) | Linux supports dozens of filesystems – ext4, XFS, Btrfs, NFS, tmpfs, vfat – yet every program uses the same `open()`, `read()` and `write()` calls on all of them. | [Exercises](../../exercises/05-storage/01-vfs-filesystem-types.md) |
| 2 | [Disk Partitioning](02-partitioning.md) | Before a new disk can hold filesystems, swap or LVM, it is usually divided into **partitions**. | [Exercises](../../exercises/05-storage/02-partitioning.md) |
| 3 | [Creating, Mounting and Repairing Filesystems](03-filesystems-mounting.md) | A partition or logical volume is just raw space until you put a **filesystem** on it and **mount** it into the directory tree. | [Exercises](../../exercises/05-storage/03-filesystems-mounting.md) |
| 4 | [Logical Volume Management (LVM)](04-lvm.md) | With plain partitions, storage is rigid: growing a full partition means moving the ones after it, and a filesystem can't span two disks. | [Exercises](../../exercises/05-storage/04-lvm.md) |
| 5 | [Swap Space](05-swap.md) | **Swap** is disk space the kernel uses as an extension of RAM. | [Exercises](../../exercises/05-storage/05-swap.md) |
| 6 | [NFS: Network File System](06-nfs.md) | **NFS** lets a server share directories over the network so that clients mount them like local filesystems. | [Exercises](../../exercises/05-storage/06-nfs.md) |
| 7 | [Network Block Devices and iSCSI](07-network-block-devices.md) | Sometimes a client needs a **disk**, not a shared folder: a database wants its own filesystem, a VM needs a virtual disk, a server boots from SAN storage. | [Exercises](../../exercises/05-storage/07-network-block-devices.md) |
| 8 | [Filesystem Automounters](08-automounters.md) | Mounting every network share at boot has drawbacks: boot slows down or fails when a server is unreachable, idle mounts consume resources, and hundreds of user home directories can't all be listed in `/etc/fstab`. | [Exercises](../../exercises/05-storage/08-automounters.md) |
| 9 | [Monitoring Storage Performance](09-storage-performance.md) | Storage is the slowest component in most servers, so it is often the real cause of "the system is slow". | [Exercises](../../exercises/05-storage/09-storage-performance.md) |

**Suggested routine per lesson:** read the theory (30–45 min) → try the commands on your lab VMs → do the exercises without opening the solutions → answer *Check yourself*.
