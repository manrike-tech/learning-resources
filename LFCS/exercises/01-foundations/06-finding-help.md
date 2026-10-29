# Exercises – Finding Help Offline

<!-- nav:start -->
> **Module 01 · Foundations & Essential Commands** — lesson 6 of 6  
> Read first: [Theory – Finding Help Offline](../../theory/01-foundations/06-finding-help.md) · [Course lesson (commands)](../../course/01-Foundations.md#lesson-6---finding-help-offline) · [Module exercises](README.md)
<!-- nav:end -->

> Rule for all exercises: **no internet**. Use only `man`, `--help`, `info` and `/usr/share/doc`.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Sections ★

1. List all manual sections that contain a page named `crontab`.
2. Open the one that describes the crontab **file format**.
3. Find which section documents `/etc/shadow`.

**Verify:** `crontab(1)` is the command, `crontab(5)` the format; shadow is in section 5.

<details>
<summary>Solution</summary>

```bash
man -f crontab        # crontab (1), crontab (5)
man 5 crontab
man -f shadow         # shadow (5), shadow (3)
```

</details>

---

## Exercise 2 – Searching by keyword ★

Using only `man -k`/`apropos`, find the commands to:

1. change a user's password expiry information,
2. display the kernel ring buffer,
3. show block devices,
4. resize an ext4 filesystem.

**Verify:** `chage`, `dmesg`, `lsblk`, `resize2fs`.

<details>
<summary>Solution</summary>

```bash
man -k "password expiry"        # chage (1)
man -k "ring buffer"            # dmesg (1)
man -k "block devices"          # lsblk (8)
man -k ext4 | grep -i resiz     # resize2fs (8)
```

If nothing is found, run `sudo mandb` first.

</details>

---

## Exercise 3 – Reading a page efficiently ★★

1. In `man 5 fstab`, find what the **fifth** and **sixth** fields mean.
2. In `man useradd`, find the option that sets the account expiry date.
3. In `man find`, jump to the EXAMPLES section.

**Verify:** `dump` / `pass` (fsck order); `-e EXPIRE_DATE`.

<details>
<summary>Solution</summary>

Inside `man`: type `/fifth`, `/-e,`, `/^EXAMPLES` and press Enter; `n` for the next match.

</details>

---

## Exercise 4 – `--help` vs man ★

Find, using `--help` and `grep`:

1. the `systemctl` command that lists timers,
2. the `ip route` subcommand that shows which route is used for an address,
3. the `nmcli connection` subcommand that reloads profiles from disk.

**Verify:** `list-timers`, `ip route get`, `nmcli connection reload`.

<details>
<summary>Solution</summary>

```bash
systemctl --help | grep -i timer
ip route help 2>&1 | grep get
nmcli connection help 2>&1 | grep -i reload
```

</details>

---

## Exercise 5 – Documentation and examples on disk ★★

1. Find the documentation directory of the `openssh` server package.
2. Find an example config shipped with any installed service (`find /usr/share/doc -name "*example*"`).
3. Show the full systemd unit of the SSH service, to use as a template.
4. List the configuration files that belong to the SSH server package.

**Verify:** you have a list of example files and the unit content.

<details>
<summary>Solution</summary>

```bash
ls -d /usr/share/doc/openssh*
find /usr/share/doc -iname "*example*" | head
systemctl cat ssh 2>/dev/null || systemctl cat sshd
dpkg -L openssh-server | grep /etc/     # Debian
rpm -qc openssh-server                  # RHEL
```

</details>

---

## Exercise 6 – systemd and network directive references ★★

1. Find which man page documents the `MemoryMax=` directive.
2. Find nmcli usage examples on the system.
3. Find the syntax of firewalld rich rules.

**Verify:** `systemd.resource-control(5)`, `nmcli-examples(7)`, `firewalld.richlanguage(5)`.

<details>
<summary>Solution</summary>

```bash
man systemd.directives    # then /MemoryMax
man 7 nmcli-examples
man 5 firewalld.richlanguage
```

</details>

---

## Challenge – Exam simulation ★★★

Set a 10-minute timer. Using only local documentation, find out how to:

1. make the systemd journal persistent (which file, which option),
2. set a GRUB menu timeout of 3 seconds (which file, which variable, which command applies it on your distro),
3. configure chrony to act as an NTP server for `192.168.56.0/24` (which directive).

Write the answers in `/root/help-challenge.txt`.

<details>
<summary>Solution</summary>

1. `man journald.conf` → `/etc/systemd/journald.conf`, `Storage=persistent` (or create `/var/log/journal`).
2. `/etc/default/grub` → `GRUB_TIMEOUT=3`; apply with `update-grub` (Debian) or `grub2-mkconfig -o /boot/grub2/grub.cfg` (RHEL). Found via `info grub` / comments in the file.
3. `man chrony.conf` → `allow 192.168.56.0/24`.

</details>

---

<!-- footer:start -->
[← Exercises: The User Environment](05-user-environment.md) · [Module exercises](README.md)
<!-- footer:end -->
