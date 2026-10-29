# Exercises – The User Environment

<!-- nav:start -->
> **Module 01 · Foundations & Essential Commands** — lesson 5 of 6  
> Read first: [Theory – The User Environment](../../theory/01-foundations/05-user-environment.md) · [Course lesson (commands)](../../course/01-Foundations.md#lesson-5---the-user-environment-ch-3) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Shell vs environment variables ★

1. Set `FRUIT=apple` without exporting; check whether a child `bash -c 'echo $FRUIT'` sees it.
2. Export it and test again.
3. Run `FRUIT=pear bash -c 'echo $FRUIT'` and then `echo $FRUIT`. Explain.

**Verify:** outputs are: empty, `apple`, `pear` then `apple`.

<details>
<summary>Solution</summary>

```bash
FRUIT=apple; bash -c 'echo $FRUIT'     # empty
export FRUIT; bash -c 'echo $FRUIT'    # apple
FRUIT=pear bash -c 'echo $FRUIT'       # pear – set only for that command
echo $FRUIT                            # apple – unchanged in the parent
```

</details>

---

## Exercise 2 – Child can't change parent ★★

Create `~/goto.sh` containing `cd /var/log; export WHERE=logs`.

1. Run it with `bash ~/goto.sh`; check `pwd` and `$WHERE`.
2. Run it with `source ~/goto.sh`; check again.

**Verify:** only after `source` are you in `/var/log` with `WHERE=logs`.

<details>
<summary>Solution</summary>

```bash
printf 'cd /var/log\nexport WHERE=logs\n' > ~/goto.sh
bash ~/goto.sh; pwd; echo "$WHERE"      # unchanged
source ~/goto.sh; pwd; echo "$WHERE"    # /var/log, logs
```

</details>

---

## Exercise 3 – Expansion and quoting ★★

In an empty directory with files `a.txt` and `b.txt`, predict and run:

```bash
echo *.txt
echo "*.txt"
echo '$HOME'
echo "$HOME"
echo file{1..3}.log
echo $((7 * 6))
name="my file"; touch $name; ls; rm -f my file; touch "$name"; ls
```

**Verify:** explain each output in one sentence.

<details>
<summary>Solution</summary>

- `a.txt b.txt` – glob expanded; `*.txt` – quoted, not expanded; `$HOME` – single quotes; `/home/you` – double quotes allow variables; `file1.log file2.log file3.log` – brace expansion; `42` – arithmetic.
- Unquoted `$name` is split into two words → creates `my` and `file`. Quoted creates one file `my file`.

</details>

---

## Exercise 4 – PATH ★★

1. Create `~/bin/hello` (a script printing `hi`), make it executable.
2. Run `hello` – why does it fail (or succeed on Ubuntu)? Add `~/bin` to `PATH` for the current session and run it.
3. Make it persistent for your user only.
4. Break your PATH on purpose (`export PATH=/nothing`), observe, then repair it without logging out.

**Verify:** after a new login, `type hello` shows `~/bin/hello`.

<details>
<summary>Solution</summary>

```bash
mkdir -p ~/bin && printf '#!/bin/bash\necho hi\n' > ~/bin/hello && chmod +x ~/bin/hello
export PATH="$PATH:$HOME/bin"; hello
echo 'export PATH="$PATH:$HOME/bin"' >> ~/.bashrc
export PATH=/nothing; ls          # command not found
export PATH=/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```

Ubuntu's default `~/.profile` already adds `~/bin` if it exists at login.

</details>

---

## Exercise 5 – Aliases, functions and history ★

1. Create alias `ports='ss -tulpn'` and a function `mkcd` that creates a directory and enters it.
2. Make both persistent for your user.
3. Use history expansion to re-run the last command that started with `ss`.

**Verify:** in a new shell, `type ports mkcd` shows alias and function.

<details>
<summary>Solution</summary>

```bash
cat >> ~/.bashrc << 'EOF'
alias ports='ss -tulpn'
mkcd() { mkdir -p "$1" && cd "$1"; }
EOF
source ~/.bashrc
type ports mkcd
!ss
```

</details>

---

## Exercise 6 – System-wide variables ★★

1. Make `COMPANY=acme` available to **every** user at login.
2. Make `EDITOR=vim` the default editor for every user.
3. Test with another user via `su - otheruser`.

**Verify:** `su - otheruser -c 'echo $COMPANY $EDITOR'` prints `acme vim`.

<details>
<summary>Solution</summary>

```bash
cat > /etc/profile.d/company.sh << 'EOF'
export COMPANY=acme
export EDITOR=vim
EOF
chmod 644 /etc/profile.d/company.sh
useradd -m otheruser 2>/dev/null
su - otheruser -c 'echo $COMPANY $EDITOR'
```

</details>

---

## Exercise 7 – Login vs non-login ★★

Add `echo "profile.d ran"` to `/etc/profile.d/trace.sh` and `echo "bashrc ran"` to your `~/.bashrc`. Then start shells with `bash`, `bash -l`, `su yourself`, `su - yourself`, `ssh localhost`, and `bash -c true`. Record which messages appear for each.

**Verify:** non-login interactive shells show only `bashrc ran`; login shells show both; `bash -c` shows neither.

<details>
<summary>Solution</summary>

| Started with | profile.d | bashrc |
|--------------|-----------|--------|
| `bash` | – | ✔ |
| `bash -l` | ✔ | ✔ (sourced by `~/.bash_profile`/`~/.profile`) |
| `su user` | – | ✔ |
| `su - user` | ✔ | ✔ |
| `ssh localhost` | ✔ | ✔ |
| `bash -c true` | – | – |

Remove the trace lines afterwards.

</details>

---

## Challenge – Onboarding defaults ★★★

Configure the system so that **every new user** automatically gets:

- a `~/README.txt` with the text `Welcome! Read /etc/motd`,
- an alias `ll='ls -alF'` in their `~/.bashrc`,
- a `~/bin` directory,

without affecting existing users. Prove it by creating `newbie`.

**Verify:** `su - newbie -c 'cat README.txt; type ll; ls -d bin'` works.

<details>
<summary>Solution</summary>

```bash
echo 'Welcome! Read /etc/motd' > /etc/skel/README.txt
echo "alias ll='ls -alF'" >> /etc/skel/.bashrc
mkdir -p /etc/skel/bin
useradd -m -s /bin/bash newbie
su - newbie -c 'cat README.txt; type ll; ls -d bin'
```

`/etc/skel` is copied only when a home directory is created, so existing users are unaffected.

</details>

---

## Cleanup

```bash
rm -f /etc/profile.d/trace.sh ~/goto.sh
userdel -r otheruser; userdel -r newbie
```

---

<!-- footer:start -->
[← Exercises: Redirection, Pipes and Archives](04-redirection-pipes-archives.md) · [Module exercises](README.md) · [Exercises: Finding Help Offline →](06-finding-help.md)
<!-- footer:end -->
