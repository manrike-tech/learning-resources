# Exercises – Backup Tools

<!-- nav:start -->
> **Module 09 · Backup, Recovery & System Rescue** — lesson 2 of 4  
> Read first: [Theory – Backup Tools: tar, rsync, dd and Friends](../../theory/09-backup-rescue/02-backup-tools.md) · [Course lesson (commands)](../../course/09-Backup-Recovery-Rescue.md#lesson-2---backup-tools) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Setup

```bash
mkdir -p /backup /restore
useradd -m -s /bin/bash bkuser 2>/dev/null
for i in 1 2 3; do echo "file $i" > /home/bkuser/f$i.txt; done
setfacl -m u:root:r /home/bkuser/f1.txt
```

---

## Exercise 1 – Full tar backup with metadata ★

Back up `/etc` and `/home` to `/backup/full-$(date +%F).tar.gz`, preserving permissions, ACLs, xattrs and SELinux labels, with relative paths. List the archive and count entries.

<details>
<summary>Solution</summary>

```bash
tar --acls --xattrs --selinux -czpf /backup/full-$(date +%F).tar.gz -C / etc home
tar -tzvf /backup/full-*.tar.gz | head; tar -tzf /backup/full-*.tar.gz | wc -l
```

</details>

---

## Exercise 2 – Incremental chain and single-file restore ★★

1. Full backup of `/home` with `--listed-incremental=/backup/home.snar`.
2. Change `f2.txt`, add `f4.txt`, then make incremental 1.
3. Delete `f1.txt`, then make incremental 2.
4. Restore only `f1.txt` from the right archive into `/restore`.
5. Restore the full chain into `/restore/chain` and check that `f1.txt` is absent (deletion applied) and `f4.txt` present.

<details>
<summary>Solution</summary>

```bash
tar --listed-incremental=/backup/home.snar -czpf /backup/home-0.tar.gz -C / home
echo changed >> /home/bkuser/f2.txt; echo new > /home/bkuser/f4.txt
tar --listed-incremental=/backup/home.snar -czpf /backup/home-1.tar.gz -C / home
rm /home/bkuser/f1.txt
tar --listed-incremental=/backup/home.snar -czpf /backup/home-2.tar.gz -C / home
tar -xzpf /backup/home-0.tar.gz -C /restore home/bkuser/f1.txt; cat /restore/home/bkuser/f1.txt
mkdir -p /restore/chain
for a in /backup/home-{0,1,2}.tar.gz; do tar --listed-incremental=/dev/null -xzpf $a -C /restore/chain; done
ls /restore/chain/home/bkuser
```

</details>

---

## Exercise 3 – rsync mirror to the other VM ★★

> Mirror `/etc` to the other VM with rsync over SSH, preserving ACLs and xattrs.

Then change a file and rerun with `-i` to see exactly what is transferred. Use a dry run with `--delete` before deleting anything.

<details>
<summary>Solution</summary>

```bash
rsync -aAXHv -e ssh /etc/ root@rhel1:/backup/deb1-etc/
echo "# test" >> /etc/motd
rsync -aAXHi -e ssh /etc/ root@rhel1:/backup/deb1-etc/
rsync -aAXHn --delete -e ssh /etc/ root@rhel1:/backup/deb1-etc/
```

</details>

---

## Exercise 4 – The trailing slash ★

Run `rsync -a /home/bkuser /restore/a/` and `rsync -a /home/bkuser/ /restore/b/`. Compare the results with `ls -R`.

<details>
<summary>Solution</summary>

`/restore/a/bkuser/f*.txt` vs `/restore/b/f*.txt` – a trailing slash on the source copies the contents.

</details>

---

## Exercise 5 – Hard-link snapshots ★★★

Create three daily snapshots of `/home` under `/backup/snap/<date-or-counter>` using `--link-dest`, changing one file between runs. Show with `du -sh` and `ls -li` that unchanged files share inodes.

<details>
<summary>Solution</summary>

```bash
mkdir -p /backup/snap
rsync -aAXH /home/ /backup/snap/1/
echo x >> /home/bkuser/f2.txt
rsync -aAXH --link-dest=/backup/snap/1 /home/ /backup/snap/2/
echo y >> /home/bkuser/f3.txt
rsync -aAXH --link-dest=/backup/snap/2 /home/ /backup/snap/3/
du -sh /backup/snap/*
ls -li /backup/snap/*/bkuser/f4.txt     # same inode in all three
```

</details>

---

## Exercise 6 – dd image ★★

1. Create a 200 MiB loop disk with an ext4 filesystem and some files.
2. Image it with `dd` (unmounted), then destroy the filesystem (`wipefs -a`).
3. Restore the image and verify the files.
4. Back up the first 512 bytes of the loop disk (MBR) – what do they contain?

<details>
<summary>Solution</summary>

```bash
truncate -s 200M /root/disk.img; L=$(losetup -f --show /root/disk.img)
mkfs.ext4 -q $L; mount $L /mnt; echo keep > /mnt/a; umount /mnt
dd if=$L of=/backup/disk.dd bs=4M status=progress
wipefs -a $L; mount $L /mnt || echo "gone"
dd if=/backup/disk.dd of=$L bs=4M status=progress conv=fsync
mount $L /mnt && cat /mnt/a; umount /mnt
dd if=/dev/vda of=/backup/mbr.bin bs=512 count=1; file /backup/mbr.bin
losetup -d $L
```

</details>

---

## Exercise 7 – cpio ★

Archive all `.conf` files under `/etc` with `find | cpio`, list the archive, extract one file into `/restore/cpio`.

<details>
<summary>Solution</summary>

```bash
find /etc -name '*.conf' | cpio -ov > /backup/conf.cpio
cpio -itv < /backup/conf.cpio | head
mkdir -p /restore/cpio && cd /restore/cpio && cpio -idmv etc/sysctl.conf < /backup/conf.cpio
```

(Paths in the archive are absolute here – `cpio` strips the leading `/` only with `--no-absolute-filenames`; use it when extracting to avoid writing into the real `/etc`.)

</details>

---

## Exercise 8 – Consistent backup with an LVM snapshot ★★★

Back up an LV (e.g. `/dev/vgapp/lvdata` mounted at `/data2`) using a snapshot while a process keeps writing to it.

<details>
<summary>Solution</summary>

```bash
( while true; do date >> /data2/live.log; sleep 0.2; done ) & W=$!
lvcreate -s -n snapdata -L 200M /dev/vgapp/lvdata
mkdir -p /mnt/snap && mount -o ro /dev/vgapp/snapdata /mnt/snap      # XFS: -o ro,nouuid
tar -czpf /backup/data2-$(date +%F).tar.gz -C /mnt/snap .
umount /mnt/snap && lvremove -y /dev/vgapp/snapdata; kill $W
```

</details>

---

## Challenge – Automate and verify ★★★

Create a systemd service + timer that every day at 02:00 creates `/backup/etc-YYYY-MM-DD.tar.gz` (with ACLs/xattrs/SELinux), writes a SHA-256 checksum next to it, and deletes archives older than 14 days. Run it once manually and verify the checksum.

<details>
<summary>Solution</summary>

```bash
cat > /usr/local/bin/backup-etc.sh << 'EOF'
#!/bin/bash
set -euo pipefail
f=/backup/etc-$(date +%F).tar.gz
tar --acls --xattrs --selinux -czpf "$f" -C / etc
sha256sum "$f" > "$f.sha256"
find /backup -name 'etc-*.tar.gz*' -mtime +14 -delete
EOF
chmod 755 /usr/local/bin/backup-etc.sh
printf '[Unit]\nDescription=Backup /etc\n[Service]\nType=oneshot\nExecStart=/usr/local/bin/backup-etc.sh\n' > /etc/systemd/system/backup-etc.service
printf '[Unit]\nDescription=Daily /etc backup\n[Timer]\nOnCalendar=*-*-* 02:00\nPersistent=true\n[Install]\nWantedBy=timers.target\n' > /etc/systemd/system/backup-etc.timer
systemctl daemon-reload && systemctl enable --now backup-etc.timer
systemctl start backup-etc.service; journalctl -u backup-etc.service -n 3
cd /backup && sha256sum -c etc-$(date +%F).tar.gz.sha256
```

(`--selinux` is accepted only by tar builds with SELinux support; drop it on Ubuntu if tar complains.)

</details>

---

<!-- footer:start -->
[← Exercises: Backup Strategy](01-backup-strategy.md) · [Module exercises](README.md) · [Exercises: The Boot Process – and Where It Breaks →](03-boot-process.md)
<!-- footer:end -->
