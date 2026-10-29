# Module 01 - Foundations & Essential Commands

**LFS207:** Ch. 1 Introduction · Ch. 2 Linux Filesystem Tree Layout · Ch. 3 User Environment
**Exam domain:** Essential Commands (20%)  
**Theory:** [Module 01 theory notes](../theory/01-foundations/README.md) · **Exercises:** [Module 01 exercises](../exercises/01-foundations/README.md)  

---

## Learning Objectives

- Explain the purpose of each top-level directory in the FHS.
- Find files by name, size, owner, time and permissions.
- Search and transform text with `grep`, `sed`, `awk`, `cut`, `sort`, `uniq`.
- Use redirection, pipes and archives confidently.
- Find documentation offline.

---

## Lesson 1 - The Filesystem Hierarchy Standard

> **Theory:** [The Linux Filesystem Hierarchy](../theory/01-foundations/01-filesystem-hierarchy.md) · **Exercises:** [practice tasks](../exercises/01-foundations/01-filesystem-hierarchy.md)

| Directory | Contents |
|-----------|----------|
| `/` | Root of the single tree |
| `/bin`, `/sbin` | Essential binaries (on modern distros, symlinks to `/usr/bin`, `/usr/sbin`) |
| `/boot` | Kernel, initramfs, GRUB |
| `/dev` | Device files (created by udev) |
| `/etc` | System-wide **configuration** (text files) |
| `/home` | User home directories |
| `/lib`, `/lib64` | Shared libraries and kernel modules |
| `/media`, `/mnt` | Mount points for removable / temporary media |
| `/opt` | Add-on, self-contained software |
| `/proc` | **Virtual**: process and kernel info (pseudo-filesystem) |
| `/sys` | **Virtual**: devices, drivers, kernel objects (sysfs) |
| `/root` | Root user's home |
| `/run` | Runtime data since boot (tmpfs) |
| `/srv` | Data served by the system (web, FTP) |
| `/tmp` | Temporary files (sticky bit) |
| `/usr` | Read-only user programs, libraries, docs |
| `/var` | **Variable** data: logs (`/var/log`), spool, cache, `/var/lib` |

```bash
man hier            # FHS description on your system
df -hT              # which filesystem each mount point uses
findmnt             # tree of mounts
```

> **Exam tip:** `/etc` = config, `/var/log` = logs, `/proc` and `/sys` = live kernel state. Most tasks start in one of these.

---

## Lesson 2 - Finding Files

> **Theory:** [Finding Files](../theory/01-foundations/02-finding-files.md) · **Exercises:** [practice tasks](../exercises/01-foundations/02-finding-files.md)

```bash
find /etc -name "*.conf"                  # by name
find / -type f -size +100M 2>/dev/null    # files > 100 MB
find /home -user alice                    # by owner
find /var -mtime -1                       # modified in last 24 h
find / -perm -4000 -type f 2>/dev/null    # SUID files
find /tmp -name "*.log" -delete           # find and delete
find /data -type f -exec chmod 640 {} \;  # run a command on each
find /srv -empty                          # empty files/dirs

locate sshd_config      # fast, uses a DB (updatedb to refresh)
which ssh; type ssh     # where a command comes from
```

> **Exam tip:** many tasks say "find all files that… and save the list to /path/file". Redirect with `> /path/file` and check the file afterwards.

---

## Lesson 3 - Working with Text

> **Theory:** [Working with Text](../theory/01-foundations/03-working-with-text.md) · **Exercises:** [practice tasks](../exercises/01-foundations/03-working-with-text.md)

```bash
grep -r "PermitRootLogin" /etc/ssh/       # recursive
grep -i error /var/log/syslog             # case-insensitive
grep -v '^#' file | grep -v '^$'          # strip comments & blank lines
grep -E 'fail(ed|ure)' file               # extended regex
grep -c pattern file                      # count matches

sed 's/old/new/g' file                    # print substituted
sed -i 's/^#Port 22/Port 2222/' file      # edit in place
sed -n '10,20p' file                      # print lines 10-20

awk -F: '{print $1, $7}' /etc/passwd      # fields with custom separator
awk -F: '$3 >= 1000 {print $1}' /etc/passwd

cut -d: -f1 /etc/passwd
sort -k3 -n -t: /etc/passwd               # numeric sort by field 3
sort file | uniq -c | sort -rn            # frequency count
wc -l file; head -n 5; tail -f /var/log/syslog
diff file1 file2; cmp a b
tr 'a-z' 'A-Z' < file
```

---

## Lesson 4 - Redirection, Pipes, Archives

> **Theory:** [Redirection, Pipes and Archives](../theory/01-foundations/04-redirection-pipes-archives.md) · **Exercises:** [practice tasks](../exercises/01-foundations/04-redirection-pipes-archives.md)

```bash
cmd > out.txt          # stdout overwrite
cmd >> out.txt         # append
cmd 2> err.txt         # stderr
cmd > all.txt 2>&1     # both
cmd | tee file         # screen + file

tar -czvf backup.tar.gz /etc        # create gzip archive
tar -cjvf backup.tar.bz2 /etc       # bzip2
tar -cJvf backup.tar.xz /etc        # xz
tar -tvf backup.tar.gz              # list
tar -xzvf backup.tar.gz -C /restore # extract to dir
gzip / gunzip; bzip2; xz; zip -r / unzip
```

---

## Lesson 5 - The User Environment (Ch. 3)

> **Theory:** [The User Environment](../theory/01-foundations/05-user-environment.md) · **Exercises:** [practice tasks](../exercises/01-foundations/05-user-environment.md)

```bash
echo $PATH; env; set
export VAR=value                     # environment variable for child processes
alias ll='ls -alF'
history; !42; !!; Ctrl+R             # history tricks
```

Startup files (login shell, bash):

| File | Scope |
|------|-------|
| `/etc/profile`, `/etc/profile.d/*.sh` | All users, login shells |
| `/etc/bash.bashrc` (Debian) / `/etc/bashrc` (RHEL) | All users, interactive shells |
| `~/.bash_profile` or `~/.profile` | One user, login shells |
| `~/.bashrc` | One user, interactive shells |

> Persist a variable for **all users**: create `/etc/profile.d/myvars.sh`. For **one user**: append to `~/.bashrc`.

---

## Lesson 6 - Finding Help Offline

> **Theory:** [Finding Help Offline](../theory/01-foundations/06-finding-help.md) · **Exercises:** [practice tasks](../exercises/01-foundations/06-finding-help.md)

```bash
man -k password       # search man pages (apropos)
man 5 passwd          # section 5 = file formats
cmd --help
ls /usr/share/doc/
```

Man sections to remember: **1** commands · **5** config files · **8** admin commands.

---

## Lab 01

1. List every file under `/etc` ending in `.conf` and modified in the last 7 days; save the list to `/root/recent-conf.txt`.
2. Find all SUID binaries and save their paths to `/root/suid.txt`.
3. Print usernames with UID ≥ 1000 and their shells, sorted alphabetically, to `/root/users.txt`.
4. Create `/root/etc-backup.tar.xz` containing `/etc`, then list its contents.
5. Make the variable `COMPANY=acme` available to all users at login.
6. Copy `/etc/ssh/sshd_config` to `/root/sshd.copy` and, with one `sed` command, remove all comment and blank lines in place.

## Self-check

- [ ] I know where config, logs, and kernel info live.
- [ ] I can write a `find` with `-exec` from memory.
- [ ] I can redirect stdout and stderr separately.
- [ ] I can create and extract tar archives with any compression.
- [ ] I know which file to edit for a system-wide vs per-user variable.
