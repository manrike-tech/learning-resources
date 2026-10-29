# Access Control Lists (ACLs)

<!-- nav:start -->
> **Module 02 · Users, Groups, Permissions & LDAP** — lesson 5 of 8  
> [Module index](README.md) · [Course lesson (commands)](../../course/02-Users-Groups-Permissions.md#lesson-5---acls-gap) · [Exercises for this lesson](../../exercises/02-users-groups-permissions/05-acls.md)
<!-- nav:end -->

Standard permissions give a file exactly **one** owner and **one** group. Real life is messier: the `developers` group needs read-write, the `auditors` group needs read-only, and one contractor needs access to a single directory. **POSIX ACLs** solve this by adding extra entries for any number of named users and groups. In this lesson you will learn how ACLs work, how the **mask** limits them, and how **default ACLs** make new files inherit access automatically.

## What you will learn

- When standard permissions are not enough
- ACL entry types: owner, named user, owning group, named group, mask, other
- Reading ACLs with `getfacl` and setting them with `setfacl`
- The **mask** and *effective* permissions
- **Default ACLs** for inheritance
- Copying, backing up and removing ACLs

---

## Key terms

| Term | Definition |
|------|-----------|
| **ACL** | *Access Control List*: a list of permission entries attached to a file, extending the basic `ugo` model. |
| **Access ACL** | The ACL that controls access to the file or directory itself. |
| **Default ACL** | An ACL on a **directory** that is copied to files and subdirectories **created inside it**. |
| **Named user / named group entry** | An entry for a specific user (`user:bob:rwx`) or group (`group:audit:r-x`). |
| **Mask** | The maximum permission granted to named users, named groups and the owning group. |
| **Effective permission** | What an entry really grants after the mask is applied. |

---

## Why ACLs?

Scenario: `/srv/project` belongs to `root:developers` with mode `2770`. Now:

- the `audit` group needs **read-only** access,
- the contractor `eve` needs **read-write** access,
- everybody else must have **no** access.

With standard permissions there's no way to express this (only one group slot). With ACLs:

```console
$ sudo setfacl -m g:audit:rx /srv/project
$ sudo setfacl -m u:eve:rwx /srv/project
```

ACLs are supported by ext4, XFS, Btrfs and most modern filesystems out of the box. The tools come from the `acl` package (installed by default on most distributions).

---

## Reading ACLs

A `+` after the permission string in `ls -l` means "this file has an ACL":

```console
$ ls -ld /srv/project
drwxrws---+ 2 root developers 4096 Oct 21 12:00 /srv/project

$ getfacl /srv/project
# file: srv/project
# owner: root
# group: developers
# flags: -s-
user::rwx
user:eve:rwx
group::rwx
group:audit:r-x
mask::rwx
other::---
```

| Entry | Meaning |
|-------|---------|
| `user::rwx` | the **owner** (same as the `u` bits) |
| `user:eve:rwx` | **named user** eve |
| `group::rwx` | the **owning group** |
| `group:audit:r-x` | **named group** audit |
| `mask::rwx` | upper limit for named users, named groups and the owning group |
| `other::---` | everyone else (same as `o` bits) |

---

## How ACL permission checks work

The kernel picks the **first** matching category, in this order:

1. If you are the **owner** → `user::` entry.
2. If there is a **named user** entry for you → that entry **AND mask**.
3. If any of your groups matches the **owning group** or a **named group** that grants the access → that entry **AND mask**.
4. Otherwise → `other::`.

### The mask

The mask caps every entry in the "group class" (named users, owning group, named groups). It does **not** affect the owner or other.

```console
$ sudo setfacl -m u:eve:rwx,m::r-x /srv/project
$ getfacl /srv/project
user::rwx
user:eve:rwx            #effective:r-x
group::rwx              #effective:r-x
group:audit:r-x
mask::r-x
other::---
```

Eve's entry says `rwx` but the mask limits it to `r-x` – `getfacl` shows the result as `#effective`.

> **Important interaction with chmod:** on a file with an ACL, the **group bits** shown by `ls -l` are the **mask**, not the owning group's entry. So `chmod g-w file` lowers the mask and silently removes write access from *every* named user and group. And `setfacl -m` recalculates the mask to the union of the group-class entries unless you pass `-n`.

---

## Setting ACLs: setfacl

| Command | Effect |
|---------|--------|
| `setfacl -m u:bob:rw file` | add/modify a named user entry |
| `setfacl -m g:audit:r file` | add/modify a named group entry |
| `setfacl -m u:bob:rw,g:audit:r file` | several entries at once |
| `setfacl -m m::rx file` | set the mask |
| `setfacl -x u:bob file` | remove one entry |
| `setfacl -b file` | remove **all** ACL entries (back to plain permissions) |
| `setfacl -R -m g:audit:rX dir` | recursive; `X` = execute only for directories |
| `setfacl -m d:g:dev:rwx dir` | set a **default** entry |
| `setfacl -k dir` | remove the default ACL |

Permissions can be written as `rwx`, `rw-`, `r` or as octal digits (`u:bob:6`).

---

## Default ACLs: inheritance

An access ACL on a directory affects only the directory itself. Files created **later** inside it get their ACL from the directory's **default ACL**.

```console
$ sudo setfacl -m g:audit:rx /srv/project          # access to the directory now
$ sudo setfacl -m d:g:audit:rx /srv/project        # inherited by new files/dirs
$ getfacl /srv/project | grep default
default:user::rwx
default:group::rwx
default:group:audit:r-x
default:mask::rwx
default:other::---

alice$ touch /srv/project/new.txt
$ getfacl /srv/project/new.txt
user::rw-
group::rwx              #effective:rw-
group:audit:r-x         #effective:r--
mask::rw-
other::---
```

Note the inherited entries on a **file** lose `x` because files are created with mode `666` (the program's requested mode limits the mask).

### Existing files are not changed by a default ACL

Default ACLs apply only to **new** files. For existing content, also run a recursive access ACL:

```console
$ sudo setfacl -R -m g:audit:rX /srv/project          # existing files and dirs
$ sudo setfacl -R -m d:g:audit:rX /srv/project        # future files (on all subdirs)
```

This "two commands" pattern is exactly what exam tasks like *"give group X read access to directory Y, including files created in the future"* expect.

---

## Copying and backing up ACLs

| Tool | Preserves ACLs? |
|------|-----------------|
| `cp` | no – use `cp -a` or `cp --preserve=all` |
| `mv` (same filesystem) | yes |
| `tar` | only with `--acls` |
| `rsync` | only with `-A` (`rsync -aAX`) |

```console
$ getfacl -R /srv/project > /root/project.acl     # save all ACLs
$ setfacl --restore=/root/project.acl             # restore them (run from /)
$ getfacl file1 | setfacl --set-file=- file2      # copy ACL from one file to another
```

---

## Try it yourself

1. Create `/srv/acltest` owned by `root:root` mode `770`. Give user `bob` read-write via ACL.
2. Look at `ls -l`: what changed? Look at `getfacl`.
3. Run `chmod g-w /srv/acltest` and look at `getfacl` again. What happened to bob?
4. Add a default ACL for group `audit` (read-only) and create a file inside. Check it.
5. Remove all ACLs with `setfacl -b`.

---

## Common mistakes

- Setting only an access ACL and expecting new files to inherit it (needs `d:`).
- Setting only a default ACL and forgetting existing files.
- Using `rwx` with `-R` (makes every file executable) – use `rX`.
- Running `chmod` on an ACL'd file and unintentionally changing the mask.
- Copying files with `cp` or `tar` without ACL options during backups.

---

## Summary

| Need | Command |
|------|---------|
| Show | `getfacl file` (look for `+` in `ls -l`) |
| Add user / group | `setfacl -m u:bob:rw file`, `setfacl -m g:audit:r file` |
| Inherit for new files | `setfacl -m d:g:audit:rX dir` |
| Apply to existing tree | `setfacl -R -m g:audit:rX dir` |
| Limit everything | `setfacl -m m::r file` |
| Remove one / all | `setfacl -x u:bob file` / `setfacl -b file` |
| Backup / restore | `getfacl -R dir > f` / `setfacl --restore=f` |

## Check yourself

1. Which entries does the mask limit?
2. `user:bob:rwx` and `mask::r--`. What can bob do?
3. What is the difference between `setfacl -m g:x:rx dir` and `setfacl -m d:g:x:rx dir`?
4. Why does `ls -l` show `rwx` for the group of a file whose owning group only has `r--`?

<details>
<summary>Answers</summary>

1. Named users, the owning group and named groups (not the owner, not other).
2. Read only.
3. The first sets access on the directory itself; the second sets the default ACL inherited by new files and subdirectories.
4. With an ACL present, the group bits shown by `ls -l` are the **mask**, not the owning group entry.

</details>

## Further reading

- `man 5 acl`, `man 1 setfacl`, `man 1 getfacl`
- [ArchWiki: Access Control Lists](https://wiki.archlinux.org/title/Access_Control_Lists)
- [Red Hat: Configuring basic system settings (RHEL 9) – see “Managing file system permissions”](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_basic_system_settings/index)

---

<!-- footer:start -->
[← File Permissions and Ownership](04-permissions-ownership.md) · [Module index](README.md) · [Personal and System-Wide Environment Profiles →](06-environment-profiles.md)  
**Practice now:** [Exercises – Access Control Lists (ACLs)](../../exercises/02-users-groups-permissions/05-acls.md)
<!-- footer:end -->
