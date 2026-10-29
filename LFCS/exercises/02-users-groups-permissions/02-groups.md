# Exercises – Groups

<!-- nav:start -->
> **Module 02 · Users, Groups, Permissions & LDAP** — lesson 2 of 8  
> Read first: [Theory – Groups](../../theory/02-users-groups-permissions/02-groups.md) · [Course lesson (commands)](../../course/02-Users-Groups-Permissions.md#lesson-2---groups-ch-5) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Setup

```bash
for u in ana luis eva; do useradd -m -s /bin/bash $u; done
```

---

## Exercise 1 – Create and inspect groups ★

1. Create group `devops` with GID `5000` and a system group `monitoring`.
2. Show both with `getent`.

**Verify:** `getent group devops monitoring` – devops has GID 5000; monitoring has a GID below 1000.

<details>
<summary>Solution</summary>

```bash
groupadd -g 5000 devops
groupadd -r monitoring
getent group devops monitoring
```

</details>

---

## Exercise 2 – Membership ★

1. Add `ana` and `luis` to `devops` using two different commands.
2. Add `eva` to both `devops` and `monitoring` in one command.
3. Remove `luis` from `devops`.

**Verify:** `getent group devops` → `ana,eva`; `id eva` shows both groups.

<details>
<summary>Solution</summary>

```bash
usermod -aG devops ana
gpasswd -a luis devops
usermod -aG devops,monitoring eva
gpasswd -d luis devops
```

</details>

---

## Exercise 3 – Primary vs supplementary ★★

1. Make `devops` the **primary** group of `luis`.
2. As `luis`, create `/tmp/luis.txt` and check its group.
3. Run `getent group devops` – is luis listed? Explain.

<details>
<summary>Solution</summary>

```bash
usermod -g devops luis
su - luis -c 'touch /tmp/luis.txt'; ls -l /tmp/luis.txt   # group devops
getent group devops    # luis NOT listed: primary membership lives in /etc/passwd
```

</details>

---

## Exercise 4 – Session refresh ★★

1. Open a shell as `ana` (`su - ana`) and keep it open.
2. In another terminal, as root, add `ana` to `monitoring`.
3. In ana's shell, run `id -nG`. Is `monitoring` there?
4. Make it visible without logging out; then do it properly.

<details>
<summary>Solution</summary>

The existing session doesn't see the new group. `newgrp monitoring` starts a subshell with it; `exit` and `su - ana` again for a clean login.

</details>

---

## Exercise 5 – Group administrator ★★

Make `ana` an administrator of `devops`, then — as `ana`, without sudo — add `luis` back to the group.

**Verify:** `getent group devops` includes luis.

<details>
<summary>Solution</summary>

```bash
gpasswd -A ana devops
su - ana -c 'gpasswd -a luis devops'
```

</details>

---

## Exercise 6 – Rename, change GID, delete ★★

1. Rename `monitoring` to `observability`.
2. Create `/srv/obs.txt` owned by group `observability`, then change the group's GID to `5100`. What does `ls -l /srv/obs.txt` show now? Fix it.
3. Try to delete `devops`. Why does it fail? Make it succeed.

<details>
<summary>Solution</summary>

```bash
groupmod -n observability monitoring
touch /srv/obs.txt && chgrp observability /srv/obs.txt
OLD=$(getent group observability | cut -d: -f3)
groupmod -g 5100 observability
ls -l /srv/obs.txt                       # shows the old number
find / -xdev -gid "$OLD" -exec chgrp observability {} + 2>/dev/null
groupdel devops                          # fails: primary group of luis
usermod -g luis luis 2>/dev/null || { groupadd luis; usermod -g luis luis; }
groupdel devops
```

</details>

---

## Challenge – Team directory ★★★

Create a group `webteam` with members `ana` and `eva`, and a directory `/srv/webteam` where:

- only root and members of `webteam` can enter,
- every file created inside belongs to `webteam` automatically,
- both members can edit each other's files.

Prove it by creating a file as ana and appending to it as eva.

<details>
<summary>Solution</summary>

```bash
groupadd webteam
gpasswd -M ana,eva webteam
mkdir /srv/webteam
chgrp webteam /srv/webteam
chmod 2770 /srv/webteam
su - ana -c 'umask 002; echo "from ana" > /srv/webteam/page.html'
su - eva -c 'echo "from eva" >> /srv/webteam/page.html'
ls -l /srv/webteam/page.html; cat /srv/webteam/page.html
```

The umask must allow group write (`002`); for a permanent solution add a default ACL (next lessons).

</details>

---

## Cleanup

```bash
for u in ana luis eva; do userdel -r $u; done
groupdel observability; groupdel webteam; groupdel luis 2>/dev/null
rm -rf /srv/webteam /srv/obs.txt /tmp/luis.txt
```

---

<!-- footer:start -->
[← Exercises: User Accounts](01-user-accounts.md) · [Module exercises](README.md) · [Exercises: Privilege Escalation with sudo and su →](03-sudo.md)
<!-- footer:end -->
