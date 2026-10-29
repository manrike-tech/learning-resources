# Exercises – Finding Files

<!-- nav:start -->
> **Module 01 · Foundations & Essential Commands** — lesson 2 of 6  
> Read first: [Theory – Finding Files](../../theory/01-foundations/02-finding-files.md) · [Course lesson (commands)](../../course/01-Foundations.md#lesson-2---finding-files) · [Module exercises](README.md)
<!-- nav:end -->

> Run as root unless stated. Add `2>/dev/null` to hide permission errors when searching `/`.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Setup

Create a practice tree:

```bash
mkdir -p /srv/findlab/{logs,data,empty,"with space"}
cd /srv/findlab
for i in 1 2 3; do echo "log $i" > logs/app$i.log; done
dd if=/dev/zero of=data/big.bin bs=1M count=20 status=none
touch data/small.txt "with space/my file.txt"
touch -d "40 days ago" logs/app1.log
useradd -M findlabuser 2>/dev/null; chown findlabuser logs/app2.log
chmod 4755 data/small.txt
```

---

## Exercise 1 – By name and type ★

1. Find all `.log` files under `/srv/findlab`.
2. Find all **directories** under `/srv/findlab`.
3. Find, case-insensitively, everything named `APP*` under `/srv/findlab`.

**Verify:** you get 3 log files, 5 directories (including `/srv/findlab`), 3 matches.

<details>
<summary>Solution</summary>

```bash
find /srv/findlab -name '*.log'
find /srv/findlab -type d
find /srv/findlab -iname 'APP*'
```

</details>

---

## Exercise 2 – By size and emptiness ★

1. Find files larger than 10 MiB under `/srv/findlab`.
2. Find empty files and empty directories.

**Verify:** `big.bin` for (1); `small.txt`, `my file.txt` and `empty/` for (2).

<details>
<summary>Solution</summary>

```bash
find /srv/findlab -type f -size +10M
find /srv/findlab -empty
```

</details>

---

## Exercise 3 – By owner, time and permission ★★

1. Find files owned by `findlabuser`.
2. Find files **not** modified in the last 30 days.
3. Find SUID files under `/srv/findlab`.
4. Find all SUID files on the system and save the sorted list to `/root/suid.txt`.

**Verify:** (1) `app2.log`, (2) `app1.log`, (3) `small.txt`, (4) `wc -l /root/suid.txt` > 5 and contains `/usr/bin/passwd`.

<details>
<summary>Solution</summary>

```bash
find /srv/findlab -user findlabuser
find /srv/findlab -type f -mtime +30
find /srv/findlab -perm -4000
find / -type f -perm -4000 2>/dev/null | sort > /root/suid.txt
grep passwd /root/suid.txt
```

</details>

---

## Exercise 4 – Combining tests ★★

Find files under `/srv/findlab` that are either `.log` **or** `.bin`, and are larger than 1 KiB.

**Verify:** only `big.bin` matches (log files are tiny).

<details>
<summary>Solution</summary>

```bash
find /srv/findlab \( -name '*.log' -o -name '*.bin' \) -size +1k
```

Without the parentheses the `-size` test would bind only to `-name '*.bin'`.

</details>

---

## Exercise 5 – Acting on results ★★

1. Change the permission of every file under `/srv/findlab/logs` to `640` using `-exec`.
2. Compress all `.log` files older than 30 days with `gzip` (one command).
3. Count the total size of all files under `/srv/findlab` using `find … -print0 | xargs -0 du -ch`.
4. Handle the file name with a space correctly – prove it by listing it with `-exec ls -l {} +`.

**Verify:** `ls -l /srv/findlab/logs` shows `-rw-r-----` and `app1.log.gz`.

<details>
<summary>Solution</summary>

```bash
find /srv/findlab/logs -type f -exec chmod 640 {} +
find /srv/findlab -name '*.log' -mtime +30 -exec gzip {} \;
find /srv/findlab -type f -print0 | xargs -0 du -ch | tail -1
find /srv/findlab -name '*space*' -exec ls -ld {} +
find /srv/findlab -name 'my file.txt' -exec ls -l {} +
```

</details>

---

## Exercise 6 – locate vs find ★

1. Install `plocate` (Ubuntu) or `plocate`/`mlocate` (RHEL).
2. Create `/srv/findlab/brandnew.txt` and try `locate brandnew.txt`. Explain the result.
3. Fix it and search again.

**Verify:** the second `locate` finds the file.

<details>
<summary>Solution</summary>

```bash
apt install -y plocate || dnf install -y plocate || dnf install -y mlocate
touch /srv/findlab/brandnew.txt
locate brandnew.txt        # nothing: database is older than the file
updatedb
locate brandnew.txt
```

</details>

---

## Exercise 7 – Where does this command come from? ★

For `ls`, `cd`, `ll`, `passwd` and `sshd`, record what *kind* of command each is and its path (if any) in `/root/commands.txt`.

**Verify:** your file says `ls` is an alias (on most distros), `cd` a builtin, `passwd` in `/usr/bin`, `sshd` in `/usr/sbin`.

<details>
<summary>Solution</summary>

```bash
{ type -a ls cd ll passwd sshd; } > /root/commands.txt 2>&1
cat /root/commands.txt
```

</details>

---

## Challenge – The audit report ★★★

Write a single command line (you may use `;` or `&&`) that creates `/root/audit.txt` containing:

1. a header line `== World-writable files ==` followed by all world-writable **regular files** outside `/proc` and `/sys`;
2. a header `== Files without owner ==` followed by files whose owner no longer exists (`userdel findlabuser` first to create some);
3. a header `== Changed in /etc in last 24h ==` followed by those files.

**Verify:** `cat /root/audit.txt` shows three headers with lists below.

<details>
<summary>Solution</summary>

```bash
userdel findlabuser
{
  echo "== World-writable files =="
  find / \( -path /proc -o -path /sys \) -prune -o -type f -perm -o+w -print 2>/dev/null
  echo "== Files without owner =="
  find / \( -path /proc -o -path /sys \) -prune -o -nouser -print 2>/dev/null
  echo "== Changed in /etc in last 24h =="
  find /etc -type f -mtime -1 2>/dev/null
} > /root/audit.txt
```

`-path X -prune -o … -print` is the idiom for "skip these directories".

</details>

---

## Cleanup

```bash
rm -rf /srv/findlab
```

---

<!-- footer:start -->
[← Exercises: The Linux Filesystem Hierarchy](01-filesystem-hierarchy.md) · [Module exercises](README.md) · [Exercises: Working with Text →](03-working-with-text.md)
<!-- footer:end -->
