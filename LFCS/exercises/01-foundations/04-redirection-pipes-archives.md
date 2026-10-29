# Exercises – Redirection, Pipes and Archives

<!-- nav:start -->
> **Module 01 · Foundations & Essential Commands** — lesson 4 of 6  
> Read first: [Theory – Redirection, Pipes and Archives](../../theory/01-foundations/04-redirection-pipes-archives.md) · [Course lesson (commands)](../../course/01-Foundations.md#lesson-4---redirection-pipes-archives) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Separating the streams ★

As a **normal user**, run `find /etc -name "*.conf"`:

1. Save results to `~/found.txt` and errors to `~/errors.txt`.
2. Run it again saving **both** into `~/all.txt`.
3. Run it a third time showing only results on screen (discard errors).

**Verify:** `wc -l ~/found.txt ~/errors.txt ~/all.txt` – the `all.txt` count equals the sum of the other two.

<details>
<summary>Solution</summary>

```bash
find /etc -name "*.conf" > ~/found.txt 2> ~/errors.txt
find /etc -name "*.conf" > ~/all.txt 2>&1
find /etc -name "*.conf" 2>/dev/null
wc -l ~/found.txt ~/errors.txt ~/all.txt
```

</details>

---

## Exercise 2 – The order of redirections ★★

Predict, then test, where errors go in each case:

```bash
ls /etc/hostname /nope > a.txt 2>&1
ls /etc/hostname /nope 2>&1 > b.txt
```

**Verify:** `a.txt` has 2 lines; `b.txt` has 1 line and the error appeared on screen.

<details>
<summary>Solution</summary>

Redirections are processed left to right. In the second command, FD 2 is pointed at the terminal (where FD 1 pointed *at that moment*) before FD 1 is redirected to the file.

</details>

---

## Exercise 3 – Here-documents and sudo ★★

1. As a normal user with sudo, create `/etc/motd` containing two lines: `Welcome to <hostname>` (expanded) and `Authorized use only`.
2. Append a third line `Maintenance: Sundays 02:00` without overwriting.
3. Explain why `sudo echo x > /etc/motd` fails.

**Verify:** `cat /etc/motd` shows three lines with your real hostname.

<details>
<summary>Solution</summary>

```bash
sudo tee /etc/motd > /dev/null << EOF
Welcome to $(hostname)
Authorized use only
EOF
echo "Maintenance: Sundays 02:00" | sudo tee -a /etc/motd
```

The redirection `>` is performed by your unprivileged shell before `sudo` runs.

</details>

---

## Exercise 4 – Pipes, tee and exit codes ★★

1. Save the output of `ip -br addr` into `/tmp/ip.txt` while also displaying it.
2. Print `config OK` only if `sshd -t` succeeds, otherwise print `config BROKEN`.
3. Show the exit status of `grep -q nosuchuser /etc/passwd`.
4. Explain why `false | true; echo $?` prints `0`, and make the pipeline report failure.

**Verify:** (3) prints `1`; with `set -o pipefail`, (4) prints `1`.

<details>
<summary>Solution</summary>

```bash
ip -br addr | tee /tmp/ip.txt
sudo sshd -t && echo "config OK" || echo "config BROKEN"
grep -q nosuchuser /etc/passwd; echo $?
false | true; echo $?                     # status of the LAST command
set -o pipefail; false | true; echo $?    # 1
set +o pipefail
```

</details>

---

## Exercise 5 – tar archives ★

1. Create `/root/etc-backup.tar.gz` containing `/etc`.
2. List its contents and count entries.
3. Extract only `etc/hosts` and `etc/passwd` into `/tmp/restore`.
4. Create the same archive with xz and with bzip2; compare sizes.

**Verify:** `/tmp/restore/etc/hosts` exists; `ls -lh /root/etc-backup.*` shows three sizes.

<details>
<summary>Solution</summary>

```bash
tar -czf /root/etc-backup.tar.gz /etc
tar -tzf /root/etc-backup.tar.gz | wc -l
mkdir -p /tmp/restore
tar -xzf /root/etc-backup.tar.gz -C /tmp/restore etc/hosts etc/passwd
tar -cJf /root/etc-backup.tar.xz /etc
tar -cjf /root/etc-backup.tar.bz2 /etc
ls -lh /root/etc-backup.*
```

</details>

---

## Exercise 6 – Preserving metadata ★★

1. Create `/srv/meta/file` owned by `nobody`, mode `640`, with an ACL `u:root:rwx` (`setfacl`).
2. Archive it *without* ACL support, extract to `/tmp/m1`; then archive *with* `--acls --xattrs`, extract to `/tmp/m2`.
3. Compare `getfacl` on both restored copies.

**Verify:** only the `/tmp/m2` copy has the named user entry.

<details>
<summary>Solution</summary>

```bash
mkdir -p /srv/meta && echo x > /srv/meta/file
chown nobody /srv/meta/file && chmod 640 /srv/meta/file
setfacl -m u:root:rwx /srv/meta/file
tar -cpf /tmp/noacl.tar /srv/meta
tar --acls --xattrs -cpf /tmp/acl.tar /srv/meta
mkdir -p /tmp/m1 /tmp/m2
tar -xpf /tmp/noacl.tar -C /tmp/m1
tar --acls --xattrs -xpf /tmp/acl.tar -C /tmp/m2
getfacl /tmp/m1/srv/meta/file /tmp/m2/srv/meta/file
```

</details>

---

## Exercise 7 – Standalone compressors ★

1. Copy `/var/log/dpkg.log` or `/var/log/dnf.log` to `/tmp/big.log`. Compress it with gzip **keeping** the original.
2. Search inside the compressed file without decompressing it.
3. Decompress it to a different name with `gunzip -c`.

**Verify:** `ls /tmp/big.log*` shows the original and the `.gz`.

<details>
<summary>Solution</summary>

```bash
cp /var/log/dpkg.log /tmp/big.log 2>/dev/null || cp /var/log/dnf.log /tmp/big.log
gzip -k /tmp/big.log
zgrep -c install /tmp/big.log.gz
gunzip -c /tmp/big.log.gz > /tmp/big-copy.log
```

</details>

---

## Challenge – Dated backup one-liner ★★★

Write a single command line that:

- creates `/backup` if missing,
- archives `/etc` and `/home` with zstd (or xz) into `/backup/sys-YYYY-MM-DD.tar.zst`, excluding `*.cache` directories,
- writes the list of archived files to `/backup/sys-YYYY-MM-DD.list`,
- prints `BACKUP OK` only if tar succeeded.

**Verify:** both files exist and `tar -tf` works on the archive.

<details>
<summary>Solution</summary>

```bash
d=$(date +%F); mkdir -p /backup && \
tar --zstd --exclude='*.cache' -cvf /backup/sys-$d.tar.zst /etc /home > /backup/sys-$d.list 2>/dev/null && \
echo "BACKUP OK"
```

(`-v` lists files on stdout, which we redirect to the list file.)

</details>

---

<!-- footer:start -->
[← Exercises: Working with Text](03-working-with-text.md) · [Module exercises](README.md) · [Exercises: The User Environment →](05-user-environment.md)
<!-- footer:end -->
