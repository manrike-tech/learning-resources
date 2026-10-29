# Exercises – The Linux Filesystem Hierarchy

<!-- nav:start -->
> **Module 01 · Foundations & Essential Commands** — lesson 1 of 6  
> Read first: [Theory – The Linux Filesystem Hierarchy](../../theory/01-foundations/01-filesystem-hierarchy.md) · [Course lesson (commands)](../../course/01-Foundations.md#lesson-1---the-filesystem-hierarchy-standard) · [Module exercises](README.md)
<!-- nav:end -->

> Work on `deb1` and `rhel1`. Try each exercise **before** opening the solution. Every exercise has a **Verify** step: if your verification passes, you're done.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Map the tree ★

1. List the top-level directories and identify which ones are symbolic links.
2. Write into `/root/fhs.txt`, one per line, the directory where each of these lives: SSH server config, system logs, kernel images, user home directories, the root user's home, third-party self-contained software.

**Verify:** `cat /root/fhs.txt` shows 6 directories.

<details>
<summary>Solution</summary>

```bash
ls -l / | grep '^l'          # /bin, /sbin, /lib, /lib64 → usr/...
cat > /root/fhs.txt << 'EOF'
/etc/ssh
/var/log
/boot
/home
/root
/opt
EOF
```

</details>

---

## Exercise 2 – Real, virtual and RAM filesystems ★

Using `findmnt` and `df`, record in `/root/fs-types.txt`:

- the filesystem type and source device of `/`
- the type of `/proc`, `/sys` and `/run`
- whether `/tmp` is a tmpfs on this system

**Verify:** compare your answers with `findmnt -no SOURCE,FSTYPE / /proc /sys /run /tmp`.

<details>
<summary>Solution</summary>

```bash
findmnt -no TARGET,SOURCE,FSTYPE /          >  /root/fs-types.txt
findmnt -no TARGET,FSTYPE /proc /sys /run   >> /root/fs-types.txt
findmnt -no TARGET,FSTYPE /tmp >> /root/fs-types.txt || echo "/tmp is on the root fs" >> /root/fs-types.txt
```

`/proc` = `proc`, `/sys` = `sysfs`, `/run` = `tmpfs`. On Fedora `/tmp` is tmpfs; on Ubuntu/RHEL it's usually part of `/`.

</details>

---

## Exercise 3 – Read live kernel information ★

Without using `free`, `uptime` or `uname`, find (from `/proc` only):

1. total RAM, 2. 1-minute load average, 3. kernel version, 4. the command line the kernel booted with.

**Verify:** compare with `free -k`, `uptime`, `uname -r`.

<details>
<summary>Solution</summary>

```bash
grep MemTotal /proc/meminfo
cut -d' ' -f1 /proc/loadavg
cat /proc/sys/kernel/osrelease     # or /proc/version
cat /proc/cmdline
```

</details>

---

## Exercise 4 – File types ★

In `/tmp/types`, create one regular file, one directory, one symlink and one named pipe. Then list them so the type character is visible, and identify a block and a character device in `/dev`.

**Verify:** `ls -l /tmp/types` shows `-`, `d`, `l`, `p` as first characters.

<details>
<summary>Solution</summary>

```bash
mkdir -p /tmp/types && cd /tmp/types
touch file; mkdir dir; ln -s file link; mkfifo pipe
ls -l
ls -l /dev/sda /dev/vda /dev/null 2>/dev/null     # b = block, c = character
```

</details>

---

## Exercise 5 – Hard links vs symbolic links ★★

1. In `/tmp/links`, create `data.txt` containing `version 1`.
2. Create a hard link `hard.txt` and a symlink `soft.txt` to it.
3. Record the inode numbers and link counts.
4. Append `version 2` through `hard.txt`. Check `data.txt`.
5. Delete `data.txt`. Which link still works? Why?
6. Try to create a hard link to `data.txt` in `/boot` (or another filesystem) – what error do you get?

**Verify:** after step 5, `cat hard.txt` prints both lines and `cat soft.txt` fails.

<details>
<summary>Solution</summary>

```bash
mkdir -p /tmp/links && cd /tmp/links
echo "version 1" > data.txt
ln data.txt hard.txt
ln -s data.txt soft.txt
ls -li                       # data.txt and hard.txt share an inode, link count 2
echo "version 2" >> hard.txt
cat data.txt                 # shows both lines – same inode, same data
rm data.txt
cat hard.txt                 # works: link count dropped to 1, data kept
cat soft.txt                 # fails: points to a name that no longer exists
sudo ln hard.txt /boot/x     # "Invalid cross-device link" if /boot is a different FS
```

</details>

---

## Exercise 6 – Inode exhaustion ★★

Show that a filesystem can be "full" while having free space.

1. Create a 10 MiB file, format it as ext4 with very few inodes and loop-mount it:
   `dd if=/dev/zero of=/root/tiny.img bs=1M count=10 && mkfs.ext4 -N 64 /root/tiny.img`
2. Mount it on `/mnt/tiny` and create empty files until you get an error.
3. Compare `df -h /mnt/tiny` and `df -i /mnt/tiny`.
4. Clean up.

**Verify:** `df -i` shows `IUse%` at 100% while `df -h` shows free space.

<details>
<summary>Solution</summary>

```bash
dd if=/dev/zero of=/root/tiny.img bs=1M count=10
mkfs.ext4 -N 64 /root/tiny.img
mkdir -p /mnt/tiny && mount -o loop /root/tiny.img /mnt/tiny
for i in $(seq 1 200); do touch /mnt/tiny/f$i || break; done   # "No space left on device"
df -h /mnt/tiny; df -i /mnt/tiny
umount /mnt/tiny && rm /root/tiny.img
```

</details>

---

## Exercise 7 – Hidden by a mount ★★

1. Create `/mnt/hidden/secret.txt`.
2. Mount a tmpfs on `/mnt/hidden` (`mount -t tmpfs tmpfs /mnt/hidden`). Where did the file go?
3. Unmount and check again.

**Verify:** the file is invisible while mounted and visible after `umount`.

<details>
<summary>Solution</summary>

```bash
mkdir -p /mnt/hidden && echo hi > /mnt/hidden/secret.txt
mount -t tmpfs tmpfs /mnt/hidden
ls /mnt/hidden          # empty – the original directory is covered
umount /mnt/hidden
ls /mnt/hidden          # secret.txt is back
```

</details>

---

## Challenge – Where is everything? ★★★

On **both** VMs, without using the internet, find and write into `/root/where.txt`:

1. the main system log file name (Debian vs RHEL)
2. the directory that stores installed package metadata
3. the directory with loaded kernel modules for the running kernel
4. the file listing filesystems mounted at boot
5. the file that maps hostnames to IPs locally
6. the directory with the systemd unit files shipped by packages

**Verify:** each path exists (`ls -d $(cat /root/where.txt)`).

<details>
<summary>Solution</summary>

| # | Debian/Ubuntu | RHEL family |
|---|---------------|-------------|
| 1 | `/var/log/syslog` | `/var/log/messages` |
| 2 | `/var/lib/dpkg` | `/var/lib/rpm` |
| 3 | `/lib/modules/$(uname -r)` | same |
| 4 | `/etc/fstab` | same |
| 5 | `/etc/hosts` | same |
| 6 | `/usr/lib/systemd/system` (also `/lib/systemd/system`) | `/usr/lib/systemd/system` |

</details>

---

<!-- footer:start -->
[Module exercises](README.md) · [Exercises: Finding Files →](02-finding-files.md)
<!-- footer:end -->
