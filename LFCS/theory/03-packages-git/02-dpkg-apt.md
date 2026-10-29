# Debian Packages: dpkg and APT

<!-- nav:start -->
> **Module 03 · Package Management & Git** — lesson 2 of 5  
> [Module index](README.md) · [Course lesson (commands)](../../course/03-Packages-and-Git.md#lesson-2---dpkg--apt-ch-89) · [Exercises for this lesson](../../exercises/03-packages-git/02-dpkg-apt.md)
<!-- nav:end -->

Debian, Ubuntu and their derivatives use `.deb` packages, managed by the low-level tool **dpkg** and the high-level tool **APT**. In this lesson you will learn the daily commands to search, install, upgrade, remove and query packages, how to configure repositories and signing keys, and how to answer the classic questions "which package owns this file?" and "which package provides this command?".

## What you will learn

- Installing, upgrading and removing with `apt`
- Searching and inspecting packages (`apt search`, `apt show`, `apt-cache policy`)
- Querying installed packages with `dpkg` (`-l`, `-L`, `-S`, `-s`)
- Verifying installed files
- Finding which package provides a file (`apt-file`)
- Repository configuration (`.sources` deb822 and `.list`), keys and pinning
- Holding packages and installing local `.deb` files
- Logs and troubleshooting

---

## Key terms

| Term | Definition |
|------|-----------|
| **dpkg** | Debian's low-level package tool: installs `.deb` files and maintains the database in `/var/lib/dpkg/`. |
| **APT** | *Advanced Package Tool*: resolves dependencies and fetches packages from repositories. Commands: `apt` (interactive), `apt-get`/`apt-cache` (stable for scripts). |
| **Suite** | A release channel in a repository: `noble`, `noble-updates`, `noble-security` (Ubuntu), `bookworm` (Debian). |
| **Component** | A section of a repository: `main`, `universe`, `restricted`, `multiverse` (Ubuntu); `main`, `contrib`, `non-free` (Debian). |
| **Candidate** | The version APT would install now (highest priority/version available). |
| **Hold** | A mark preventing a package from being upgraded. |
| **Purge** | Remove a package **and** its configuration files. |

---

## Everyday APT commands

```console
$ sudo apt update                       # refresh repository metadata – always first
$ apt list --upgradable                 # what would be upgraded
$ sudo apt upgrade                      # upgrade installed packages (never removes any)
$ sudo apt full-upgrade                 # upgrade, removing/adding packages if needed (dist-upgrade)
$ sudo apt install nginx                # install with dependencies
$ sudo apt install nginx=1.24.0-2ubuntu7   # a specific version
$ sudo apt install --reinstall nginx
$ sudo apt remove nginx                 # remove, keep config files
$ sudo apt purge nginx                  # remove + config files
$ sudo apt autoremove                   # remove orphaned dependencies
$ sudo apt clean                        # empty the download cache /var/cache/apt/archives
```

`-y` answers yes to prompts (`apt install -y tree`), useful in scripts.

### What `apt install` does

```console
$ sudo apt install tree
Reading package lists... Done
Building dependency tree... Done
The following NEW packages will be installed:
  tree
0 upgraded, 1 newly installed, 0 to remove and 3 not upgraded.
Need to get 47.1 kB of archives.
Get:1 http://archive.ubuntu.com/ubuntu noble/universe amd64 tree amd64 2.1.1-2 [47.1 kB]
Unpacking tree (2.1.1-2) ...
Setting up tree (2.1.1-2) ...
```

1. Reads metadata, computes dependencies.
2. Downloads `.deb` files into `/var/cache/apt/archives/`.
3. Calls `dpkg` to unpack and configure them (running maintainer scripts).

> On Debian/Ubuntu, installing a service package usually **starts and enables** it immediately (unlike RHEL).

---

## Searching and inspecting

```console
$ apt search "web server" | head          # search names and descriptions
$ apt show nginx                          # details of the candidate version
Package: nginx
Version: 1.24.0-2ubuntu7
Depends: nginx-common (= 1.24.0-2ubuntu7), libc6 (>= 2.34), libssl3t64 (>= 3.0.0) ...
Description: small, powerful, scalable web/proxy server

$ apt-cache policy nginx                  # installed vs candidate, and from which repo
nginx:
  Installed: (none)
  Candidate: 1.24.0-2ubuntu7.1
  Version table:
     1.24.0-2ubuntu7.1 500
        500 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 Packages
     1.24.0-2ubuntu7 500
        500 http://archive.ubuntu.com/ubuntu noble/main amd64 Packages

$ apt-cache depends nginx                 # dependencies
$ apt-cache rdepends libssl3t64 | head    # reverse: who depends on it
$ apt list --installed | grep ssh
```

---

## Querying installed packages with dpkg

| Command | Answers |
|---------|---------|
| `dpkg -l` | list installed packages (status, name, version) |
| `dpkg -l 'openssh*'` | list matching a pattern |
| `dpkg -s openssh-server` | status and metadata of an installed package |
| `dpkg -L openssh-server` | files installed by the package |
| `dpkg -S /usr/sbin/sshd` | which package owns this file |
| `dpkg -c file.deb` | contents of a `.deb` file (not installed) |
| `dpkg -I file.deb` | metadata of a `.deb` file |
| `dpkg --verify openssh-server` | check installed files against the package |
| `dpkg --get-selections` | list of all packages and their selection state (backup) |

```console
$ dpkg -S /usr/sbin/sshd
openssh-server: /usr/sbin/sshd

$ dpkg -L openssh-server | grep /etc/
/etc/ssh/moduli
/etc/ssh/sshd_config.d
/etc/pam.d/sshd
...

$ dpkg -l | head -6
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name           Version        Architecture Description
+++-==============-==============-============-=====================
ii  adduser        3.137ubuntu1   all          add and remove users and groups
```

The first two letters: `ii` = installed OK, `rc` = removed but config files remain, `hi` = held.

### Verifying files

```console
$ sudo dpkg --verify openssh-server
??5??????   /usr/sbin/sshd
```

The `5` means the MD5 checksum differs: the binary changed since installation. No output means everything matches. (`debsums -c` from the `debsums` package is an alternative.) Note that only some config files are checked.

---

## Which package provides a file I don't have?

`dpkg -S` only knows **installed** packages. To search all packages in the repositories:

```console
$ sudo apt install apt-file
$ sudo apt-file update
$ apt-file search bin/dig
bind9-dnsutils: /usr/bin/dig
$ sudo apt install bind9-dnsutils
```

When you type a missing command, Ubuntu's `command-not-found` helper also suggests the package.

---

## Repositories

### Configuration files

| Location | Format |
|----------|--------|
| `/etc/apt/sources.list` | classic one-line format (empty on Ubuntu 24.04) |
| `/etc/apt/sources.list.d/*.list` | classic one-line format |
| `/etc/apt/sources.list.d/*.sources` | **deb822** format (default since Ubuntu 24.04) |

Classic one-line format:

```text
deb [signed-by=/etc/apt/keyrings/example.gpg] https://repo.example.com/apt stable main
│    │                                         │                          │      └ components
│    │                                         │                          └ suite
│    │                                         └ URI
│    └ options
└ type (deb = binaries, deb-src = sources)
```

deb822 format (one key per line, easier to read and edit):

```text
# /etc/apt/sources.list.d/ubuntu.sources
Types: deb
URIs: http://archive.ubuntu.com/ubuntu/
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

Disable a deb822 repository by adding `Enabled: no`.

### Adding a third-party repository (the modern way)

```console
$ sudo install -d -m 0755 /etc/apt/keyrings
$ curl -fsSL https://repo.example.com/key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/example.gpg
$ sudo tee /etc/apt/sources.list.d/example.sources << 'EOF'
Types: deb
URIs: https://repo.example.com/apt
Suites: stable
Components: main
Signed-By: /etc/apt/keyrings/example.gpg
EOF
$ sudo apt update
```

`Signed-By` ties the key to **this repository only**. The old `apt-key add` trusted a key for *every* repository and is deprecated.

On Ubuntu, components and PPAs can also be managed with `add-apt-repository` (e.g. `sudo add-apt-repository universe`).

---

## Holding and pinning versions

```console
$ sudo apt-mark hold openssh-server       # never upgrade it
$ apt-mark showhold
openssh-server
$ sudo apt-mark unhold openssh-server
$ sudo apt-mark auto libfoo               # mark as automatically installed (autoremove candidate)
```

**Pinning** (`/etc/apt/preferences.d/`) sets priorities between repositories, e.g. to prefer a vendor repo for one package:

```text
Package: nginx
Pin: origin nginx.org
Pin-Priority: 900
```

---

## Local .deb files

```console
$ sudo apt install ./google-chrome-stable_current_amd64.deb   # resolves dependencies (note the ./)
$ sudo dpkg -i package.deb && sudo apt -f install             # alternative: fix missing deps afterwards
$ apt download tree                                           # fetch a .deb without installing
```

---

## Logs and troubleshooting

| Need | Look at |
|------|---------|
| What did apt install and when? | `/var/log/apt/history.log` |
| Detailed dpkg actions | `/var/log/dpkg.log` |
| "Could not get lock /var/lib/dpkg/lock-frontend" | another apt/dpkg (often `unattended-upgrades`) is running – wait, check `ps aux \| grep -E 'apt|dpkg'` |
| Interrupted install, broken state | `sudo dpkg --configure -a` then `sudo apt -f install` |
| "Unable to locate package" | `apt update`, check component (`universe`) and spelling |
| Signature / NO_PUBKEY errors | missing key or wrong `Signed-By` path |

Automatic security updates: package `unattended-upgrades`, configured in `/etc/apt/apt.conf.d/50unattended-upgrades`.

---

## Try it yourself

1. Install `tree`, find all files it installed, and the package owning `/usr/bin/tree`.
2. Find which package provides `/usr/bin/dig` using `apt-file`, and install it.
3. Check the candidate version and repository of `openssh-server` with `apt-cache policy`.
4. Hold `openssh-server`, run `apt upgrade` and confirm it is kept back; unhold it.
5. Remove `tree`, then look at `dpkg -l tree`; purge it and look again.

---

## Summary

| Task | Command |
|------|---------|
| Refresh metadata | `apt update` |
| Install / remove / purge | `apt install`, `apt remove`, `apt purge` |
| Upgrade | `apt upgrade`, `apt full-upgrade` |
| Search / show | `apt search`, `apt show`, `apt-cache policy` |
| Installed files / owner | `dpkg -L pkg`, `dpkg -S /path` |
| Provider of a missing file | `apt-file search path` |
| Verify | `dpkg --verify pkg` |
| Hold | `apt-mark hold pkg` |
| Repos | `/etc/apt/sources.list.d/*.sources` + key in `/etc/apt/keyrings/` |
| Logs | `/var/log/apt/history.log`, `/var/log/dpkg.log` |

## Check yourself

1. What is the difference between `apt upgrade` and `apt full-upgrade`?
2. `dpkg -l nginx` shows `rc`. What does it mean?
3. Which command shows which repository a package's candidate version comes from?
4. Why is `Signed-By` better than `apt-key add`?

<details>
<summary>Answers</summary>

1. `upgrade` never removes packages; `full-upgrade` may remove or install packages to complete upgrades.
2. Removed, but its configuration files are still present (purge them with `apt purge nginx`).
3. `apt-cache policy <pkg>`.
4. It limits trust of a key to one repository instead of all.

</details>

## Further reading

- `man 8 apt`, `man 1 dpkg`, `man 5 sources.list`, `man 8 apt-mark`, `man 5 apt_preferences`
- [Ubuntu Server: Package management](https://documentation.ubuntu.com/server/how-to/software/package-management/)
- [Debian Administrator's Handbook – Ch. 6 APT](https://debian-handbook.info/browse/stable/apt.html)
- [Debian Wiki: SourcesList](https://wiki.debian.org/SourcesList)

---

<!-- footer:start -->
[← Package Management Concepts](01-package-management-concepts.md) · [Module index](README.md) · [RPM Packages: rpm and dnf →](03-rpm-dnf.md)  
**Practice now:** [Exercises – Debian Packages: dpkg and APT](../../exercises/03-packages-git/02-dpkg-apt.md)
<!-- footer:end -->
