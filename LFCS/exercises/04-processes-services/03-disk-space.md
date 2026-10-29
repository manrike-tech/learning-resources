# Exercises – Troubleshooting Disk Space

<!-- nav:start -->
> **Module 04 · Processes, Services, Scheduling & Performance** — lesson 3 of 6  
> Read first: [Theory – Troubleshooting Disk Space](../../theory/04-processes-services/03-disk-space.md) · [Course lesson (commands)](../../course/04-Processes-Services-Performance.md#lesson-3---troubleshooting-disk-space-gap) · [Module exercises](README.md)
<!-- nav:end -->

> To avoid filling your root filesystem, most exercises use a small loop-mounted test filesystem.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Setup

```bash
dd if=/dev/zero of=/root/small.img bs=1M count=200 status=none
mkfs.ext4 -q /root/small.img
mkdir -p /mnt/small && mount -o loop /root/small.img /mnt/small
```

---

## Exercise 1 – Where is the space? ★

1. Show usage and inode usage of all filesystems.
2. Find the 5 biggest directories under `/var` and `/usr` (one filesystem only).
3. Find files larger than 50 MiB anywhere on `/`.

<details>
<summary>Solution</summary>

```bash
df -h; df -i
du -xh --max-depth=1 /var 2>/dev/null | sort -h | tail -5
du -xh --max-depth=1 /usr 2>/dev/null | sort -h | tail -5
find / -xdev -type f -size +50M -exec ls -lh {} + 2>/dev/null
```

</details>

---

## Exercise 2 – Fill and find ★

Fill `/mnt/small` with a 150 MiB file in a subdirectory, then find it starting only from `df` output.

<details>
<summary>Solution</summary>

```bash
mkdir -p /mnt/small/app/logs && fallocate -l 150M /mnt/small/app/logs/huge.log
df -h /mnt/small
du -xh --max-depth=2 /mnt/small | sort -h | tail -3
rm /mnt/small/app/logs/huge.log
```

</details>

---

## Exercise 3 – Deleted but open ★★

1. Create a 150 MiB file in `/mnt/small`, keep it open with `tail -f file > /dev/null &`, then `rm` it.
2. Show that `df` still reports the space used while `du` does not.
3. Find the file with `lsof`, then free the space **without** killing `tail`.

**Verify:** after the fix, `df -h /mnt/small` shows the space free again and `tail` is still running.

<details>
<summary>Solution</summary>

```bash
fallocate -l 150M /mnt/small/data.log
tail -f /mnt/small/data.log > /dev/null & T=$!
rm /mnt/small/data.log
df -h /mnt/small; du -sh /mnt/small
lsof +L1 | grep data.log            # note PID and FD (e.g. 3r)
truncate -s 0 /proc/$T/fd/3
df -h /mnt/small; ps -p $T
kill $T
```

</details>

---

## Exercise 4 – Inode exhaustion ★★

1. Create a new test filesystem with only 1000 inodes (`mkfs.ext4 -N 1000`), mount it on `/mnt/inodes`.
2. Create empty files until you get "No space left on device".
3. Show the cause with `df`, then find the directory with the most files using `du --inodes`.

<details>
<summary>Solution</summary>

```bash
dd if=/dev/zero of=/root/inodes.img bs=1M count=50 status=none
mkfs.ext4 -q -N 1000 /root/inodes.img; mkdir -p /mnt/inodes; mount -o loop /root/inodes.img /mnt/inodes
mkdir /mnt/inodes/cache; for i in $(seq 2000); do touch /mnt/inodes/cache/f$i || break; done
df -h /mnt/inodes; df -i /mnt/inodes
du --inodes -x --max-depth=1 /mnt/inodes | sort -n
find /mnt/inodes/cache -type f -delete
```

</details>

---

## Exercise 5 – Hidden under a mount point ★★★

1. Unmount `/mnt/small`, write a 100 MiB file into the (now plain) directory `/mnt/small` on the root filesystem, then mount the image again.
2. Show the file is invisible and that `du` can't see it, then find it with a bind mount of `/`.
3. Remove it properly.

<details>
<summary>Solution</summary>

```bash
umount /mnt/small
fallocate -l 100M /mnt/small/hidden.bin
mount -o loop /root/small.img /mnt/small
ls /mnt/small                         # not there
mkdir -p /mnt/rootfs && mount --bind / /mnt/rootfs
ls -lh /mnt/rootfs/mnt/small/hidden.bin
rm /mnt/rootfs/mnt/small/hidden.bin && umount /mnt/rootfs
```

</details>

---

## Exercise 6 – Journal and logs ★

1. Show how much space the journal uses. Reduce it to 100 MiB now.
2. Limit it to 200 MiB permanently.
3. Create a logrotate config that keeps 4 compressed weekly rotations of `/var/log/myapp.log`, and dry-run it.

<details>
<summary>Solution</summary>

```bash
journalctl --disk-usage
journalctl --vacuum-size=100M
mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nSystemMaxUse=200M\n' > /etc/systemd/journald.conf.d/size.conf
systemctl restart systemd-journald
touch /var/log/myapp.log
cat > /etc/logrotate.d/myapp << 'EOF'
/var/log/myapp.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
}
EOF
logrotate -d /etc/logrotate.d/myapp
```

</details>

---

## Challenge – Exam-style ★★★

> `/var` is reported full. Find the cause and free the space without rebooting.

Setup (run on a lab VM, not the exam!): 

```bash
mkdir -p /var/lib/demo && fallocate -l 300M /var/lib/demo/cache.bin
nohup tail -f /var/lib/demo/cache.bin > /dev/null 2>&1 &
rm /var/lib/demo/cache.bin
fallocate -l 200M /var/log/demo-debug.log
```

<details>
<summary>Solution</summary>

```bash
df -h /var
du -xh --max-depth=2 /var | sort -h | tail       # finds demo-debug.log
truncate -s 0 /var/log/demo-debug.log             # or rm if not in use
lsof +L1 | grep cache.bin                         # deleted-but-open
kill <tail PID>                                   # or truncate /proc/PID/fd/N
df -h /var
```

</details>

---

## Cleanup

```bash
umount /mnt/small /mnt/inodes 2>/dev/null; rm -f /root/small.img /root/inodes.img
rm -f /etc/logrotate.d/myapp /var/log/myapp.log /var/log/demo-debug.log
```

---

<!-- footer:start -->
[← Exercises: Monitoring System Performance](02-monitoring.md) · [Module exercises](README.md) · [Exercises: Managing Services with systemd →](04-systemd.md)
<!-- footer:end -->
