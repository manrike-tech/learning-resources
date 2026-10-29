# Privilege Escalation with sudo and su

<!-- nav:start -->
> **Module 02 · Users, Groups, Permissions & LDAP** — lesson 3 of 8  
> [Module index](README.md) · [Course lesson (commands)](../../course/02-Users-Groups-Permissions.md#lesson-3---sudo) · [Exercises for this lesson](../../exercises/02-users-groups-permissions/03-sudo.md)
<!-- nav:end -->

Logging in as root for daily work is dangerous: one typo can destroy the system, and nobody can tell who did what. Linux offers two ways to temporarily gain other privileges: `su` (switch user) and `sudo` (run one command as another user, according to rules). In this lesson you will learn how both work and how to write precise `sudo` rules.

## What you will learn

- The principle of **least privilege**
- `su` vs `su -` vs `sudo` vs `sudo -i`
- How `sudo` decides what you may run: the `sudoers` file
- Writing rules for users, groups, specific commands and `NOPASSWD`
- Editing safely with `visudo` and drop-in files
- Where `sudo` logs and how to check your rights

---

## Key terms

| Term | Definition |
|------|-----------|
| **root / superuser** | UID 0; bypasses most permission checks. |
| **Least privilege** | Give each user only the rights needed for their job. |
| **`su`** | *Substitute user*: start a shell as another user (root by default); asks for the **target user's** password. |
| **`sudo`** | Run a command as another user if the **sudoers policy** allows it; asks for **your own** password. |
| **sudoers** | The policy file `/etc/sudoers` plus files in `/etc/sudoers.d/`. |
| **`visudo`** | The safe editor for sudoers: locks the file and checks syntax before saving. |
| **Admin group** | Group whose members get full sudo: `sudo` (Debian/Ubuntu), `wheel` (RHEL). |

---

## su

```console
$ su                    # become root, keep MY environment (PATH, HOME, cwd)
Password:               ← root's password
# exit
$ su -                  # become root with a full LOGIN environment (root's HOME, PATH)
$ su - alice            # become alice (her password)
$ su - alice -c "whoami; pwd"   # run one command as alice
alice
/home/alice
```

- `su` without `-` keeps your environment – this can lead to surprises (e.g. root running programs from your `PATH`). **Prefer `su -`.**
- Root can `su` to any user without a password.
- On Ubuntu the root account has **no password** by default, so `su` fails; use `sudo -i` instead.

---

## sudo

```console
$ sudo systemctl restart nginx
[sudo] password for alice:     ← YOUR password
$ sudo -u postgres psql        # run as another user (not root)
$ sudo -i                      # interactive root LOGIN shell (like su -)
$ sudo -s                      # root shell keeping your environment
$ sudo -l                      # list what I'm allowed to run
$ sudo -k                      # forget cached credentials now
$ sudoedit /etc/hosts          # edit a file as root using YOUR editor safely (= sudo -e)
```

After a successful password prompt, sudo caches your credentials for a few minutes (the `timestamp_timeout` setting, 5 minutes in upstream sudo), per terminal.

### Why sudo is better than sharing the root password

| | Shared root password + `su` | `sudo` |
|--|-----------------------------|--------|
| Who knows the secret | everyone | only each user knows their own password |
| Granularity | everything | per user, per command, per host |
| Audit trail | "root did it" | every command logged with the real user name |
| Revoke access | change root password for everyone | remove one rule / group membership |

---

## How sudo decides: the sudoers file

`/etc/sudoers` is read top to bottom; it usually ends with `@includedir /etc/sudoers.d` (or `#includedir` – the `#` is **not** a comment here!), so each file in `/etc/sudoers.d/` is included. When several rules match, the **last** matching one wins.

### Rule syntax

```text
WHO     WHERE = (AS_WHOM)   [TAGS:] WHAT
alice   ALL   = (ALL:ALL)   ALL
%ops    ALL   = (root)      NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl reload nginx
```

| Part | Meaning | Examples |
|------|---------|----------|
| WHO | user, `%group`, or alias | `alice`, `%wheel`, `%sudo` |
| WHERE | host the rule applies to (useful when sudoers is shared between machines) | `ALL`, `web01` |
| AS_WHOM | `(user)` or `(user:group)` you may become | `(ALL)`, `(root)`, `(postgres)` |
| TAGS | options | `NOPASSWD:`, `PASSWD:`, `NOEXEC:` |
| WHAT | commands, **with full paths**, optionally with arguments | `ALL`, `/usr/bin/systemctl restart nginx` |

### Default admin rules

```text
# Debian/Ubuntu
%sudo   ALL=(ALL:ALL) ALL
# RHEL family
%wheel  ALL=(ALL)     ALL
```

So "make alice an administrator" is simply `usermod -aG sudo alice` (Ubuntu) or `usermod -aG wheel alice` (RHEL).

### Command restrictions

- Commands must be **absolute paths** (`which systemctl` → `/usr/bin/systemctl`).
- `/usr/bin/systemctl restart nginx` allows exactly those arguments. `/usr/bin/systemctl` (no arguments) allows **any** arguments. `""` as the argument list means "no arguments allowed".
- Wildcards are possible (`/usr/bin/systemctl * nginx`) but risky – `*` can match spaces and extra arguments.

> **Security warning:** many programs can spawn a shell: `vi`, `less`, `find -exec`, `tar`, `awk`, `python`… Allowing `sudo vi /etc/hosts` effectively gives full root (`:!bash`). Use `sudoedit` for file editing rights.

### Aliases

```text
User_Alias   WEBADMINS = alice, bob
Cmnd_Alias   WEBCMDS   = /usr/bin/systemctl restart nginx, /usr/sbin/nginx -t
WEBADMINS    ALL = (root) NOPASSWD: WEBCMDS
```

### Defaults

```text
Defaults    timestamp_timeout=10       # cache credentials 10 minutes
Defaults    logfile=/var/log/sudo.log  # extra log file
Defaults:alice  !requiretty
Defaults    secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
```

`secure_path` is the `PATH` used for commands run through sudo – which is why a script in your `~/bin` may not be found with `sudo`.

---

## Editing safely with visudo

A syntax error in sudoers can make **sudo unusable for everyone**, locking you out of root on systems without a root password.

```console
$ sudo visudo                           # edit /etc/sudoers
$ sudo visudo -f /etc/sudoers.d/webadmins   # create/edit a drop-in file
$ sudo visudo -c                        # check all sudoers files
/etc/sudoers: parsed OK
/etc/sudoers.d/webadmins: parsed OK
```

If you save a broken file, visudo tells you and offers: `What now? (e)dit again, e(x)it without saving, (Q)uit and save anyway`. Press **e** and fix it.

Drop-in file rules:

- Put your own rules in `/etc/sudoers.d/` – package upgrades never overwrite them.
- File names must **not** contain a `.` or end with `~` (they would be ignored): `webadmins` ✔, `webadmins.conf` ✘.
- Permissions must be `0440`, owner root (visudo does this).

Set the editor used by visudo: `sudo EDITOR=vim visudo` or `export EDITOR=vim`.

---

## Example: delegate service restarts to a team

Goal: members of `ops` may restart and check nginx without a password, and nothing else.

```console
$ sudo groupadd ops; sudo usermod -aG ops bob
$ sudo visudo -f /etc/sudoers.d/ops
```

```text
%ops ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx, /usr/sbin/nginx -t
```

Test as bob (after a new login):

```console
bob$ sudo -l
User bob may run the following commands on web01:
    (root) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx, /usr/sbin/nginx -t
bob$ sudo systemctl restart nginx          ✔ no password
bob$ sudo systemctl stop nginx
Sorry, user bob is not allowed to execute '/usr/bin/systemctl stop nginx' as root on web01.
```

---

## Logging

Every sudo use is logged with the real user:

```console
$ sudo journalctl _COMM=sudo -n 3
Oct 21 11:02:13 web01 sudo[4321]:   bob : TTY=pts/1 ; PWD=/home/bob ; USER=root ; COMMAND=/usr/bin/systemctl restart nginx
```

Also in `/var/log/auth.log` (Debian/Ubuntu) or `/var/log/secure` (RHEL).

---

## Try it yourself

1. Check what you may run: `sudo -l`.
2. Compare `su` and `su -` by running `pwd; echo $PATH` in each.
3. Create a user `helper` that may only run `/usr/bin/journalctl` as root, with a password.
4. Break the syntax on purpose in a drop-in with `visudo -f` and see how visudo protects you.
5. Find the sudo log line for your test.

---

## Common mistakes

- Editing `/etc/sudoers` with `vi` directly and saving a typo.
- Using relative command names (`systemctl` instead of `/usr/bin/systemctl`).
- Naming a drop-in `myrule.conf` (ignored because of the dot).
- Granting editors, pagers or interpreters through sudo (shell escape).
- Thinking `#includedir` is a comment and deleting it.

---

## Summary

| Need | Command / file |
|------|----------------|
| Full admin | add user to `sudo` (Debian) / `wheel` (RHEL) |
| Specific rights | `/etc/sudoers.d/<name>` via `visudo -f` |
| Rule format | `who where=(as_whom) [NOPASSWD:] /full/path args` |
| Check rights | `sudo -l` (`sudo -l -U user` as root) |
| Validate syntax | `visudo -c` |
| Root login shell | `sudo -i` / `su -` |
| Logs | `journalctl _COMM=sudo`, `/var/log/auth.log`, `/var/log/secure` |

## Check yourself

1. Whose password does `su -` ask for? And `sudo`?
2. What's wrong with this rule: `bob ALL=(ALL) systemctl restart nginx`?
3. Why is a file named `/etc/sudoers.d/web.conf` ignored?
4. Why is `alice ALL=(root) /usr/bin/less /var/log/*` dangerous?

<details>
<summary>Answers</summary>

1. `su -`: the target user's (root's). `sudo`: your own.
2. The command must be an absolute path: `/usr/bin/systemctl restart nginx`.
3. Files in `sudoers.d` whose names contain a `.` are skipped.
4. `less` can run shell commands (`!sh`), so Alice gets a root shell; the wildcard can also match other files.

</details>

## Further reading

- `man 5 sudoers`, `man 8 sudo`, `man 8 visudo`, `man 1 su`
- [ArchWiki: Sudo](https://wiki.archlinux.org/title/Sudo)
- [Sudo project – sudoers manual](https://www.sudo.ws/docs/man/sudoers.man/)
- [GTFOBins](https://gtfobins.github.io/) – which programs can be abused to escape from sudo restrictions

---

<!-- footer:start -->
[← Groups](02-groups.md) · [Module index](README.md) · [File Permissions and Ownership →](04-permissions-ownership.md)  
**Practice now:** [Exercises – Privilege Escalation with sudo and su](../../exercises/02-users-groups-permissions/03-sudo.md)
<!-- footer:end -->
