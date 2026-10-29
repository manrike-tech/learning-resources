# Finding Files

<!-- nav:start -->
> **Module 01 · Foundations & Essential Commands** — lesson 2 of 6  
> [Module index](README.md) · [Course lesson (commands)](../../course/01-Foundations.md#lesson-2---finding-files) · [Exercises for this lesson](../../exercises/01-foundations/02-finding-files.md)
<!-- nav:end -->

On a real server – and in the LFCS exam – many tasks start with "find all files that…". In this lesson you will learn to search the filesystem precisely with `find`, quickly with `locate`, and to identify where a command comes from with `which`, `type` and `whereis`.

## What you will learn

- The structure of a `find` command: **starting points**, **tests**, **operators** and **actions**
- How to search by name, type, size, owner, time and permissions
- How to act on the results (`-delete`, `-exec`, `xargs`) safely
- When to use `locate` instead of `find`
- How the shell decides which program a command name refers to

---

## Key terms

| Term | Definition |
|------|-----------|
| **Starting point** | Directory where `find` begins walking the tree recursively (`/etc`, `.`). |
| **Test** | A condition evaluated on each file (`-name`, `-size`, `-user`…). True or false. |
| **Action** | What to do with matching files (`-print` (default), `-delete`, `-exec`, `-ls`). |
| **Operator** | Combines tests: AND (implicit), `-o` (OR), `!` or `-not` (NOT), `\( \)` grouping. |
| **Glob** | Shell-style wildcard pattern (`*.conf`, `file?.txt`) used by `-name`. |
| **`$PATH`** | Colon-separated list of directories the shell searches for commands. |

---

## Anatomy of `find`

```text
find  /var/log   -type f -name "*.log" -size +10M   -exec ls -lh {} \;
      ───┬────   ──────────────┬────────────────    ─────────┬────────
   starting point        tests (ANDed)                     action
```

`find` walks every file below the starting point, evaluates the tests left to right and runs the action on each file for which they are all true.

> **Always quote patterns.** `find /etc -name *.conf` lets the *shell* expand `*.conf` first (using files in your current directory!). Write `-name '*.conf'` or `-name "*.conf"`.

---

## Searching by name and type

```console
$ find /etc -name "*.conf" | head -3
/etc/sysctl.conf
/etc/host.conf
/etc/ld.so.conf

$ find / -iname "readme*" 2>/dev/null       # -iname = case-insensitive
$ find /etc -type d -name "ssh*"            # directories only
/etc/ssh
$ find /dev -type b                         # block devices
$ find /usr/bin -type l | head -3           # symbolic links
```

| `-type` | Meaning |
|---------|---------|
| `f` | regular file |
| `d` | directory |
| `l` | symbolic link |
| `b` / `c` | block / character device |
| `s` / `p` | socket / named pipe |

`2>/dev/null` hides "Permission denied" messages (stderr) when searching as a normal user – see [Redirection](04-redirection-pipes-archives.md).

Limit depth with `-maxdepth N` (place it before the tests):

```console
$ find /etc -maxdepth 1 -name "*.conf" | wc -l
27
```

---

## Searching by size

`-size` uses units: `c` bytes, `k` KiB, `M` MiB, `G` GiB. A `+` means *more than*, `-` means *less than*, no sign means *exactly* (rounded up to the unit).

```console
$ find /var -type f -size +100M 2>/dev/null
/var/lib/libvirt/images/vm1.qcow2

$ find /home -type f -size -1k             # smaller than 1 KiB
$ find /srv -empty                         # empty files AND empty directories
```

> **Gotcha:** `-size -1M` means "0 MiB after rounding up", i.e. only empty files. Use a smaller unit (`-size -1024k`) for "less than 1 MiB".

---

## Searching by owner and group

```console
$ find /home -user alice
$ find / -group developers -type f 2>/dev/null
$ find / -nouser 2>/dev/null       # files whose UID has no account (e.g. after userdel)
```

`-nouser` / `-nogroup` are handy after deleting users: the files remain, owned by a bare number.

---

## Searching by time

| Test | Unit | Based on |
|------|------|----------|
| `-mtime N` | days | content modified |
| `-mmin N` | minutes | content modified |
| `-atime`, `-amin` | days / minutes | last access |
| `-ctime`, `-cmin` | days / minutes | inode change (perms, owner) |
| `-newer FILE` | – | modified more recently than FILE |

`-N` means "less than N ago", `+N` "more than N ago", `N` "exactly N (in whole units)".

```console
$ find /etc -mtime -1              # changed in the last 24 h
$ find /var/log -mtime +30         # older than 30 days
$ find /etc -mmin -60 -type f      # changed in the last hour – great for "what did I just break?"
$ touch -d "2 hours ago" /tmp/ref; find /etc -newer /tmp/ref
```

---

## Searching by permissions

| Form | Meaning | Example |
|------|---------|---------|
| `-perm 644` | permissions **exactly** 644 | |
| `-perm -4000` | **all** of these bits are set | SUID files |
| `-perm /022` | **any** of these bits is set | group- or world-writable |

```console
$ find / -perm -4000 -type f 2>/dev/null       # SUID binaries (security audit)
/usr/bin/passwd
/usr/bin/sudo
/usr/bin/su
...
$ find /srv -perm /o+w -type f                 # world-writable files
$ find /home -type f ! -perm -u+r              # files the owner can't read
```

---

## Combining tests

Tests are ANDed by default. Use `-o` for OR and `!` for NOT; group with escaped parentheses.

```console
# .log OR .gz files, larger than 1 MiB
$ find /var/log \( -name "*.log" -o -name "*.gz" \) -size +1M

# everything under /etc NOT owned by root
$ find /etc ! -user root
```

---

## Acting on results

### Built-in actions

```console
$ find /tmp -name "*.tmp" -delete            # delete matches (test without -delete first!)
$ find /etc -name "*.conf" -ls               # ls -dils style output
$ find /data -type f -printf "%s %p\n"       # custom format: size and path
```

### `-exec`

`{}` is replaced by the file name; the command ends with `\;` (one command per file) or `+` (as many files per command as possible – faster).

```console
$ find /srv/app -type f -exec chmod 640 {} \;
$ find /srv/app -type d -exec chmod 750 {} +
$ find /var/log -name "*.log" -mtime +7 -exec gzip {} +
$ find /etc -name "*.conf" -exec grep -l "Listen" {} +
```

`-ok` works like `-exec` but asks for confirmation for each file.

### `xargs`

`xargs` turns lines from stdin into arguments. Use `-print0` with `xargs -0` so file names with spaces are handled correctly:

```console
$ find /home -name "*.mp3" -print0 | xargs -0 du -ch | tail -1
```

> **Safety habit:** run the `find` command **without** `-delete`/`-exec` first and read the list. Then add the action.

---

## Saving results (typical exam task)

```console
$ find / -user alice -type f 2>/dev/null > /root/alice-files.txt
$ find /usr/share -type f -size +5M 2>/dev/null | sort > /opt/big.txt
$ wc -l /root/alice-files.txt
42 /root/alice-files.txt
```

Always `cat` or `wc -l` the output file to confirm it contains what was asked (full paths, sorted, etc.).

---

## `locate`: fast search from a database

`locate` searches a prebuilt index instead of walking the disk, so it answers instantly – but only knows files that existed when the index was last built (usually nightly).

```console
$ sudo apt install plocate      # or: dnf install mlocate / plocate
$ sudo updatedb                 # rebuild the index now
$ locate sshd_config
/etc/ssh/sshd_config
/usr/share/man/man5/sshd_config.5.gz
$ locate -i "*.CONF" | head     # case-insensitive
$ locate -c ".conf"             # count matches
```

| | `find` | `locate` |
|--|--------|----------|
| Speed | slow on large trees | instant |
| Up to date | always | only since last `updatedb` |
| Criteria | name, size, time, owner, perms… | name/path only |
| Can act on files | yes | no |

---

## Where does a command come from?

When you type `ls`, the shell looks for it in this order: **alias → keyword → function → builtin → `$PATH` directories (left to right)**.

```console
$ type ls
ls is aliased to `ls --color=auto'
$ type cd
cd is a shell builtin
$ type -a python3
python3 is /usr/bin/python3
$ which sshd
/usr/sbin/sshd
$ whereis passwd
passwd: /usr/bin/passwd /etc/passwd /usr/share/man/man1/passwd.1.gz /usr/share/man/man5/passwd.5.gz
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

- `which` – first match in `$PATH` (doesn't know aliases/builtins).
- `type` – what the shell will *actually* run. Most accurate.
- `whereis` – binary, source and man pages.

To find which **package** owns a command, see [dpkg & APT](../03-packages-git/02-dpkg-apt.md) and [RPM & dnf](../03-packages-git/03-rpm-dnf.md).

---

## Try it yourself

1. List all directories directly under `/etc` (depth 1) whose name starts with `sys`.
2. Find files in `/var/log` modified in the last 30 minutes.
3. Find all SUID **and** SGID files (hint: `-perm /6000`).
4. Count how many `.conf` files exist under `/etc`.
5. Compare the time of `find / -name passwd 2>/dev/null` with `locate passwd` (`time` prefix).

---

## Common mistakes

- Unquoted `-name *.txt`.
- Putting `-maxdepth` after other tests (find warns about it).
- Using `-size -1M` when you mean "under one megabyte".
- `find … -delete` before verifying the list. `-delete` placed **before** the tests deletes everything under the starting point!
- Forgetting `2>/dev/null` and saving error messages into the result file (they go to stderr, so `>` alone is fine – but `&>` would include them).

---

## Summary

| Need | Command |
|------|---------|
| By name | `find DIR -name 'pattern'` (`-iname` case-insensitive) |
| By type | `-type f/d/l` |
| By size | `-size +100M` |
| By owner | `-user u`, `-group g`, `-nouser` |
| By time | `-mtime -1`, `-mmin -60`, `-newer f` |
| By permission | `-perm -4000` (all), `/022` (any) |
| Act | `-delete`, `-exec cmd {} \;`, `-exec cmd {} +`, `-print0 \| xargs -0` |
| Fast name search | `locate` (after `updatedb`) |
| What is this command? | `type -a`, `which`, `whereis` |

## Check yourself

1. What is the difference between `-perm -0600` and `-perm /0600`?
2. Why is `-exec … {} +` faster than `-exec … {} \;`?
3. A file created five minutes ago is not found by `locate`. Why?
4. Write a command that lists files under `/srv` larger than 50 MiB and not modified in 90 days.

<details>
<summary>Answers</summary>

1. `-0600` matches files where **both** owner-read and owner-write are set; `/0600` matches files where **either** is set.
2. `+` passes many file names to a single command invocation instead of starting one process per file.
3. `locate` uses a database built by `updatedb`, which hasn't run since the file was created.
4. `find /srv -type f -size +50M -mtime +90`

</details>

## Further reading

- `man 1 find`, `man 1 xargs`, `man 8 updatedb`
- [GNU Findutils](https://www.gnu.org/software/findutils/)
- [man7.org: find(1)](https://man7.org/linux/man-pages/man1/find.1.html)

---

<!-- footer:start -->
[← The Linux Filesystem Hierarchy](01-filesystem-hierarchy.md) · [Module index](README.md) · [Working with Text →](03-working-with-text.md)  
**Practice now:** [Exercises – Finding Files](../../exercises/01-foundations/02-finding-files.md)
<!-- footer:end -->
