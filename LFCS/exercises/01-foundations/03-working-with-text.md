# Exercises – Working with Text

<!-- nav:start -->
> **Module 01 · Foundations & Essential Commands** — lesson 3 of 6  
> Read first: [Theory – Working with Text](../../theory/01-foundations/03-working-with-text.md) · [Course lesson (commands)](../../course/01-Foundations.md#lesson-3---working-with-text) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Setup

Create a sample log file:

```bash
mkdir -p /root/textlab && cd /root/textlab
cat > access.log << 'EOF'
192.168.56.10 - - [21/Oct/2026:10:00:01 +0000] "GET /index.html HTTP/1.1" 200 512
192.168.56.11 - - [21/Oct/2026:10:00:05 +0000] "GET /login HTTP/1.1" 200 1024
192.168.56.10 - - [21/Oct/2026:10:00:09 +0000] "POST /login HTTP/1.1" 401 64
10.0.0.5 - - [21/Oct/2026:10:01:00 +0000] "GET /admin HTTP/1.1" 403 128
192.168.56.10 - - [21/Oct/2026:10:01:30 +0000] "POST /login HTTP/1.1" 401 64
192.168.56.12 - - [21/Oct/2026:10:02:00 +0000] "GET /index.html HTTP/1.1" 200 512
10.0.0.5 - - [21/Oct/2026:10:02:30 +0000] "GET /missing HTTP/1.1" 404 0
192.168.56.10 - - [21/Oct/2026:10:03:00 +0000] "POST /login HTTP/1.1" 200 2048
EOF
cp /etc/ssh/sshd_config ./sshd_config
```

Fields (space-separated): `$1` IP, `$4`–`$5` time, `$6` method, `$7` path, `$9` status, `$10` bytes.

---

## Exercise 1 – grep basics ★

1. Show all lines with status `401`.
2. Count requests from `10.0.0.5`.
3. Show lines that are **not** `GET` requests.
4. Show the line numbers of requests to `/login`.

**Verify:** (1) 2 lines, (2) 2, (3) 3 lines, (4) lines 2, 3, 5, 8.

<details>
<summary>Solution</summary>

```bash
grep '" 401 ' access.log
grep -c '^10\.0\.0\.5 ' access.log
grep -v '"GET ' access.log
grep -n '/login' access.log
```

</details>

---

## Exercise 2 – Regular expressions ★★

1. Show requests whose status is any `4xx` code (use ERE).
2. Extract only the IP addresses (use `grep -oE`), unique, sorted.
3. Show active (non-comment, non-blank) lines of `sshd_config`.

**Verify:** (1) 4 lines, (2) 4 unique IPs.

<details>
<summary>Solution</summary>

```bash
grep -E '" 4[0-9]{2} ' access.log
grep -oE '^[0-9]{1,3}(\.[0-9]{1,3}){3}' access.log | sort -u
grep -Ev '^\s*(#|$)' sshd_config
```

</details>

---

## Exercise 3 – sed ★★

Work on the copy `sshd_config` in `/root/textlab`.

1. Print lines 10–15.
2. Without `-i`, show the file with `#Port 22` replaced by `Port 2222`.
3. In place (keep a `.bak`), set `PermitRootLogin no` whether the line is currently commented or not.
4. Delete all comment lines in place.

**Verify:** `grep -E '^(Port|PermitRootLogin)' sshd_config` and `ls sshd_config.bak`.

<details>
<summary>Solution</summary>

```bash
sed -n '10,15p' sshd_config
sed 's/^#Port 22/Port 2222/' sshd_config | grep -n '^Port'
sed -i.bak -E 's/^#?PermitRootLogin .*/PermitRootLogin no/' sshd_config
sed -i '/^\s*#/d' sshd_config
```

</details>

---

## Exercise 4 – cut and awk ★★

1. With `cut`, print usernames and home directories from `/etc/passwd`.
2. With `awk`, print usernames of accounts whose UID is between 1000 and 60000, sorted.
3. With `awk`, sum the bytes column (`$10`) of `access.log`.
4. With `awk`, print the path (`$7`) of every request with status ≥ 400.

**Verify:** (3) total = 4352.

<details>
<summary>Solution</summary>

```bash
cut -d: -f1,6 /etc/passwd
awk -F: '$3>=1000 && $3<=60000 {print $1}' /etc/passwd | sort
awk '{s+=$10} END {print s}' access.log
awk '$9>=400 {print $7}' access.log
```

</details>

---

## Exercise 5 – sort, uniq, tr, wc ★★

1. Produce a table of requests per IP, most active first.
2. Produce a table of requests per status code.
3. Convert `access.log` to uppercase into `upper.log`.
4. How many distinct paths were requested?

**Verify:** `192.168.56.10` appears first with count 4; 4 distinct paths.

<details>
<summary>Solution</summary>

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn
awk '{print $9}' access.log | sort | uniq -c
tr 'a-z' 'A-Z' < access.log > upper.log
awk '{print $7}' access.log | sort -u | wc -l
```

</details>

---

## Exercise 6 – diff ★

Compare `sshd_config.bak` with `sshd_config` using unified format. Save the diff to `/root/textlab/changes.diff`.

**Verify:** the diff contains `-#PermitRootLogin` and `+PermitRootLogin no`.

<details>
<summary>Solution</summary>

```bash
diff -u sshd_config.bak sshd_config > changes.diff
grep -E '^[-+]PermitRootLogin|^[-+]#PermitRootLogin' changes.diff
```

</details>

---

## Challenge – Brute-force report ★★★

Write a one-liner that prints, for each IP with **two or more** failed logins (`POST /login` with status `401`), the IP and the number of failures, e.g. `192.168.56.10 2`. Then write the same with a single `awk` program (no `grep`, `sort` or `uniq`).

**Verify:** output is exactly `192.168.56.10 2`.

<details>
<summary>Solution</summary>

```bash
grep '"POST /login' access.log | awk '$9==401 {print $1}' | sort | uniq -c | awk '$1>=2 {print $2, $1}'

awk '$6=="\"POST" && $7=="/login" && $9==401 {c[$1]++} END {for (ip in c) if (c[ip]>=2) print ip, c[ip]}' access.log
```

</details>

---

## Cleanup

```bash
rm -rf /root/textlab
```

---

<!-- footer:start -->
[← Exercises: Finding Files](02-finding-files.md) · [Module exercises](README.md) · [Exercises: Redirection, Pipes and Archives →](04-redirection-pipes-archives.md)
<!-- footer:end -->
