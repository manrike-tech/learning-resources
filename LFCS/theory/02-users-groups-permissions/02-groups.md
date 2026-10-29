# Groups

<!-- nav:start -->
> **Module 02 · Users, Groups, Permissions & LDAP** — lesson 2 of 8  
> [Module index](README.md) · [Course lesson (commands)](../../course/02-Users-Groups-Permissions.md#lesson-2---groups-ch-5) · [Exercises for this lesson](../../exercises/02-users-groups-permissions/02-groups.md)
<!-- nav:end -->

Groups let you grant the same access to many users at once: instead of giving five developers access to a directory one by one, you give access to the `developers` group and add the five users to it. In this lesson you will learn how groups work, the difference between primary and supplementary groups, and how to manage them.

## What you will learn

- Primary vs supplementary groups and when each is used
- User Private Groups (UPG)
- The `/etc/group` and `/etc/gshadow` files
- Creating, renaming and deleting groups
- Adding and removing members with `usermod`, `gpasswd`
- Why group changes need a new login, and `newgrp`

---

## Key terms

| Term | Definition |
|------|-----------|
| **Group** | A named set of users, identified by a numeric **GID**. |
| **Primary group** | The group stored in field 4 of `/etc/passwd`. New files a user creates belong to this group. Every user has exactly one. |
| **Supplementary group** | Any additional group a user belongs to, listed in `/etc/group`. Grants extra access. |
| **User Private Group (UPG)** | The convention of creating a group with the same name as each user, used as their primary group. |
| **Group administrator** | A user allowed to manage a group's membership with `gpasswd` (stored in `/etc/gshadow`). |

---

## Primary and supplementary groups

```console
$ id alice
uid=1001(alice) gid=1001(alice) groups=1001(alice),27(sudo),1005(developers)
                ─────┬─────────        ─────────────┬────────────────────────
              primary group               all groups (primary + supplementary)
```

When Alice creates a file, its group is her **primary** group:

```console
$ touch report.txt; ls -l report.txt
-rw-rw-r-- 1 alice alice 0 Oct 21 10:00 report.txt
```

When the kernel checks whether Alice may access a file owned by group `developers`, it considers **all** her groups. So supplementary groups are about *access*, the primary group is about *ownership of new files*.

### User Private Groups

Most distributions create a private group for each user (`alice` → group `alice`). This keeps new files private by default, while still allowing a group-writable `umask` (`002`). Combined with SGID directories (see [Permissions](04-permissions-ownership.md)) it makes team folders work smoothly.

---

## `/etc/group` and `/etc/gshadow`

```console
$ grep developers /etc/group
developers:x:1005:alice,bob
```

| # | Field | Meaning |
|---|-------|---------|
| 1 | name | group name |
| 2 | password | `x` → see `/etc/gshadow` (group passwords are rarely used) |
| 3 | GID | group ID |
| 4 | members | comma-separated **supplementary** members |

> Users whose **primary** group is `developers` are not listed in field 4 – their membership comes from `/etc/passwd`. Use `getent group developers` *and* check primary groups when auditing.

`/etc/gshadow` (root-only) holds group passwords and **group administrators**:

```console
$ sudo grep developers /etc/gshadow
developers:!::alice,bob
#          │ │  └─ members
#          │ └─ administrators (none)
#          └─ no password
```

---

## Managing groups

```console
$ sudo groupadd developers               # next free GID
$ sudo groupadd -g 3000 ops              # specific GID
$ sudo groupadd -r monitoring            # system group (low GID)
$ sudo groupmod -n devs developers       # rename
$ sudo groupmod -g 3100 ops              # change GID (files keep the OLD number!)
$ sudo groupdel ops                      # delete (fails if it's someone's primary group)
$ getent group devs
devs:x:1005:alice,bob
```

---

## Managing membership

| Task | Command |
|------|---------|
| Add user to a group | `usermod -aG devs carol` or `gpasswd -a carol devs` |
| Add to several groups | `usermod -aG devs,docker carol` |
| Remove user from a group | `gpasswd -d carol devs` |
| Set the full member list | `gpasswd -M alice,bob,carol devs` |
| Change primary group | `usermod -g devs carol` |
| Make a user group admin | `gpasswd -A alice devs` (alice can now run `gpasswd -a/-d` for devs) |
| List a user's groups | `groups carol`, `id carol` |
| List a group's members | `getent group devs` |

```console
$ sudo gpasswd -a carol devs
Adding user carol to group devs
$ groups carol
carol : carol devs
```

> **Exam trap:** `usermod -G devs carol` (without `-a`) removes Carol from every other supplementary group. Prefer `gpasswd -a` or `usermod -aG`.

---

## Why changes need a new login

Group membership is read when a session starts and copied into every process of that session. A running shell does not notice that `/etc/group` changed.

```console
$ sudo usermod -aG docker alice     # (Alice is logged in)
alice$ id -nG
alice sudo developers                ← no docker yet
alice$ newgrp docker                 # starts a NEW shell with docker as primary group
alice$ id -nG
docker alice sudo developers
```

Proper fix: log out and log back in (or start a new SSH session). Services must be **restarted** to pick up new group memberships of their user.

---

## Example: a team directory

```console
$ sudo groupadd projectx
$ sudo usermod -aG projectx alice; sudo usermod -aG projectx bob
$ sudo mkdir /srv/projectx
$ sudo chgrp projectx /srv/projectx
$ sudo chmod 2770 /srv/projectx          # rwx for owner and group, SGID bit
$ ls -ld /srv/projectx
drwxrws--- 2 root projectx 4096 Oct 21 10:30 /srv/projectx
```

Alice and Bob can now create files there; thanks to the SGID bit (`s`) new files belong to `projectx`, so both can work on them. Details in [Permissions & Ownership](04-permissions-ownership.md).

---

## Try it yourself

1. Create group `qa` with GID 4100.
2. Add two existing users to it with two different commands.
3. Compare `getent group qa` and `id user` for each.
4. Remove one user with `gpasswd -d`.
5. Rename the group to `quality` and confirm the GID didn't change.

---

## Common mistakes

- Forgetting `-a` in `usermod -aG`.
- Expecting group changes to apply to an existing session.
- Deleting a group that is still the primary group of a user (it fails) – change the user's primary group first.
- Changing a GID with `groupmod -g` and forgetting that existing files keep the old number (`find / -gid OLD -exec chgrp NEW {} +`).

---

## Summary

- Primary group (in `/etc/passwd`) = group of new files; supplementary groups (in `/etc/group`) = extra access.
- `groupadd`, `groupmod`, `groupdel` manage groups; `usermod -aG`, `gpasswd -a/-d/-M` manage members.
- New memberships take effect at the next login (`newgrp` for a quick test).

## Check yourself

1. Where is a user's primary group stored?
2. Why might `getent group devs` not list a user who is in the group?
3. What does `gpasswd -A alice devs` allow Alice to do?
4. Alice was just added to `docker` but still gets "permission denied". Why?

<details>
<summary>Answers</summary>

1. Field 4 of `/etc/passwd` (as a GID).
2. Because `devs` is that user's **primary** group – primary membership is not listed in `/etc/group`.
3. Add and remove members of `devs` without being root.
4. Her current session was started before the change; she must log in again (or use `newgrp docker`).

</details>

## Further reading

- `man 5 group`, `man 5 gshadow`, `man 8 groupadd`, `man 1 gpasswd`, `man 1 newgrp`
- [Ubuntu Server: User management](https://documentation.ubuntu.com/server/how-to/security/user-management/)
- [ArchWiki: Users and groups – Group management](https://wiki.archlinux.org/title/Users_and_groups#Group_management)

---

<!-- footer:start -->
[← User Accounts](01-user-accounts.md) · [Module index](README.md) · [Privilege Escalation with sudo and su →](03-sudo.md)  
**Practice now:** [Exercises – Groups](../../exercises/02-users-groups-permissions/02-groups.md)
<!-- footer:end -->
