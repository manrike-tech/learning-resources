# Exercises – User Accounts

<!-- nav:start -->
> **Module 02 · Users, Groups, Permissions & LDAP** — lesson 1 of 8  
> Read first: [Theory – User Accounts](../../theory/02-users-groups-permissions/01-user-accounts.md) · [Course lesson (commands)](../../course/02-Users-Groups-Permissions.md#lesson-1---user-accounts-ch-4) · [Module exercises](README.md)
<!-- nav:end -->

> Run as root (or with sudo). Do the exercises on both VMs and notice the distribution differences.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Create a regular user ★

Create user `ana` with a home directory, shell `/bin/bash`, comment `Ana Garcia`, and set her password to `Lfcs2026!`.

**Verify:**
```bash
getent passwd ana        # ends with :/home/ana:/bin/bash
ls -ld /home/ana
sudo passwd -S ana       # status P (usable password)
```

<details>
<summary>Solution</summary>

```bash
useradd -m -s /bin/bash -c "Ana Garcia" ana
echo 'ana:Lfcs2026!' | chpasswd
```

</details>

---

## Exercise 2 – Read the account files ★

For `ana`, write into `/root/ana-fields.txt`: her UID, primary GID, home, shell, the hashing algorithm used for her password, and the date of her last password change (human readable).

<details>
<summary>Solution</summary>

```bash
getent passwd ana | awk -F: '{print "UID="$3, "GID="$4, "HOME="$6, "SHELL="$7}' > /root/ana-fields.txt
getent shadow ana | cut -d: -f2 | cut -d'$' -f2 >> /root/ana-fields.txt   # y = yescrypt, 6 = SHA-512
chage -l ana | grep "Last password change" >> /root/ana-fields.txt
```

</details>

---

## Exercise 3 – Specific UID, groups and expiry ★★

Create user `luis` with UID `2001`, supplementary groups `wheel` (RHEL) or `sudo` (Ubuntu) and a new group `projects`, account expiry `2027-06-30`.

**Verify:** `id luis` and `chage -l luis | grep "Account expires"`.

<details>
<summary>Solution</summary>

```bash
groupadd projects
ADMIN=$(getent group wheel >/dev/null && echo wheel || echo sudo)
useradd -m -s /bin/bash -u 2001 -G "$ADMIN",projects -e 2027-06-30 luis
```

</details>

---

## Exercise 4 – System account ★

Create a service account `svc_backup`: system UID, home `/var/lib/backup` (created), no interactive login.

**Verify:** `getent passwd svc_backup` shows a UID < 1000 and `/usr/sbin/nologin`; `su - svc_backup` prints "This account is currently not available."

<details>
<summary>Solution</summary>

```bash
useradd -r -m -d /var/lib/backup -s /usr/sbin/nologin svc_backup
```

</details>

---

## Exercise 5 – Password aging ★★

For `ana`:
1. Password must be changed every 60 days, not more than once a day, warn 10 days before.
2. Disable the account 7 days after the password expires.
3. Force her to change the password at next login.

**Verify:** `chage -l ana`; then `su - ana` from a non-root user (or SSH) asks for a new password.

<details>
<summary>Solution</summary>

```bash
chage -M 60 -m 1 -W 10 -I 7 ana
chage -d 0 ana
chage -l ana
```

</details>

---

## Exercise 6 – Defaults for all future users ★★

Make every account created from now on have a maximum password age of 90 days and a warning of 14 days. Create `marta` and check her aging info. Does `ana` change?

<details>
<summary>Solution</summary>

```bash
sed -i -E 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS\t90/; s/^PASS_WARN_AGE.*/PASS_WARN_AGE\t14/' /etc/login.defs
useradd -m marta
chage -l marta     # 90 / 14
chage -l ana       # unchanged – login.defs applies only to new accounts
```

</details>

---

## Exercise 7 – Modify accounts ★★

1. Change `marta`'s shell to `/bin/sh` and her comment to `Marta (QA)`.
2. Add `marta` to the `projects` group **without** removing other groups.
3. Rename `marta` to `martina` and move her home to `/home/martina`.
4. Now run `usermod -G projects luis`. What happened to luis's admin group? Fix it.

**Verify:** `id martina`, `getent passwd martina`, `id luis`.

<details>
<summary>Solution</summary>

```bash
usermod -s /bin/sh -c "Marta (QA)" marta
usermod -aG projects marta
usermod -l martina -d /home/martina -m marta
usermod -G projects luis          # luis LOST wheel/sudo – -G replaces the list
usermod -aG "$(getent group wheel >/dev/null && echo wheel || echo sudo)" luis
```

</details>

---

## Exercise 8 – Lock vs expire ★★

1. Add an SSH key for `ana` (generate one as root, append the `.pub` to `/home/ana/.ssh/authorized_keys` with correct permissions).
2. Lock `ana`'s password. Can she log in with the key (`ssh -i key ana@localhost`)?
3. Expire the account instead. Try again.
4. Restore access.

<details>
<summary>Solution</summary>

```bash
ssh-keygen -t ed25519 -f /root/ana_key -N ''
install -d -m 700 -o ana -g ana /home/ana/.ssh
install -m 600 -o ana -g ana /root/ana_key.pub /home/ana/.ssh/authorized_keys
usermod -L ana
ssh -i /root/ana_key ana@localhost true && echo "key login still works"
usermod -U ana
chage -E 0 ana
ssh -i /root/ana_key ana@localhost true || echo "account expired: blocked"
chage -E -1 ana
```

(Some distributions' sshd also refuse locked accounts when `UsePAM no`; with the default `UsePAM yes`, key login usually still works with a locked password.)

</details>

---

## Exercise 9 – Delete users and orphaned files ★★

1. As `martina`, create `/tmp/martina-notes.txt`.
2. Delete `martina` **with** her home directory.
3. Find files on the system that no longer have an owner and give them to root.

<details>
<summary>Solution</summary>

```bash
su - martina -c 'touch /tmp/martina-notes.txt'
pkill -u martina; userdel -r martina
find / -xdev -nouser 2>/dev/null
find / -xdev -nouser -exec chown root:root {} + 2>/dev/null
```

</details>

---

## Challenge – Bulk onboarding ★★★

Create `/root/newusers.csv`:

```text
pedro,Pedro Ruiz,developers
sofia,Sofia Lopez,developers
jorge,Jorge Diaz,qa
```

Write a short script `/root/onboard.sh` that, for each line, creates the group if missing, creates the user with home and bash, the given comment and supplementary group, sets a random password, forces a change at first login, and writes `user:password` to `/root/initial-passwords.txt` (mode 600).

**Verify:** `id pedro`, `chage -l sofia | head -1`, `stat -c %a /root/initial-passwords.txt`.

<details>
<summary>Solution</summary>

```bash
#!/bin/bash
set -euo pipefail
out=/root/initial-passwords.txt
install -m 600 /dev/null "$out"
while IFS=, read -r user name group; do
    getent group "$group" >/dev/null || groupadd "$group"
    useradd -m -s /bin/bash -c "$name" -G "$group" "$user"
    pass=$(openssl rand -base64 12)
    echo "$user:$pass" | chpasswd
    chage -d 0 "$user"
    echo "$user:$pass" >> "$out"
done < /root/newusers.csv
```

</details>

---

## Cleanup

```bash
for u in ana luis svc_backup pedro sofia jorge; do userdel -r $u 2>/dev/null; done
groupdel projects; groupdel developers; groupdel qa
```

---

<!-- footer:start -->
[Module exercises](README.md) · [Exercises: Groups →](02-groups.md)
<!-- footer:end -->
