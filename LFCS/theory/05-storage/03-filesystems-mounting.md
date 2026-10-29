# Creating, Mounting and Repairing Filesystems

<!-- nav:start -->
> **Module 05 · Storage** — lesson 3 of 9  
> [Module index](README.md) · [Course lesson (commands)](../../course/05-Storage.md#lesson-3---creating-checking--mounting-ch-2122) · [Exercises for this lesson](../../exercises/05-storage/03-filesystems-mounting.md)
<!-- nav:end -->

A partition or logical volume is just raw space until you put a **filesystem** on it and **mount** it into the directory tree. In this lesson you will learn to create ext4 and XFS filesystems, mount them temporarily and permanently with `/etc/fstab`, choose mount options, check and repair filesystems, and tune them.

## What you will learn

- Creating filesystems with `mkfs.ext4`, `mkfs.xfs` and others
- Identifying filesystems: UUIDs and labels
- Mounting and unmounting; handling "target is busy"
- `/etc/fstab`: fields, options and safe editing
- How systemd turns fstab into mount units
- Checking and repairing with `fsck`, `e2fsck`, `xfs_repair`
- Tuning ext4 with `tune2fs` and inspecting XFS with `xfs_info`
- File attributes with `chattr`

---

## Key terms

| Term | Definition |
|------|-----------|
| **mkfs** | *Make filesystem*: writes a new, empty filesystem on a device (destroying previous content). |
| **UUID** | A unique ID generated when the filesystem is created; stable across reboots and device renames. |
| **Label** | A human-readable name you assign to a filesystem (`DATA`). |
| **Mount point** | An existing directory where the filesystem is attached. |
| **`/etc/fstab`** | *Filesystem table*: filesystems mounted automatically at boot. |
| **Mount options** | Flags controlling a mount: `ro`, `noexec`, `noatime`, `_netdev`, `nofail`… |
| **fsck** | *Filesystem check*: verifies and repairs a filesystem's structure. Must run on an **unmounted** filesystem. |
| **Dirty** | A filesystem that wasn't unmounted cleanly; checked or journal-replayed at the next mount. |

---

## Creating filesystems

```console
$ sudo mkfs.ext4 -L DATA /dev/vdb1
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 262144 4k blocks and 65536 inodes
Filesystem UUID: 3f1c2a7e-9b1d-4d5e-8c2f-1a2b3c4d5e6f
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376
Writing superblocks and filesystem accounting information: done

$ sudo mkfs.xfs -L LOGS /dev/vdb2
meta-data=/dev/vdb2              isize=512    agcount=4, agsize=32768 blks
data     =                       bsize=4096   blocks=131072, imaxpct=25
```

| Command | Filesystem | Useful options |
|---------|-----------|----------------|
| `mkfs.ext4` (= `mkfs -t ext4`) | ext4 | `-L label`, `-m 1` (reserved %), `-N inodes`, `-b 4096` |
| `mkfs.xfs` | XFS | `-L label`, `-f` (force overwrite an existing FS) |
| `mkfs.btrfs` | Btrfs | `-L label` |
| `mkfs.vfat -F 32` | FAT32 | `-n LABEL` |
| `mkswap` | swap | `-L label` (see [Swap](05-swap.md)) |

> `mkfs` destroys everything on the device. Run `lsblk -f` first and make sure the device is not mounted and is the one you mean.

### UUIDs and labels

```console
$ sudo blkid /dev/vdb1
/dev/vdb1: LABEL="DATA" UUID="3f1c2a7e-…" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="a1b2…"
$ lsblk -o NAME,FSTYPE,LABEL,UUID,MOUNTPOINTS /dev/vdb
$ ls -l /dev/disk/by-uuid/ /dev/disk/by-label/
```

Change a label: `e2label /dev/vdb1 NEWNAME` (ext4), `xfs_admin -L NEWNAME /dev/vdb2` (XFS, unmounted). A new `mkfs` creates a **new UUID** – update `/etc/fstab` afterwards.

---

## Mounting and unmounting

```console
$ sudo mkdir -p /data
$ sudo mount /dev/vdb1 /data                 # type auto-detected
$ sudo mount -t xfs -o noatime /dev/vdb2 /logs
$ sudo mount UUID=3f1c2a7e-… /data           # by UUID
$ sudo mount LABEL=DATA /data                # by label
$ findmnt /data
TARGET SOURCE    FSTYPE OPTIONS
/data  /dev/vdb1 ext4   rw,relatime
$ sudo mount -o remount,ro /data             # change options of a mounted FS
$ sudo umount /data                          # by mount point (or device)
```

### "target is busy"

```console
$ sudo umount /data
umount: /data: target is busy.
$ sudo lsof +f -- /data          # which processes use it
$ sudo fuser -vm /data
                     USER   PID ACCESS COMMAND
/data:               alice 2210 ..c..  bash     ← a shell whose current directory is /data
```

Stop those processes (or `cd` away), then unmount. `umount -l` (lazy) detaches now and cleans up later – use with care.

### Other useful mounts

```console
$ sudo mount -o loop disk.img /mnt/img          # a file as a block device
$ sudo mount -o loop,ro ubuntu.iso /mnt/iso     # an ISO image
$ sudo mount --bind /srv/data /var/www/data     # make a directory visible elsewhere
```

---

## Persistent mounts: /etc/fstab

Each line describes one filesystem to mount at boot:

```text
# <device>                                 <mount point> <type> <options>                 <dump> <pass>
UUID=3f1c2a7e-9b1d-4d5e-8c2f-1a2b3c4d5e6f  /data         ext4   defaults                  0      2
LABEL=LOGS                                 /logs         xfs    defaults,noatime          0      0
/dev/vg01/lv_web                           /var/www      xfs    defaults                  0      0
UUID=9a2b…                                 none          swap   defaults                  0      0
server1:/exports/share                     /mnt/nfs      nfs    defaults,_netdev,nofail   0      0
tmpfs                                      /scratch      tmpfs  size=512M,mode=1777       0      0
```

| Field | Meaning |
|-------|---------|
| 1 device | `UUID=…` (recommended), `LABEL=…`, `/dev/mapper/…` / `/dev/vg/lv`, `server:/path` |
| 2 mount point | an existing directory (`none` for swap) |
| 3 type | `ext4`, `xfs`, `swap`, `nfs`, `tmpfs`, `auto` … |
| 4 options | comma-separated, `defaults` = `rw,suid,dev,exec,auto,nouser,async` |
| 5 dump | `0` (legacy backup flag – always 0 today) |
| 6 pass | fsck order at boot: `0` = never, `1` = root FS, `2` = others. Use `0` for XFS, swap, network |

### Common mount options

| Option | Effect |
|--------|--------|
| `ro` / `rw` | read-only / read-write |
| `noatime`, `relatime` | don't update / only lazily update access times (performance) |
| `noexec` | programs on this FS can't be executed |
| `nosuid` | SUID/SGID bits ignored |
| `nodev` | device files ignored |
| `nofail` | boot continues if the device is missing |
| `noauto` | don't mount at boot (only with `mount /point`) |
| `user`, `users` | allow normal users to mount it |
| `_netdev` | needs the network – wait for it |
| `x-systemd.automount` | mount on first access (see [Automounters](08-automounters.md)) |
| `x-systemd.device-timeout=10s` | how long to wait for the device |
| `uid=`, `gid=`, `umask=` | ownership emulation for vfat/ntfs |

Security hardening example: `/tmp` and `/var/tmp` with `nodev,nosuid,noexec`.

### Editing fstab safely

A wrong line can drop the system into **emergency mode** at the next boot. Always test:

```console
$ sudo cp /etc/fstab /etc/fstab.bak
$ sudo vi /etc/fstab
$ sudo findmnt --verify            # syntax and devices check
Success, no errors or warnings detected
$ sudo systemctl daemon-reload     # regenerate mount units from fstab
$ sudo mount -a                    # mount everything not yet mounted – errors show now, not at boot
$ findmnt /data
```

Repairing a broken fstab from emergency mode is covered in [System Rescue](../09-backup-rescue/04-system-rescue.md).

### fstab and systemd

At boot, `systemd-fstab-generator` converts every fstab line into a **mount unit** named after the path (`/var/www` → `var-www.mount`):

```console
$ systemctl status data.mount
● data.mount - /data
     Loaded: loaded (/etc/fstab; generated)
     Active: active (mounted) since …
$ systemctl list-units --type=mount
```

You can also write mount units by hand in `/etc/systemd/system/` – the file name must match the path (`systemd-escape -p --suffix=mount /var/www` → `var-www.mount`).

---

## Checking and repairing

Run checks only on **unmounted** filesystems (or read-only ones). Checking a mounted read-write filesystem can corrupt it.

### ext4

```console
$ sudo umount /data
$ sudo e2fsck -f /dev/vdb1             # -f force even if clean
e2fsck 1.47.0 (5-Feb-2023)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
DATA: 11/65536 files (0.0% non-contiguous), 12955/262144 blocks
$ sudo e2fsck -fy /dev/vdb1            # -y answer yes to every repair
$ sudo e2fsck -b 32768 /dev/vdb1       # use a backup superblock if the primary is damaged
```

`fsck /dev/vdb1` detects the type and calls `fsck.ext4` (= `e2fsck`). Recovered files without names end up in `lost+found/` at the filesystem root.

### XFS

```console
$ sudo umount /logs
$ sudo xfs_repair -n /dev/vdb2         # -n check only, no modifications
$ sudo xfs_repair /dev/vdb2
```

`fsck.xfs` does nothing (XFS replays its log at mount time). If `xfs_repair` complains about a dirty log, mount and unmount the filesystem once to replay it; `xfs_repair -L` (zero the log) is a last resort and may lose recent changes.

### Checking at boot

- Field 6 of fstab controls automatic checks for ext filesystems.
- Force a check of all filesystems at next boot with the kernel parameter `fsck.mode=force` (`fsck.repair=yes` to auto-fix).
- ext4 can also schedule checks: `tune2fs -c 30` (every 30 mounts), `tune2fs -i 1m` (monthly).

---

## Tuning and inspecting

### ext4: tune2fs and dumpe2fs

```console
$ sudo tune2fs -l /dev/vdb1 | grep -E 'volume name|UUID|Block count|Reserved block count|Mount count|Last checked'
Filesystem volume name:   DATA
Filesystem UUID:          3f1c2a7e-…
Block count:              262144
Reserved block count:     13107
Mount count:              3
Last checked:             Tue Oct 21 10:02:11 2026
$ sudo tune2fs -L ARCHIVE /dev/vdb1      # set label
$ sudo tune2fs -m 1 /dev/vdb1            # reserve 1% for root instead of 5%
$ sudo tune2fs -U random /dev/vdb1       # new UUID (unmounted)
$ sudo tune2fs -O ^has_journal …         # change features (advanced)
$ sudo dumpe2fs /dev/vdb1 | less         # full details incl. backup superblocks
```

### XFS

```console
$ xfs_info /logs                         # geometry of a mounted XFS
$ sudo xfs_admin -l -u /dev/vdb2         # label and UUID (unmounted)
$ sudo xfs_admin -L NEWLABEL /dev/vdb2
```

---

## File attributes

Some filesystems support extra per-file attributes:

```console
$ sudo chattr +i /etc/important.conf     # immutable: no change, delete or rename – even by root
$ sudo chattr +a /var/log/audit.trail    # append-only
$ lsattr /etc/important.conf
----i---------e------- /etc/important.conf
$ sudo chattr -i /etc/important.conf
```

---

## Example: a new disk from start to finish

```console
$ lsblk /dev/vdb                                      # 1. confirm the disk
$ sudo parted /dev/vdb --script mklabel gpt mkpart data 1MiB 100%   # 2. partition
$ sudo mkfs.ext4 -L DATA /dev/vdb1                    # 3. filesystem
$ sudo mkdir -p /data                                 # 4. mount point
$ echo "UUID=$(sudo blkid -s UUID -o value /dev/vdb1) /data ext4 defaults 0 2" | sudo tee -a /etc/fstab
$ sudo systemctl daemon-reload && sudo mount -a       # 5. test fstab
$ df -h /data                                         # 6. verify
```

---

## Try it yourself

1. Create an ext4 filesystem with label `DATA` on a partition, mount it at `/data` persistently by UUID, reboot and verify.
2. Create an XFS filesystem on another partition and mount it at `/logs` with `noatime` persistently.
3. From a shell `cd` into `/data` and try to unmount it from another shell; find the blocker.
4. Add a deliberately wrong UUID line with `nofail` and see that `mount -a` warns but boot would continue.
5. Unmount `/data` and run a forced `e2fsck`; run `xfs_repair -n` on `/logs`.

---

## Common mistakes

- Using `/dev/sdb1` in fstab (names can change) instead of `UUID=`.
- Not running `mount -a` after editing fstab.
- Running `fsck` on a mounted filesystem.
- `pass` = 1 or 2 for XFS (harmless, but `fsck.xfs` does nothing) or for network filesystems.
- Re-running `mkfs` and forgetting the UUID changed.
- Mount point directory doesn't exist.

---

## Summary

| Task | Command |
|------|---------|
| Create | `mkfs.ext4 -L L /dev/X`, `mkfs.xfs -L L /dev/X` |
| Identify | `blkid`, `lsblk -f` |
| Mount / unmount | `mount DEV DIR`, `umount DIR` |
| Who blocks umount | `fuser -vm DIR`, `lsof +f -- DIR` |
| Persistent | `/etc/fstab` with `UUID=` → `findmnt --verify` → `daemon-reload` → `mount -a` |
| Check ext4 / XFS | `e2fsck -f` / `xfs_repair -n` (unmounted!) |
| Tune ext4 | `tune2fs -l`, `-L`, `-m`, `-c`, `-i` |
| Info XFS | `xfs_info`, `xfs_admin` |

## Check yourself

1. Why use `UUID=` rather than `/dev/vdb1` in fstab?
2. What does `nofail` do?
3. Which fstab field controls fsck at boot, and what value should root have?
4. Why must you not run `e2fsck` on a mounted filesystem?

<details>
<summary>Answers</summary>

1. Device names can change between boots; UUIDs identify the filesystem itself.
2. If the device is missing or fails to mount, boot continues instead of dropping into emergency mode.
3. Field 6 (pass); `1` for the root filesystem, `2` for others, `0` to skip.
4. The kernel is changing the filesystem at the same time; repairs could corrupt it.

</details>

## Further reading

- `man 8 mount`, `man 5 fstab`, `man 8 mkfs.ext4` (`mke2fs`), `man 8 mkfs.xfs`, `man 8 e2fsck`, `man 8 xfs_repair`, `man 8 tune2fs`, `man 5 systemd.mount`
- [ArchWiki: fstab](https://wiki.archlinux.org/title/Fstab) · [ext4](https://wiki.archlinux.org/title/Ext4) · [XFS](https://wiki.archlinux.org/title/XFS)
- [Red Hat: Managing file systems (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_file_systems/index)

---

<!-- footer:start -->
[← Disk Partitioning](02-partitioning.md) · [Module index](README.md) · [Logical Volume Management (LVM) →](04-lvm.md)  
**Practice now:** [Exercises – Creating, Mounting and Repairing Filesystems](../../exercises/05-storage/03-filesystems-mounting.md)
<!-- footer:end -->
