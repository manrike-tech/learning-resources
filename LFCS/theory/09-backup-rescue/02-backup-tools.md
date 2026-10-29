# Backup Tools: tar, rsync, dd and Friends

<!-- nav:start -->
> **Module 09 · Backup, Recovery & System Rescue** — lesson 2 of 4  
> [Module index](README.md) · [Course lesson (commands)](../../course/09-Backup-Recovery-Rescue.md#lesson-2---backup-tools) · [Exercises for this lesson](../../exercises/09-backup-rescue/02-backup-tools.md)
<!-- nav:end -->

Linux ships with powerful backup tools: **tar** for archives, **rsync** for efficient copies and mirrors, **dd** for raw disk images, **cpio** for file lists, plus filesystem-specific dump tools and LVM snapshots for consistency. In this lesson you will learn what each tool preserves, how to create full and incremental backups, how to restore single files or everything, and how to automate and verify backups.

## What you will learn

- Choosing the right tool for the job
- tar: full and incremental (`--listed-incremental`) backups, compression, metadata, restoring
- rsync: syntax, trailing slashes, preserving metadata, `--delete`, remote copies, hard-link snapshots
- dd: disk and partition images, MBR backup, cloning, safety
- cpio, dump/restore, xfsdump/xfsrestore
- LVM snapshots for consistent backups
- Modern deduplicating tools (borg, restic) in brief
- Automating with a systemd timer; verifying backups

---

## Key terms

| Term | Definition |
|------|-----------|
| **Archive** | A single file containing many files plus metadata. |
| **Metadata** | Owner, group, permissions, timestamps, ACLs, extended attributes (incl. SELinux labels), hard links, sparse regions. |
| **Snapshot file (tar)** | The `.snar` file used by GNU tar to track what was backed up, for incrementals. |
| **Delta transfer** | rsync's algorithm sending only the changed parts of files. |
| **Mirror** | A copy kept identical to the source (incl. deletions). |
| **Block-level copy** | Copying a device byte for byte, regardless of filesystem (`dd`). |
| **Deduplication** | Storing identical data chunks only once across backups. |

---

## Choosing a tool

| Tool | Level | Best for | Preserves metadata |
|------|-------|---------|--------------------|
| `tar` | file | archives, full/incremental backups, transport | with `-p`, `--acls --xattrs --selinux` |
| `rsync` | file | copies, mirrors, remote sync, snapshot trees | with `-aAXH` |
| `cpio` | file | archives from `find` lists; initramfs/RPM internals | yes (`-p`/`-m` options) |
| `dd` | block | whole-disk/partition images, MBR, cloning | everything (it's raw) |
| `dump`/`restore` | filesystem (ext) | levels 0–9 incremental per filesystem | yes |
| `xfsdump`/`xfsrestore` | filesystem (XFS) | XFS incremental dumps | yes |
| borg / restic | file, deduplicated | production backups with encryption and retention | yes |

---

## tar

### Full backup

```console
$ sudo tar --acls --xattrs --selinux -czpf /backup/etc-$(date +%F).tar.gz -C / etc
$ sudo tar -cJpf /backup/home-full.tar.xz --exclude='*/.cache' -C / home
```

| Option | Meaning |
|--------|---------|
| `-c` / `-x` / `-t` | create / extract / list |
| `-f FILE` | archive file (`-` = stdout/stdin) |
| `-z` `-j` `-J` `--zstd` | gzip / bzip2 / xz / zstd |
| `-p` | preserve permissions (default for root on extract) |
| `--acls --xattrs --selinux` | ACLs, extended attributes, SELinux contexts |
| `-C DIR` | change to DIR first (store relative paths / extract there) |
| `--exclude=PATTERN`, `-X FILE` | skip files |
| `--one-file-system` | don't cross into other mounted filesystems |
| `-v` | list files processed |

Using `-C / etc` stores paths as `etc/…` and avoids the "Removing leading `/`" message.

### Incremental backups with GNU tar

tar records file state in a **snapshot file**. The first run with a new snapshot file is a full (level 0) backup; later runs with the same file save only changes (including information about deleted files).

```console
# Sunday: full
$ sudo tar --listed-incremental=/backup/home.snar -czpf /backup/home-0-full.tar.gz -C / home
# Monday: incremental (same .snar is updated)
$ sudo tar --listed-incremental=/backup/home.snar -czpf /backup/home-1-mon.tar.gz -C / home
# Tuesday
$ sudo tar --listed-incremental=/backup/home.snar -czpf /backup/home-2-tue.tar.gz -C / home
```

For **differential** backups, copy the level-0 snapshot file before each run (`cp home.snar.0 home.snar`), so every run compares with the full.

Restore in order, using `/dev/null` as the snapshot so tar applies deletions:

```console
$ sudo tar --listed-incremental=/dev/null -xzpf /backup/home-0-full.tar.gz -C /restore
$ sudo tar --listed-incremental=/dev/null -xzpf /backup/home-1-mon.tar.gz -C /restore
$ sudo tar --listed-incremental=/dev/null -xzpf /backup/home-2-tue.tar.gz -C /restore
```

### Restoring single files

```console
$ tar -tzvf /backup/etc-2026-10-21.tar.gz | grep sshd_config
-rw------- root/root      3282 2026-10-01 10:12 etc/ssh/sshd_config
$ sudo tar -xzpf /backup/etc-2026-10-21.tar.gz -C /tmp/restore etc/ssh/sshd_config
$ sudo tar -xzpf /backup/etc-2026-10-21.tar.gz -C /tmp/restore --wildcards 'etc/ssh/*'
$ diff /tmp/restore/etc/ssh/sshd_config /etc/ssh/sshd_config
```

Restore to a temporary directory, compare, then copy into place – never blindly over a live system.

### Archives over the network

```console
$ sudo tar -czpf - -C / etc | ssh backup@nas "cat > /backups/web01-etc.tar.gz"
$ ssh backup@nas "cat /backups/web01-etc.tar.gz" | sudo tar -xzpf - -C /tmp/restore
```

---

## rsync

`rsync` copies only differences – fast for repeated backups and perfect for remote copies over SSH.

```console
$ rsync -av /srv/data/ /backup/data/                    # local copy
$ rsync -aAXHv --delete /home/ /backup/home/            # exact mirror incl. ACLs, xattrs, hard links, deletions
$ rsync -avz -e ssh /etc/ backup@nas:/backups/web01/etc/          # to a remote host (compressed)
$ rsync -avz -e "ssh -p 2222" backup@nas:/backups/web01/etc/ /tmp/restore/etc/   # restore from remote
$ rsync -avn --delete /home/ /backup/home/              # -n dry run: show what would happen
$ rsync -av --progress --partial bigfile.iso nas:/isos/ # resumable transfer
```

| Option | Meaning |
|--------|---------|
| `-a` | archive = `-rlptgoD` (recursive, links, perms, times, group, owner, devices) |
| `-A` / `-X` / `-H` | ACLs / extended attributes (SELinux!) / hard links |
| `-v`, `--progress`, `-i` | verbose, progress, itemized changes |
| `-z` | compress during transfer |
| `-n` | dry run |
| `--delete` | delete files in the destination that are gone from the source |
| `--exclude=PAT`, `--exclude-from=FILE` | skip |
| `-e ssh` | remote shell (default today) |
| `--numeric-ids` | keep UID/GID numbers (restoring onto another system) |
| `-c` | compare by checksum instead of size+time |

### The trailing slash

```console
$ rsync -a /srv/data  /backup/     # creates /backup/data/…     (copies the directory)
$ rsync -a /srv/data/ /backup/     # puts the CONTENTS into /backup/…
```

Rule: a trailing slash on the **source** means "the contents of". Combined with `--delete`, a wrong slash can delete a lot – use `-n` first.

### Snapshot backups with hard links

```console
$ today=$(date +%F); last=$(ls -1d /backup/snap/* 2>/dev/null | tail -1)
$ sudo rsync -aAXH --delete ${last:+--link-dest=$last} /home/ /backup/snap/$today/
$ du -sh /backup/snap/*        # each snapshot looks complete; unchanged files share inodes
```

Each day's directory is a full browsable copy, but unchanged files are **hard links** to the previous snapshot – only changes use space.

---

## dd

`dd` copies **blocks**. It knows nothing about files, so it copies everything – used space, free space, partition tables, boot code.

```console
$ sudo dd if=/dev/vdb of=/backup/vdb.img bs=4M status=progress conv=fsync      # image a whole disk
$ sudo dd if=/dev/vdb1 of=/backup/vdb1.img bs=4M status=progress               # one partition
$ sudo dd if=/dev/vda of=/backup/mbr.bin bs=512 count=1                         # MBR: boot code + partition table
$ sudo dd if=/backup/vdb.img of=/dev/vdb bs=4M status=progress conv=fsync      # restore
$ sudo dd if=/dev/vdb bs=4M status=progress | gzip -c > /backup/vdb.img.gz     # compressed image
$ gunzip -c /backup/vdb.img.gz | sudo dd of=/dev/vdb bs=4M status=progress
$ sudo dd if=/dev/sdb of=/dev/sdc bs=4M status=progress                         # clone disk to disk
```

| Operand | Meaning |
|---------|---------|
| `if=` | input file / device |
| `of=` | output file / device |
| `bs=` | block size (4M is a good default) |
| `count=` | number of blocks |
| `status=progress` | show progress |
| `conv=fsync` | flush to disk at the end |

> **`dd` is "disk destroyer" if you swap `if` and `of`.** Double-check device names with `lsblk`. The source should be **unmounted** (or mounted read-only) for a consistent image, and the target must be at least as large as the source.

Restoring only the MBR boot code (not the partition table): `dd if=/backup/mbr.bin of=/dev/vda bs=446 count=1`. For GPT disks use `sgdisk --backup` / `sfdisk -d` instead.

Images can be mounted to extract files: `losetup -fP --show vdb.img` then `mount /dev/loop0p1 /mnt`.

---

## cpio

Reads a list of file names from stdin – pairs naturally with `find`:

```console
$ find /etc -depth -print | sudo cpio -ov --format=newc > /backup/etc.cpio      # create
$ cpio -itv < /backup/etc.cpio | head                                           # list
$ mkdir /tmp/r && cd /tmp/r && cpio -idmv < /backup/etc.cpio                    # extract (-d dirs, -m keep mtime)
$ find /srv/app -newer /backup/last-run | cpio -pdm /backup/app-changed         # pass-through copy
```

The initramfs and the payload of RPM packages are cpio archives (`rpm2cpio pkg.rpm | cpio -idv` extracts an RPM).

---

## Filesystem dump tools

```console
# ext2/3/4 (package dump)
$ sudo dump -0uf /backup/home.0.dump /home       # level 0 = full, -u records in /etc/dumpdates
$ sudo dump -1uf /backup/home.1.dump /home       # level 1 = changes since last level 0
$ sudo restore -rf /backup/home.0.dump           # restore all into the current directory
$ sudo restore -if /backup/home.0.dump           # interactive: browse and pick files

# XFS (package xfsdump)
$ sudo xfsdump -l 0 -f /backup/data.xfsdump /data
$ sudo xfsrestore -f /backup/data.xfsdump /restore
$ sudo xfsrestore -i -f /backup/data.xfsdump /restore    # interactive
```

---

## Consistent backups with LVM snapshots

```console
$ sudo lvcreate -s -n snap_home -L 1G /dev/vg01/lv_home     # instant point-in-time view
$ sudo mount -o ro /dev/vg01/snap_home /mnt/snap            # XFS: add nouuid
$ sudo tar --acls --xattrs -czpf /backup/home-$(date +%F).tar.gz -C /mnt/snap .
$ sudo umount /mnt/snap && sudo lvremove -y /dev/vg01/snap_home
```

The snapshot freezes the data at one moment while the live LV keeps changing. Make it large enough for the changes during the backup (see [LVM](../05-storage/04-lvm.md)).

---

## Deduplicating backup tools (overview)

| | restic | BorgBackup |
|--|--------|-----------|
| Encryption | always | optional (recommended) |
| Deduplication | yes | yes |
| Targets | local, SFTP, S3, many clouds | local, SSH |
| Example | `restic -r sftp:nas:/repo backup /etc /home` | `borg create nas:/repo::{now} /etc /home` |
| Retention | `restic forget --keep-daily 7 --keep-weekly 4 --prune` | `borg prune --keep-daily 7 …` |
| Restore | `restic restore latest --target /restore` | `borg extract repo::archive` |

These are what you'd use in production; the LFCS focuses on the standard tools.

---

## Automating

```ini
# /etc/systemd/system/backup-etc.service
[Unit]
Description=Backup /etc
[Service]
Type=oneshot
ExecStart=/bin/sh -c 'tar --acls --xattrs --selinux -czpf /backup/etc-$(date +%%F).tar.gz -C / etc'
ExecStartPost=/usr/bin/find /backup -name "etc-*.tar.gz" -mtime +14 -delete
```

```ini
# /etc/systemd/system/backup-etc.timer
[Timer]
OnCalendar=*-*-* 02:00
Persistent=true
[Install]
WantedBy=timers.target
```

```console
$ sudo systemctl daemon-reload && sudo systemctl enable --now backup-etc.timer
$ sudo systemctl start backup-etc.service && journalctl -u backup-etc.service -n 5
```

(`%%` escapes `%` in unit files.) See [Scheduling Jobs](../04-processes-services/06-scheduling-jobs.md).

---

## Verifying

```console
$ tar -tzf /backup/etc-2026-10-21.tar.gz > /dev/null && echo "archive readable"
$ cd / && sudo tar -dzf /backup/etc-2026-10-21.tar.gz | head     # --compare: differences vs filesystem
$ sha256sum /backup/*.tar.gz > /backup/SHA256SUMS; sha256sum -c /backup/SHA256SUMS
$ rsync -aAXHn --delete -c /home/ /backup/home/ | head           # dry-run checksum compare
```

And periodically: restore into a scratch directory or VM and check that services work with the restored data.

---

## Try it yourself

1. Create a full and two incremental tar backups of `/home` (changing files in between); delete a file; restore only that file; then restore the whole set into `/restore` in order.
2. Mirror `/etc` to the other VM with rsync over SSH, preserving ACLs and xattrs; change a file and rerun with `-i` to see what's transferred.
3. Explain the difference between `rsync -a src dst` and `rsync -a src/ dst` by testing it.
4. Image a small loop-device filesystem with `dd`, corrupt the original, restore from the image.
5. Take an LVM snapshot of an LV, back it up with tar, remove the snapshot.
6. Automate the `/etc` backup with a systemd timer and verify a run in the journal.

---

## Common mistakes

- Forgetting `--acls --xattrs --selinux` (tar) or `-AX` (rsync): restored services fail on permissions/labels.
- Absolute paths in archives, extracting over a live system.
- Wrong trailing slash with `rsync --delete`.
- Swapped `if`/`of` in `dd`; imaging a mounted read-write filesystem.
- Reusing the same `.snar` file for unrelated backup chains.
- Backups that are never verified or restored.

---

## Summary

| Task | Command |
|------|---------|
| Full tar backup | `tar --acls --xattrs --selinux -czpf FILE -C / DIR` |
| Incremental tar | `tar --listed-incremental=FILE.snar -czpf …` |
| List / extract one file | `tar -tzf A`, `tar -xzpf A -C /tmp/r path/in/archive` |
| Mirror | `rsync -aAXH --delete SRC/ DST/` |
| Remote | `rsync -avz -e ssh SRC/ user@host:DST/` |
| Snapshot trees | `rsync -aAXH --link-dest=PREV SRC/ NEW/` |
| Disk image | `dd if=/dev/X of=X.img bs=4M status=progress` |
| MBR | `dd if=/dev/sda of=mbr.bin bs=512 count=1` |
| cpio | `find DIR \| cpio -ov > a.cpio`, `cpio -idmv < a.cpio` |
| ext / XFS dumps | `dump -0uf` / `restore -rf`; `xfsdump -l 0` / `xfsrestore` |
| Consistency | LVM snapshot → mount ro → back up → remove |

## Check yourself

1. Which tar options preserve ACLs and SELinux labels?
2. What does a trailing slash on an rsync source mean?
3. How does tar know what changed for an incremental backup?
4. Why should a partition be unmounted before imaging it with `dd`?
5. What does `--link-dest` do?

<details>
<summary>Answers</summary>

1. `--acls --xattrs --selinux` (plus `-p` to preserve permissions).
2. Copy the directory's contents rather than the directory itself.
3. From the snapshot file given with `--listed-incremental`, which records the state of each file at the previous run.
4. A mounted read-write filesystem changes during the copy, so the image would be inconsistent.
5. Files unchanged compared with that previous snapshot are hard-linked instead of copied, so each snapshot looks complete but only changes use space.

</details>

## Further reading

- `man 1 tar`, `man 1 rsync`, `man 1 dd`, `man 1 cpio`, `man 8 dump`, `man 8 xfsdump`
- [GNU tar manual – Incremental dumps](https://www.gnu.org/software/tar/manual/html_node/Incremental-Dumps.html)
- [rsync documentation](https://rsync.samba.org/documentation.html)
- [ArchWiki: rsync](https://wiki.archlinux.org/title/Rsync) · [dd](https://wiki.archlinux.org/title/Dd) · [Full system backup with tar](https://wiki.archlinux.org/title/Full_system_backup_with_tar)
- [restic documentation](https://restic.readthedocs.io/) · [BorgBackup documentation](https://borgbackup.readthedocs.io/)

---

<!-- footer:start -->
[← Backup Strategy](01-backup-strategy.md) · [Module index](README.md) · [The Boot Process – and Where It Breaks →](03-boot-process.md)  
**Practice now:** [Exercises – Backup Tools: tar, rsync, dd and Friends](../../exercises/09-backup-rescue/02-backup-tools.md)
<!-- footer:end -->
