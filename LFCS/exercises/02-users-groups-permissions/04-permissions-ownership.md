# Exercises – File Permissions and Ownership

<!-- nav:start -->
> **Module 02 · Users, Groups, Permissions & LDAP** — lesson 4 of 8  
> Read first: [Theory – File Permissions and Ownership](../../theory/02-users-groups-permissions/04-permissions-ownership.md) · [Course lesson (commands)](../../course/02-Users-Groups-Permissions.md#lesson-4---permissions--ownership-ch-6) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Setup

```bash
groupadd sales; for u in ana luis eva; do useradd -m -s /bin/bash $u; done
usermod -aG sales ana; usermod -aG sales luis
mkdir -p /srv/permlab
```

---

## Exercise 1 – Octal and symbolic ★

Create `/srv/permlab/report.txt`. Using **octal**, set `rw-r-----`. Then, using **symbolic** notation only, add write for the group and remove all access for others (already none).

**Verify:** `stat -c '%A %a' /srv/permlab/report.txt` → `-rw-rw---- 660`.

<details>
<summary>Solution</summary>

```bash
touch /srv/permlab/report.txt
chmod 640 /srv/permlab/report.txt
chmod g+w,o= /srv/permlab/report.txt
```

</details>

---

## Exercise 2 – Ownership ★

Make `report.txt` owned by `ana` and group `sales` with one command. Then as `ana`, try to change its owner to `luis`. What happens?

<details>
<summary>Solution</summary>

```bash
chown ana:sales /srv/permlab/report.txt
su - ana -c 'chown luis /srv/permlab/report.txt'   # Operation not permitted – only root can change owners
```

</details>

---

## Exercise 3 – Predict the access ★★

`/srv/permlab/odd.txt` is owned by `ana:sales` with mode `0407` (`-r-----rwx`). Predict, then test, whether `ana`, `luis` (in sales) and `eva` (not in sales) can **write** to it.

<details>
<summary>Solution</summary>

```bash
touch /srv/permlab/odd.txt; chown ana:sales /srv/permlab/odd.txt; chmod 0407 /srv/permlab/odd.txt
for u in ana luis eva; do su - $u -c 'echo x >> /srv/permlab/odd.txt' && echo "$u: yes" || echo "$u: no"; done
```

ana (owner bits `r--`): no · luis (group bits `---`): no · eva (other bits `rwx`): **yes**. Only one class applies.

</details>

---

## Exercise 4 – Directory permissions ★★

1. Create `/srv/permlab/dir` with a file `inside.txt`. Set the directory to `744` and the file to `644`. As `eva`, try `ls` the directory and `cat` the file.
2. Change the directory to `711`. Try again.
3. Explain each result.

<details>
<summary>Solution</summary>

- `744` (`r--` for others): `ls` lists names (with errors for details), `cat` fails – no `x` to traverse.
- `711` (`--x` for others): `ls` fails (no `r`), but `cat /srv/permlab/dir/inside.txt` works because the exact name is known and `x` allows traversal.

</details>

---

## Exercise 5 – Deleting files ★★

As `ana`, create `/srv/permlab/shared/ana.txt` in a directory with mode `777`. As `eva`, delete it. Then add the sticky bit and repeat.

<details>
<summary>Solution</summary>

```bash
mkdir -m 777 /srv/permlab/shared
su - ana -c 'touch /srv/permlab/shared/ana.txt'
su - eva -c 'rm -f /srv/permlab/shared/ana.txt'   # works: w on the directory is enough
chmod +t /srv/permlab/shared                     # 1777
su - ana -c 'touch /srv/permlab/shared/ana.txt'
su - eva -c 'rm -f /srv/permlab/shared/ana.txt'   # Operation not permitted
```

</details>

---

## Exercise 6 – SGID team directory ★★

Create `/srv/sales` so that members of `sales` can create and edit files, new files automatically get group `sales`, and non-members have no access.

**Verify:** `ls -ld /srv/sales` → `drwxrws---`; a file created by ana has group `sales`.

<details>
<summary>Solution</summary>

```bash
mkdir /srv/sales && chgrp sales /srv/sales && chmod 2770 /srv/sales
su - ana -c 'touch /srv/sales/q1.txt'; ls -l /srv/sales/q1.txt
```

</details>

---

## Exercise 7 – Find special bits ★

Save into `/root/special.txt` all SUID **or** SGID regular files on the system, with their mode in octal.

<details>
<summary>Solution</summary>

```bash
find / -xdev -type f -perm /6000 -printf '%m %p\n' 2>/dev/null | sort -k2 > /root/special.txt
```

</details>

---

## Exercise 8 – umask ★★

1. In a subshell, set umask so that new files are `640` and new directories `750`. Create one of each and verify.
2. Make that umask the default for user `luis` only.

<details>
<summary>Solution</summary>

```bash
( umask 027; touch /tmp/u.f; mkdir /tmp/u.d; stat -c '%a %n' /tmp/u.f /tmp/u.d )
echo 'umask 027' >> /home/luis/.bashrc
su - luis -c 'umask'     # 0027
```

</details>

---

## Exercise 9 – Immutable files ★

Make `/srv/permlab/report.txt` immutable. As root, try to append to it, rename it and delete it. Remove the attribute.

<details>
<summary>Solution</summary>

```bash
chattr +i /srv/permlab/report.txt
echo x >> /srv/permlab/report.txt     # Operation not permitted
mv /srv/permlab/report.txt /tmp/      # Operation not permitted
lsattr /srv/permlab/report.txt
chattr -i /srv/permlab/report.txt
```

</details>

---

## Challenge – Fix a broken web tree ★★★

Run this to create a mess:

```bash
mkdir -p /srv/site/{css,img,cgi}
touch /srv/site/index.html /srv/site/css/a.css /srv/site/img/logo.png /srv/site/cgi/run.sh
chmod -R 777 /srv/site; chown -R eva:eva /srv/site
```

Fix it so that:
- everything is owned by `root:sales`,
- directories are `2755`, regular files `644`,
- only `cgi/run.sh` is executable (`755`),
- no file is world-writable.

Use `find` – no manual `chmod` on individual files except `run.sh`.

**Verify:** `find /srv/site -perm -o+w` returns nothing; `find /srv/site -type f -perm -u+x` returns only `run.sh`.

<details>
<summary>Solution</summary>

```bash
chown -R root:sales /srv/site
find /srv/site -type d -exec chmod 2755 {} +
find /srv/site -type f -exec chmod 644 {} +
chmod 755 /srv/site/cgi/run.sh
```

</details>

---

## Cleanup

```bash
rm -rf /srv/permlab /srv/sales /srv/site /tmp/u.f /tmp/u.d
for u in ana luis eva; do userdel -r $u; done; groupdel sales
```

---

<!-- footer:start -->
[← Exercises: Privilege Escalation with sudo and su](03-sudo.md) · [Module exercises](README.md) · [Exercises: Access Control Lists (ACLs) →](05-acls.md)
<!-- footer:end -->
