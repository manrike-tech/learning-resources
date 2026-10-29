# Exercises – sudo and su

<!-- nav:start -->
> **Module 02 · Users, Groups, Permissions & LDAP** — lesson 3 of 8  
> Read first: [Theory – Privilege Escalation with sudo and su](../../theory/02-users-groups-permissions/03-sudo.md) · [Course lesson (commands)](../../course/02-Users-Groups-Permissions.md#lesson-3---sudo) · [Module exercises](README.md)
<!-- nav:end -->

> Keep a root shell open in a second terminal while doing these exercises, in case you break sudo.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Setup

```bash
for u in admin1 op1 dev1; do useradd -m -s /bin/bash $u; echo "$u:Passw0rd!" | chpasswd; done
```

---

## Exercise 1 – su vs su - ★

As `dev1`, run `su admin1` and then `su - admin1`. In each, print `pwd`, `echo $HOME` and `echo $PATH`. Note the differences.

<details>
<summary>Solution</summary>

`su admin1` keeps dev1's working directory and most environment; `su - admin1` starts a login shell: `pwd` = `/home/admin1`, `HOME=/home/admin1`, PATH reset by admin1's profile.

</details>

---

## Exercise 2 – Full administrator ★

Give `admin1` full sudo rights using the distribution's admin group.

**Verify:** `sudo -l -U admin1` shows `(ALL : ALL) ALL` or `(ALL) ALL`; `su - admin1 -c 'sudo -k; echo Passw0rd! | sudo -S id -u'` prints `0`.

<details>
<summary>Solution</summary>

```bash
usermod -aG sudo admin1     # Ubuntu
usermod -aG wheel admin1    # RHEL
```

</details>

---

## Exercise 3 – A restricted rule ★★

Allow `op1` to run **only** `systemctl restart sshd` (Ubuntu: `ssh`) and `systemctl status sshd` as root, without a password. Use a drop-in file.

**Verify:**
```bash
visudo -c
sudo -l -U op1
su - op1 -c 'sudo systemctl status sshd | head -2'   # works (use ssh on Ubuntu)
su - op1 -c 'sudo systemctl stop sshd'                           # refused
```

<details>
<summary>Solution</summary>

```bash
visudo -f /etc/sudoers.d/op1
```
```text
op1 ALL=(root) NOPASSWD: /usr/bin/systemctl restart sshd, /usr/bin/systemctl status sshd
```

Check the path with `command -v systemctl`. The arguments must match exactly: `sudo systemctl status sshd --no-pager` would be refused, because `--no-pager` is not in the rule. (Piping into `head` disables the pager anyway.)

</details>

---

## Exercise 4 – Group rule with alias ★★

1. Create group `dbadmins` and add `dev1`.
2. Define a `Cmnd_Alias` `DBCMDS` with `/usr/bin/journalctl -u postgresql` and `/usr/bin/systemctl restart postgresql`.
3. Allow `%dbadmins` to run `DBCMDS` as root with a password.

**Verify:** `sudo -l -U dev1`.

<details>
<summary>Solution</summary>

```bash
groupadd dbadmins; usermod -aG dbadmins dev1
visudo -f /etc/sudoers.d/dbadmins
```
```text
Cmnd_Alias DBCMDS = /usr/bin/journalctl -u postgresql, /usr/bin/systemctl restart postgresql
%dbadmins ALL=(root) DBCMDS
```

</details>

---

## Exercise 5 – Run as a non-root user ★★

Allow `dev1` to run any command as user `op1` (but not as root). Test with `sudo -u op1 whoami`.

<details>
<summary>Solution</summary>

```text
# /etc/sudoers.d/dev1-as-op1
dev1 ALL=(op1) ALL
```

`su - dev1 -c 'sudo -u op1 whoami'` → `op1`; `sudo whoami` → refused.

</details>

---

## Exercise 6 – visudo protects you ★

With `visudo -f /etc/sudoers.d/broken`, write `dev1 ALL=(ALL) ALL ALL` (invalid) and try to save. Choose **x** to exit without saving. Then check that no file was created.

<details>
<summary>Solution</summary>

visudo reports `syntax error near line 1` and asks *What now?*. `x` discards. `ls /etc/sudoers.d/broken` → No such file.

</details>

---

## Exercise 7 – Ignored drop-in ★★

Create `/etc/sudoers.d/dev1.conf` granting `dev1 ALL=(ALL) ALL`. Does it work? Why? Fix it.

<details>
<summary>Solution</summary>

It's ignored because the file name contains a dot. Rename: `mv /etc/sudoers.d/dev1.conf /etc/sudoers.d/dev1` (and keep mode 0440).

</details>

---

## Exercise 8 – Audit ★

Find the log lines that show every command `op1` ran with sudo today.

<details>
<summary>Solution</summary>

```bash
journalctl _COMM=sudo --since today | grep op1
grep 'sudo:.*op1' /var/log/auth.log   # Ubuntu
grep 'sudo:.*op1' /var/log/secure     # RHEL
```

</details>

---

## Challenge – Safe file editing delegation ★★★

User `dev1` must be able to edit `/etc/hosts` and `/etc/motd` as root, but must **not** be able to get a root shell. Write the rule, test it, and explain why `NOPASSWD: /usr/bin/vim /etc/hosts` would be a bad solution.

<details>
<summary>Solution</summary>

```text
# /etc/sudoers.d/dev1-edit
dev1 ALL=(root) sudoedit /etc/hosts, sudoedit /etc/motd
```

`su - dev1 -c 'sudoedit /etc/hosts'` edits a temporary copy with dev1's own editor, running as dev1, and copies it back as root. With `vim` run as root, `:!bash` gives a root shell.

</details>

---

## Cleanup

```bash
rm -f /etc/sudoers.d/{op1,dbadmins,dev1-as-op1,dev1,dev1-edit,dev1.conf}
for u in admin1 op1 dev1; do userdel -r $u; done; groupdel dbadmins
```

---

<!-- footer:start -->
[← Exercises: Groups](02-groups.md) · [Module exercises](README.md) · [Exercises: File Permissions and Ownership →](04-permissions-ownership.md)
<!-- footer:end -->
