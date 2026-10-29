# Working with Text

<!-- nav:start -->
> **Module 01 · Foundations & Essential Commands** — lesson 3 of 6  
> [Module index](README.md) · [Course lesson (commands)](../../course/01-Foundations.md#lesson-3---working-with-text) · [Exercises for this lesson](../../exercises/01-foundations/03-working-with-text.md)
<!-- nav:end -->

Linux configuration files, logs and command output are all **plain text**. The classic Unix tools – `grep`, `sed`, `awk`, `cut`, `sort`, `uniq`, `tr`, `wc` – let you search, extract and transform that text without opening an editor. In this lesson you will learn what each tool is for, how regular expressions work, and how to combine the tools into pipelines.

## What you will learn

- How to view text efficiently (`cat`, `less`, `head`, `tail`)
- Basic and extended **regular expressions**
- Searching with `grep`
- Editing streams and files with `sed`
- Extracting fields and computing with `awk` and `cut`
- Sorting, counting and comparing with `sort`, `uniq`, `wc`, `diff`

---

## Key terms

| Term | Definition |
|------|-----------|
| **Stream** | Text flowing through a program line by line (stdin → program → stdout). |
| **Filter** | A program that reads a stream, transforms it and writes a stream: `grep`, `sort`, `sed`… |
| **Regular expression (regex)** | A pattern language describing sets of strings, e.g. `^[0-9]+$` = "a line made only of digits". |
| **BRE / ERE** | *Basic* and *Extended* regular expressions. ERE (`grep -E`, `sed -E`) needs fewer backslashes. |
| **Field** | A column in a line, separated by a **delimiter** (`:` in `/etc/passwd`, whitespace in most output). |
| **In-place edit** | Modifying the file itself (`sed -i`) rather than printing the result. |

---

## Viewing text

```console
$ cat /etc/hostname                 # print whole file
$ less /var/log/syslog              # page through (q quit, / search, n next, G end, g start)
$ head -n 5 /etc/passwd             # first 5 lines
$ tail -n 20 /var/log/syslog        # last 20 lines
$ tail -f /var/log/nginx/access.log # follow new lines as they arrive (Ctrl+C to stop)
$ wc -l /etc/passwd                 # count lines (-w words, -c bytes)
36 /etc/passwd
```

`cat -n` numbers lines; `cat -A` shows hidden characters (tabs as `^I`, line ends as `$`) – very useful when a config file "looks right" but doesn't work.

---

## Regular expressions

A regex describes **a pattern** to match inside a line.

| Pattern | Matches | Example | Matches lines like |
|---------|---------|---------|--------------------|
| `abc` | literal text | `error` | `disk error on sda` |
| `.` | any single character | `h.t` | `hat`, `hot`, `h9t` |
| `^` | start of line | `^root` | `root:x:0:0…` |
| `$` | end of line | `bash$` | `alice:…:/bin/bash` |
| `[abc]` | one of the listed characters | `[Ee]rror` | `Error`, `error` |
| `[^abc]` | any char **except** those | `^[^#]` | lines not starting with `#` |
| `[0-9]`, `[a-z]` | a range | `[0-9][0-9]` | two digits |
| `[[:digit:]]`, `[[:space:]]`, `[[:alpha:]]` | POSIX classes (locale-safe) | | |
| `*` | 0 or more of previous | `ab*c` | `ac`, `abc`, `abbbc` |
| `+` (ERE) | 1 or more | `[0-9]+` | `7`, `2026` |
| `?` (ERE) | 0 or 1 | `colou?r` | `color`, `colour` |
| `{n,m}` (ERE) | between n and m times | `[0-9]{1,3}` | `5`, `42`, `255` |
| `a\|b` (ERE) | alternation (BRE: `a\\|b`) | `fail(ed\|ure)` | `failed`, `failure` |
| `( )` (ERE) | group | `(ab)+` | `ab`, `abab` |
| `\` | escape a special char | `\.conf$` | lines ending in `.conf` |

> **BRE vs ERE:** in basic regex (`grep`, `sed` without `-E`) the characters `+ ? | ( ) { }` are literal unless escaped (`\+`, `\(`). With `-E` they are special. Rule of thumb: **always use `grep -E` and `sed -E`**.

> **Regex vs glob:** in the shell, `*.conf` is a *glob* meaning "anything ending in .conf". In a regex, `*` means "repeat the previous character" – the regex for the same idea is `\.conf$`.

---

## grep – search lines

`grep PATTERN FILE…` prints the lines that match.

```console
$ grep root /etc/passwd
root:x:0:0:root:/root:/bin/bash

$ grep -n "PermitRootLogin" /etc/ssh/sshd_config      # -n line numbers
33:#PermitRootLogin prohibit-password

$ grep -i "error" /var/log/syslog | tail -3           # -i ignore case
$ grep -c "/bin/bash$" /etc/passwd                    # -c count matches
4
$ grep -v "^#" /etc/fstab                             # -v invert: lines NOT matching
$ grep -r "Listen" /etc/apache2/                      # -r recursive through a directory
$ grep -rl "server_name" /etc/nginx/                  # -l only file names
$ grep -w "ssh" /etc/services                         # -w whole word
$ grep -E "^(root|alice):" /etc/passwd                # extended regex
$ grep -A2 -B1 "Failed password" /var/log/auth.log    # 2 lines After, 1 Before
$ grep -o -E "[0-9]{1,3}(\.[0-9]{1,3}){3}" access.log # -o print only the matched part (IPs)
```

### Example: show only the active lines of a config file

```console
$ grep -Ev '^\s*(#|$)' /etc/ssh/sshd_config
Include /etc/ssh/sshd_config.d/*.conf
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding yes
PrintMotd no
AcceptEnv LANG LC_*
Subsystem       sftp    /usr/lib/openssh/sftp-server
```

`^\s*(#|$)` = "optional whitespace then either `#` or end of line" – i.e. comments and blank lines; `-v` removes them.

---

## sed – the stream editor

`sed` applies editing commands to each line. The most important is **substitute**: `s/regex/replacement/flags`.

```console
$ echo "hello world world" | sed 's/world/Linux/'      # first match per line
hello Linux world
$ echo "hello world world" | sed 's/world/Linux/g'     # g = all matches
hello Linux Linux
$ sed 's/#Port 22/Port 2222/' /etc/ssh/sshd_config | grep ^Port
Port 2222
```

Without `-i`, sed only **prints** the result – the file is unchanged. Use that to test.

| Command | Meaning |
|---------|---------|
| `sed -n '10,20p' f` | print only lines 10–20 (`-n` suppresses default output) |
| `sed -n '/start/,/end/p' f` | print from a line matching *start* to one matching *end* |
| `sed '5d' f` | delete line 5 |
| `sed '/^#/d' f` | delete comment lines |
| `sed '/^$/d' f` | delete empty lines |
| `sed '3i\new line' f` | insert before line 3 |
| `sed '/^\[main\]/a key=value' f` | append after a matching line |
| `sed -i 's/a/b/g' f` | edit the file **in place** |
| `sed -i.bak 's/a/b/g' f` | in place, keeping `f.bak` as backup |
| `sed -E 's/([0-9]+)-([0-9]+)/\2-\1/' f` | groups and back-references (swap two numbers) |

Any character can be the delimiter, which avoids escaping slashes in paths:

```console
$ sed 's|/var/www/html|/srv/web|g' site.conf
```

### Example: safely change a setting in place

```console
$ sudo cp -a /etc/ssh/sshd_config{,.bak}                         # backup first
$ sudo sed -i -E 's/^#?PasswordAuthentication .*/PasswordAuthentication no/' /etc/ssh/sshd_config
$ grep ^PasswordAuthentication /etc/ssh/sshd_config
PasswordAuthentication no
```

The regex `^#?PasswordAuthentication .*` matches the line whether or not it is commented out.

---

## cut – extract columns

```console
$ cut -d: -f1 /etc/passwd | head -3        # field 1, delimiter ":"
root
daemon
bin
$ cut -d: -f1,7 /etc/passwd | tail -2      # fields 1 and 7
alice:/bin/bash
bob:/bin/bash
$ cut -c1-8 /etc/hostname                  # characters 1–8
```

`cut` handles only a **single-character** delimiter. For columns separated by variable spaces (like `ps` output), use `awk`.

---

## awk – fields, conditions and calculations

`awk` processes each line as **fields** `$1`, `$2`, … (`$0` = whole line, `NF` = number of fields, `NR` = line number). A program is `pattern { action }`.

```console
$ awk -F: '{ print $1, $7 }' /etc/passwd | head -2    # -F sets the separator
root /bin/bash
daemon /usr/sbin/nologin

$ awk -F: '$3 >= 1000 && $3 < 65534 { print $1 }' /etc/passwd   # regular users
alice
bob

$ df -h | awk 'NR > 1 { print $6, $5 }'      # skip header, mount point and Use%
$ ps aux | awk '{ sum += $6 } END { print sum/1024 " MiB RSS total" }'
1432.5 MiB RSS total
$ awk '/Failed password/ { print $(NF-3) }' /var/log/auth.log | sort | uniq -c   # IPs of failed logins
```

`BEGIN { }` runs before the first line, `END { }` after the last – used for headers and totals.

---

## sort, uniq, tr, wc

```console
$ sort names.txt                          # alphabetical
$ sort -n sizes.txt                       # numeric
$ sort -rn                                # reverse numeric
$ sort -t: -k3 -n /etc/passwd             # by field 3 (UID), ":" delimiter
$ sort -u list.txt                        # sort and remove duplicates
$ du -sh /var/* | sort -h                 # human sizes (K, M, G)

$ uniq                                    # removes ADJACENT duplicates → sort first!
$ sort access.log | uniq -c | sort -rn | head   # frequency table

$ tr 'a-z' 'A-Z' < file                   # translate characters
$ tr -d '\r' < windows.txt > unix.txt     # delete carriage returns
$ tr -s ' ' < file                        # squeeze repeated spaces
```

### Example: the classic "top 5" pipeline

Which shells are used by accounts on this system, and how often?

```console
$ cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn
     22 /usr/sbin/nologin
      4 /bin/bash
      1 /bin/sync
      1 /bin/false
```

Each tool does one small job; the pipe `|` connects them. This "small tools + pipes" approach is the Unix philosophy.

---

## Comparing files

```console
$ diff old.conf new.conf          # lines to change
$ diff -u old.conf new.conf       # unified format (like git diff)
--- old.conf
+++ new.conf
@@ -1,3 +1,3 @@
 Port 22
-PermitRootLogin yes
+PermitRootLogin no
$ cmp a.bin b.bin                 # byte comparison (binary files)
$ comm -12 <(sort a) <(sort b)    # lines common to both
```

---

## Try it yourself

1. Print all usernames with UID ≥ 1000, sorted alphabetically.
2. Show `/etc/login.defs` without comments and blank lines.
3. Using `sed` (without `-i`), print `/etc/hosts` with every `localhost` replaced by `LOCAL`.
4. From `ps aux`, print the PID and command of the 3 processes using the most memory.
5. Count how many lines in `/etc/services` mention `tcp`.

---

## Common mistakes

- Using `uniq` without `sort` – it only removes *adjacent* duplicates.
- Forgetting `-E` and wondering why `+` or `|` don't work.
- Running `sed -i` with an untested expression – always run without `-i` first or use `-i.bak`.
- Using `cut -d' '` on output with variable spacing – use `awk` instead.
- Regex `.` means *any character*: `grep 192.168.1.1` also matches `192x168y1z1`. Escape: `192\.168\.1\.1` or use `grep -F` (fixed string).

---

## Summary

| Tool | Use it to… |
|------|-----------|
| `grep` | find lines matching a pattern |
| `sed` | substitute, delete or print lines; edit files in place |
| `awk` | work with columns, conditions and arithmetic |
| `cut` | extract fields with a single-char delimiter |
| `sort` / `uniq -c` | order lines and count duplicates |
| `tr` | translate or delete characters |
| `wc` | count lines, words, bytes |
| `diff -u` | compare two files |

## Check yourself

1. What does `grep -v '^$'` do?
2. Why does `sed 's/a/b/'` on a file not change the file?
3. Write an `awk` command printing the 1st and 3rd field of `/etc/group`.
4. What's the regex for "a line that ends with `.log`"?

<details>
<summary>Answers</summary>

1. Prints all non-empty lines (removes empty lines).
2. Without `-i`, sed writes the result to stdout and leaves the file untouched.
3. `awk -F: '{print $1, $3}' /etc/group`
4. `\.log$`

</details>

## Further reading

- `man 7 regex`, `man 1 grep`, `man 1 sed`, `man 1 awk`
- [GNU grep manual](https://www.gnu.org/software/grep/manual/grep.html) · [GNU sed manual](https://www.gnu.org/software/sed/manual/sed.html) · [GNU Awk User's Guide](https://www.gnu.org/software/gawk/manual/gawk.html)
- [The Linux Command Line – Chapters 19–20: Regular Expressions, Text Processing](https://linuxcommand.org/tlcl.php)
- [regex101](https://regex101.com/) – test regular expressions interactively

---

<!-- footer:start -->
[← Finding Files](02-finding-files.md) · [Module index](README.md) · [Redirection, Pipes and Archives →](04-redirection-pipes-archives.md)  
**Practice now:** [Exercises – Working with Text](../../exercises/01-foundations/03-working-with-text.md)
<!-- footer:end -->
