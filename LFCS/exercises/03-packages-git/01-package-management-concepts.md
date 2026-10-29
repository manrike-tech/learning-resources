# Exercises – Package Management Concepts

<!-- nav:start -->
> **Module 03 · Package Management & Git** — lesson 1 of 5  
> Read first: [Theory – Package Management Concepts](../../theory/03-packages-git/01-package-management-concepts.md) · [Course lesson (commands)](../../course/03-Packages-and-Git.md#lesson-1---package-management-concepts-ch-7) · [Module exercises](README.md)
<!-- nav:end -->

> Do each exercise on **both** `deb1` and `rhel1` and compare.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Identify your tools ★

For each VM, record in `/root/pkg-tools.txt`: the package format, the low-level tool, the high-level tool, the database directory, and the repository configuration directory.

<details>
<summary>Solution</summary>

| | deb1 | rhel1 |
|--|------|-------|
| format | `.deb` | `.rpm` |
| low-level | `dpkg` | `rpm` |
| high-level | `apt` | `dnf` |
| database | `/var/lib/dpkg` | `/var/lib/rpm` |
| repos | `/etc/apt/sources.list.d/` | `/etc/yum.repos.d/` |

</details>

---

## Exercise 2 – Count and list ★

1. Count installed packages.
2. List the 5 largest installed packages.

<details>
<summary>Solution</summary>

```bash
# deb1
dpkg -l | grep -c '^ii'
dpkg-query -W -f='${Installed-Size}\t${Package}\n' | sort -rn | head -5
# rhel1
rpm -qa | wc -l
rpm -qa --qf '%{SIZE}\t%{NAME}\n' | sort -rn | head -5
```

</details>

---

## Exercise 3 – Low-level vs high-level ★★

1. Download the `.deb`/`.rpm` of a package with dependencies that isn't installed (e.g. `nmap`, or `tmux`) **without installing it**.
2. Try installing it with the low-level tool. What happens?
3. Install it with the high-level tool from the local file.

<details>
<summary>Solution</summary>

```bash
# deb1
cd /tmp && apt download tmux
dpkg -i tmux_*.deb           # may fail with "dependency problems" (e.g. libutempter0)
apt install -y ./tmux_*.deb  # or: apt -f install
# rhel1
cd /tmp && dnf download tmux
rpm -ivh tmux-*.rpm          # fails if a dependency is missing
dnf install -y ./tmux-*.rpm
```

</details>

---

## Exercise 4 – Inspect a package file ★

Without installing, list the files and the dependencies of the downloaded package.

<details>
<summary>Solution</summary>

```bash
dpkg -c /tmp/tmux_*.deb; dpkg -I /tmp/tmux_*.deb | grep Depends
rpm -qlp /tmp/tmux-*.rpm; rpm -qRp /tmp/tmux-*.rpm
```

</details>

---

## Exercise 5 – Trust ★★

1. List the signing keys trusted on each VM.
2. On rhel1, check the signature of the downloaded RPM.
3. Explain in one sentence why `gpgcheck=0` is dangerous.

<details>
<summary>Solution</summary>

```bash
ls /etc/apt/keyrings /usr/share/keyrings /etc/apt/trusted.gpg.d 2>/dev/null
rpm -qa gpg-pubkey* --qf '%{NAME}-%{VERSION}\t%{SUMMARY}\n'
rpm -K /tmp/tmux-*.rpm
```

Without signature checks, anyone able to tamper with a mirror or the network could deliver packages whose scripts run as root.

</details>

---

## Exercise 6 – Orphans ★★

Install a package that pulls in dependencies, remove only the package, list the orphaned dependencies and clean them up.

<details>
<summary>Solution</summary>

```bash
# deb1
apt install -y nmap; apt remove -y nmap; apt autoremove --dry-run; apt autoremove -y
# rhel1
dnf install -y nmap; dnf remove -y nmap     # dnf removes unneeded deps automatically (clean_requirements_on_remove)
dnf autoremove
```

</details>

---

## Challenge – Config files across upgrades ★★★

On rhel1:
1. Install `chrony` (if missing) and add a comment line to `/etc/chrony.conf`.
2. Reinstall the package (`dnf reinstall chrony`). Was your change kept? Is there an `.rpmnew`?
3. Use `rpm -qc` and `rpm -V` to explain.

On deb1, do the same with `/etc/chrony/chrony.conf` and `apt install --reinstall chrony`.

<details>
<summary>Solution</summary>

`chrony.conf` is marked `%config(noreplace)`: your modified file is kept and, if the package version differs, the new one is written as `.rpmnew`. `rpm -V chrony` shows `S.5....T.  c /etc/chrony.conf`. On Debian, dpkg keeps the modified conffile and asks only if the package's version of the file changed too.

</details>

---

<!-- footer:start -->
[Module exercises](README.md) · [Exercises: Debian Packages: dpkg and APT →](02-dpkg-apt.md)
<!-- footer:end -->
