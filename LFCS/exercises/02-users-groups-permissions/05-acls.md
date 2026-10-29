# Exercises – Access Control Lists

<!-- nav:start -->
> **Module 02 · Users, Groups, Permissions & LDAP** — lesson 5 of 8  
> Read first: [Theory – Access Control Lists (ACLs)](../../theory/02-users-groups-permissions/05-acls.md) · [Course lesson (commands)](../../course/02-Users-Groups-Permissions.md#lesson-5---acls-gap) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Setup

```bash
groupadd devops; groupadd audit
for u in ana luis eve auditor; do useradd -m -s /bin/bash $u; done
usermod -aG devops ana; usermod -aG devops luis; usermod -aG audit auditor
mkdir /srv/devops && chown root:devops /srv/devops && chmod 2770 /srv/devops
```

---

## Exercise 1 – Named user ★

Give `eve` (not in devops) read-write-enter access to `/srv/devops` with an ACL.

**Verify:** `ls -ld /srv/devops` shows a `+`; `su - eve -c 'touch /srv/devops/eve.txt'` works.

<details>
<summary>Solution</summary>

```bash
setfacl -m u:eve:rwx /srv/devops
getfacl /srv/devops
```

</details>

---

## Exercise 2 – Named group, read only ★

Give the `audit` group read-only access (list and enter) to `/srv/devops`.

**Verify:** as `auditor`, `ls /srv/devops` works, `touch /srv/devops/x` fails.

<details>
<summary>Solution</summary>

```bash
setfacl -m g:audit:rx /srv/devops
```

</details>

---

## Exercise 3 – The mask ★★

1. Run `chmod g=rx /srv/devops`. Look at `getfacl`. What happened to eve? Test.
2. Restore the mask to `rwx` with `setfacl`.

<details>
<summary>Solution</summary>

`chmod g=` changes the **mask** on an ACL'd file: `user:eve:rwx #effective:r-x`, so eve can no longer create files.

```bash
setfacl -m m::rwx /srv/devops
```

</details>

---

## Exercise 4 – Default ACL ★★

1. As `ana`, create `/srv/devops/before.txt`.
2. Add a **default** ACL so that `audit` gets read access to new files and directories.
3. As `ana`, create `after.txt` and a subdirectory `sub`.
4. Can `auditor` read `before.txt`? `after.txt`? Explain, then fix `before.txt`.

<details>
<summary>Solution</summary>

```bash
su - ana -c 'umask 007; echo old > /srv/devops/before.txt'
setfacl -m d:g:audit:rX /srv/devops
su - ana -c 'umask 007; echo new > /srv/devops/after.txt; mkdir /srv/devops/sub'
getfacl /srv/devops/after.txt /srv/devops/sub
su - auditor -c 'cat /srv/devops/after.txt'    # works (inherited)
su - auditor -c 'cat /srv/devops/before.txt'   # denied – existed before the default ACL
setfacl -R -m g:audit:rX /srv/devops
```

</details>

---

## Exercise 5 – Remove entries ★

1. Remove only eve's entry from `/srv/devops`.
2. On `/srv/devops/sub`, remove the default ACL but keep the access ACL.
3. On `before.txt`, remove all ACL entries.

<details>
<summary>Solution</summary>

```bash
setfacl -x u:eve /srv/devops
setfacl -k /srv/devops/sub
setfacl -b /srv/devops/before.txt
```

</details>

---

## Exercise 6 – Preserve ACLs during copy ★★

Copy `/srv/devops` to `/tmp/copy1` with `cp -r` and to `/tmp/copy2` with `cp -a`. Compare ACLs. Then back up all ACLs to `/root/devops.acl`, run `setfacl -R -b /srv/devops`, and restore.

<details>
<summary>Solution</summary>

```bash
cp -r /srv/devops /tmp/copy1; cp -a /srv/devops /tmp/copy2
getfacl /tmp/copy1 /tmp/copy2 | grep -E 'file|group:audit'
cd / && getfacl -R srv/devops > /root/devops.acl
setfacl -R -b /srv/devops
setfacl --restore=/root/devops.acl
getfacl /srv/devops
```

</details>

---

## Challenge – Exam-style task ★★★

> Create directory `/srv/reports` owned by `root:devops`. Members of `devops` must have full access, members of `audit` read-only access, and user `eve` read-write access – including for all files and directories created in the future. Nobody else may access it. New files must belong to group `devops`.

**Verify:**
```bash
su - ana -c 'mkdir /srv/reports/2026; echo hi > /srv/reports/2026/r.txt'
su - eve -c 'echo more >> /srv/reports/2026/r.txt'
su - auditor -c 'cat /srv/reports/2026/r.txt; touch /srv/reports/no' ; echo "(touch must fail)"
su - luis -c 'cat /srv/reports/2026/r.txt'
useradd -m outsider; su - outsider -c 'ls /srv/reports'   # must fail
```

<details>
<summary>Solution</summary>

```bash
mkdir /srv/reports
chown root:devops /srv/reports
chmod 2770 /srv/reports
setfacl -m g:audit:rx,u:eve:rwx /srv/reports
setfacl -m d:g:devops:rwx,d:g:audit:rx,d:u:eve:rwx,d:o::- /srv/reports
getfacl /srv/reports
```

The default `group:devops` entry makes new files group-writable even if a user's umask is `022`.

</details>

---

## Cleanup

```bash
rm -rf /srv/devops /srv/reports /tmp/copy1 /tmp/copy2 /root/devops.acl
for u in ana luis eve auditor outsider; do userdel -r $u 2>/dev/null; done
groupdel devops; groupdel audit
```

---

<!-- footer:start -->
[← Exercises: File Permissions and Ownership](04-permissions-ownership.md) · [Module exercises](README.md) · [Exercises: Personal and System-Wide Environment Profiles →](06-environment-profiles.md)
<!-- footer:end -->
