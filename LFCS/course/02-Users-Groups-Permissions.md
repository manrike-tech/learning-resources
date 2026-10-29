# Module 02 - Users, Groups, Permissions & LDAP

**LFS207:** Ch. 4 User Account Management · Ch. 5 Group Management · Ch. 6 File Permissions and Ownership · Ch. 29 LDAP
**Exam domain:** Users & Groups (10%)  
**Theory:** [Module 02 theory notes](../theory/02-users-groups-permissions/README.md) · **Exercises:** [Module 02 exercises](../exercises/02-users-groups-permissions/README.md)  
**Gaps covered:** resource limits, ACLs, environment profiles

---

## Learning Objectives

- Create, modify, lock and delete local users and groups.
- Configure password aging and sudo.
- Apply permissions, special bits, umask and ACLs.
- Set per-user resource limits.
- Join a system to an LDAP directory for users and groups.

---

## Lesson 1 - User Accounts (Ch. 4)

> **Theory:** [User Accounts](../theory/02-users-groups-permissions/01-user-accounts.md) · **Exercises:** [practice tasks](../exercises/02-users-groups-permissions/01-user-accounts.md)

Key files: `/etc/passwd` (accounts) · `/etc/shadow` (hashes, aging) · `/etc/login.defs` (defaults) · `/etc/skel/` (home template) · `/etc/default/useradd`.

```bash
useradd -m -s /bin/bash -c "Alice Smith" alice    # create with home & shell
useradd -m -u 1500 -G wheel,dev -e 2026-12-31 bob  # UID, groups, expiry
useradd -r -s /usr/sbin/nologin svc_app            # system account
passwd alice
usermod -aG docker alice        # ADD to group (-a is critical!)
usermod -s /bin/zsh alice
usermod -L alice / usermod -U alice   # lock / unlock
usermod -d /home/new -m alice         # move home
userdel -r alice                      # delete + home
id alice; getent passwd alice

chage -l alice                 # show aging
chage -M 90 -m 1 -W 7 alice    # max 90 days, min 1, warn 7
chage -d 0 alice               # force password change at next login
chage -E 2026-12-31 alice      # account expiry
```

> **Exam trap:** `usermod -G dev alice` **replaces** all supplementary groups. Always use `-aG` to append.

---

## Lesson 2 - Groups (Ch. 5)

> **Theory:** [Groups](../theory/02-users-groups-permissions/02-groups.md) · **Exercises:** [practice tasks](../exercises/02-users-groups-permissions/02-groups.md)

```bash
groupadd dev
groupadd -g 3000 ops
groupmod -n developers dev
groupdel ops
gpasswd -a alice developers     # add member
gpasswd -d alice developers     # remove member
groups alice; getent group developers
newgrp developers               # switch primary group in current shell
```

---

## Lesson 3 - sudo

> **Theory:** [Privilege Escalation with sudo and su](../theory/02-users-groups-permissions/03-sudo.md) · **Exercises:** [practice tasks](../exercises/02-users-groups-permissions/03-sudo.md)

```bash
visudo                                   # always edit with visudo
visudo -f /etc/sudoers.d/ops
```
```
alice   ALL=(ALL)       ALL
%ops    ALL=(ALL)       NOPASSWD: /usr/bin/systemctl restart nginx
```
Admin group: `sudo` on Debian/Ubuntu, `wheel` on RHEL.

---

## Lesson 4 - Permissions & Ownership (Ch. 6)

> **Theory:** [File Permissions and Ownership](../theory/02-users-groups-permissions/04-permissions-ownership.md) · **Exercises:** [practice tasks](../exercises/02-users-groups-permissions/04-permissions-ownership.md)

```bash
chmod 750 dir; chmod u+x,g-w,o= file
chown alice:dev file; chown -R alice /srv/app; chgrp dev file
umask            # 022 → new files 644, dirs 755
umask 027        # new files 640, dirs 750
```

| Special bit | Octal | On file | On directory |
|-------------|-------|---------|--------------|
| SUID | 4000 | Runs as file owner | — |
| SGID | 2000 | Runs as group | New files inherit the directory's group |
| Sticky | 1000 | — | Only owner can delete own files (`/tmp`) |

```bash
chmod 2770 /srv/shared        # shared team directory
chmod +t /srv/drop
lsattr file; chattr +i file   # immutable attribute (even root can't change)
```

---

## Lesson 5 - ACLs (Gap)

> **Theory:** [Access Control Lists (ACLs)](../theory/02-users-groups-permissions/05-acls.md) · **Exercises:** [practice tasks](../exercises/02-users-groups-permissions/05-acls.md)

Fine-grained permissions beyond owner/group/other.

```bash
getfacl /srv/project
setfacl -m u:bob:rwx /srv/project          # user entry
setfacl -m g:audit:rx /srv/project         # group entry
setfacl -m d:g:dev:rwx /srv/project        # DEFAULT ACL (inherited by new files)
setfacl -R -m u:bob:rX /srv/project        # recursive; X = exec only on dirs
setfacl -x u:bob /srv/project              # remove entry
setfacl -b /srv/project                    # remove all ACLs
```

- A `+` at the end of `ls -l` permissions means an ACL exists.
- The **mask** limits the maximum effective permission of named users/groups.

---

## Lesson 6 - Environment Profiles (Gap)

> **Theory:** [Personal and System-Wide Environment Profiles](../theory/02-users-groups-permissions/06-environment-profiles.md) · **Exercises:** [practice tasks](../exercises/02-users-groups-permissions/06-environment-profiles.md)

| Need | Where |
|------|-------|
| Default files for **new** users | `/etc/skel/` |
| Variables/aliases for all users | `/etc/profile.d/custom.sh` |
| One user | `~/.bashrc`, `~/.bash_profile` / `~/.profile` |

---

## Lesson 7 - User Resource Limits (Gap)

> **Theory:** [User Resource Limits](../theory/02-users-groups-permissions/07-resource-limits.md) · **Exercises:** [practice tasks](../exercises/02-users-groups-permissions/07-resource-limits.md)

```bash
ulimit -a              # current limits
ulimit -n 4096         # soft open files (current shell only)
```

Persistent: `/etc/security/limits.conf` or a file in `/etc/security/limits.d/`:

```
# <domain>  <type>  <item>    <value>
alice       hard    nproc     50
@dev        soft    nofile    4096
@dev        hard    nofile    8192
*           hard    maxlogins 3
```

Applied by `pam_limits` at login. For **systemd services**, use `LimitNOFILE=` in the unit file instead.

---

## Lesson 8 - LDAP Clients (Ch. 29)

> **Theory:** [Using LDAP for Users and Groups](../theory/02-users-groups-permissions/08-ldap-clients.md) · **Exercises:** [practice tasks](../exercises/02-users-groups-permissions/08-ldap-clients.md)

Concepts: directory tree, **DN** (distinguished name), **base DN** (`dc=example,dc=com`), `ou=People`, `ou=Groups`, objectClasses `posixAccount` / `posixGroup`.

Typical client with **SSSD**:

```bash
# Debian/Ubuntu
apt install sssd-ldap ldap-utils libnss-sss libpam-sss
# RHEL family
dnf install sssd sssd-ldap openldap-clients oddjob-mkhomedir
```

`/etc/sssd/sssd.conf` (mode **600**, owner root):

```ini
[sssd]
services = nss, pam
domains = example

[domain/example]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://ldap.example.com
ldap_search_base = dc=example,dc=com
ldap_id_use_start_tls = true
ldap_tls_reqcert = demand
cache_credentials = true
```

```bash
chmod 600 /etc/sssd/sssd.conf
systemctl enable --now sssd
# RHEL: authselect select sssd with-mkhomedir --force
# Ubuntu: pam-auth-update --enable mkhomedir
getent passwd ldapuser          # must resolve via LDAP
ldapsearch -x -H ldap://ldap.example.com -b dc=example,dc=com "(uid=ldapuser)"
```

> **Practice:** run a test OpenLDAP server in a container (e.g. an `osixia/openldap` or `bitnami/openldap` image) on `rhel1`, and point `deb1` at it.

---

## Lab 02

1. Create group `devops` (GID 5000) and users `ana`, `luis` (members of `devops`), shell `/bin/bash`.
2. Force `luis` to change password at next login; set max password age 60 days.
3. Create `/srv/devops` owned by `root:devops`, mode `2770`.
4. Give user `audit` read-only access to `/srv/devops` via ACL, including files created in the future.
5. Allow `devops` members to restart `sshd` with sudo, without password, and nothing else.
6. Limit `ana` to 30 processes (hard limit).
7. Make every new user get a `~/README.txt` automatically.

## Self-check

- [ ] I never forget `-a` with `usermod -G`.
- [ ] I can explain SGID on a directory and the sticky bit.
- [ ] I can set a default ACL and explain the mask.
- [ ] I know the difference between `limits.conf` and `LimitNOFILE=`.
- [ ] I can configure SSSD and verify with `getent`.
