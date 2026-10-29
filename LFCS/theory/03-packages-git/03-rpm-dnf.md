# RPM Packages: rpm and dnf

<!-- nav:start -->
> **Module 03 · Package Management & Git** — lesson 3 of 5  
> [Module index](README.md) · [Course lesson (commands)](../../course/03-Packages-and-Git.md#lesson-3---rpm-dnfyum-ch-1011) · [Exercises for this lesson](../../exercises/03-packages-git/03-rpm-dnf.md)
<!-- nav:end -->

Red Hat Enterprise Linux, Rocky Linux, AlmaLinux, CentOS Stream and Fedora use **RPM** packages, managed by the low-level tool **rpm** and the high-level tool **dnf** (`yum` is now an alias to dnf). In this lesson you will learn to install, query, verify and roll back packages, manage repositories, and use package groups.

## What you will learn

- Installing, upgrading and removing with `dnf`
- Searching, getting information and finding providers (`dnf provides`)
- Querying the RPM database (`rpm -q` family)
- Verifying installed files (`rpm -V`) and importing keys
- Repository configuration (`/etc/yum.repos.d/*.repo`), EPEL and CRB
- History and undo; package groups; modules
- Logs and troubleshooting

---

## Key terms

| Term | Definition |
|------|-----------|
| **RPM** | Both the package format (`.rpm`) and the low-level tool `rpm`. Database in `/var/lib/rpm/`. |
| **dnf** | *Dandified YUM*: high-level tool resolving dependencies from repositories. `yum` is a compatibility alias. |
| **Repository ID** | Short name of a repo in brackets in a `.repo` file, e.g. `[appstream]`. |
| **BaseOS / AppStream** | RHEL 9 core OS packages / applications, languages and databases. |
| **CRB** | *CodeReady Builder*: extra development packages, needed by many EPEL packages. |
| **EPEL** | *Extra Packages for Enterprise Linux*: community repository maintained by Fedora. |
| **Package group** | A named set of packages installed together ("Development Tools"). |
| **Transaction** | One dnf operation (install/upgrade/remove); recorded in history and can be undone. |

---

## Everyday dnf commands

```console
$ sudo dnf install httpd                # install with dependencies
$ sudo dnf install httpd-2.4.57-8.el9   # specific version
$ sudo dnf reinstall httpd
$ sudo dnf remove httpd                 # also removes dependent packages – read the list!
$ sudo dnf upgrade                      # upgrade everything (also: dnf update)
$ sudo dnf upgrade openssh-server       # one package
$ dnf check-update                      # list available updates (exit code 100 if any)
$ sudo dnf autoremove                   # remove orphaned dependencies
$ sudo dnf clean all                    # drop cached metadata and packages
$ sudo dnf makecache                    # download metadata now
```

`-y` assumes yes, `--refresh` forces a metadata refresh, `--downloadonly` just downloads.

> On RHEL-family systems, installing a service **does not start or enable it**. Always follow with `systemctl enable --now <service>` (and open the firewall).

---

## Searching and inspecting

```console
$ dnf search nginx                 # names and summaries
$ dnf info nginx                   # version, repo, size, description
$ dnf list installed | grep ssh
$ dnf list available 'php*'
$ dnf repoquery --requires nginx   # dependencies
$ dnf repoquery -l nginx           # files of a package (even if not installed)
```

### Which package provides a file or command?

```console
$ dnf provides /usr/bin/dig
bind-utils-32:9.16.23-24.el9_5.x86_64 : Utilities for querying DNS name servers
Repo        : appstream
Matched from:
Filename    : /usr/bin/dig

$ dnf provides '*/semanage'        # wildcard when you don't know the full path
policycoreutils-python-utils-3.6-2.1.el9.noarch : SELinux policy core python utilities
```

This works for packages that are **not installed** – it searches repository metadata.

---

## Querying the RPM database

`rpm -q` (query) options are combined: `-a` all, `-i` info, `-l` list files, `-c` config files, `-d` docs, `-f` file owner, `-p` query a package **file** instead of the database, `-R` requires.

| Command | Answers |
|---------|---------|
| `rpm -qa` | all installed packages |
| `rpm -qa \| grep -i ssh` | installed packages matching |
| `rpm -q openssh-server` | is it installed? which version? |
| `rpm -qi openssh-server` | detailed info |
| `rpm -ql openssh-server` | files of the package |
| `rpm -qc openssh-server` | **configuration** files |
| `rpm -qd openssh-server` | documentation files |
| `rpm -qf /usr/sbin/sshd` | which package owns this file |
| `rpm -qR openssh-server` | requirements |
| `rpm -qlp file.rpm` | files inside a package file |
| `rpm -q --scripts openssh-server` | install/remove scriptlets |
| `rpm -q --changelog openssh-server \| head` | changelog (e.g. CVE fixes) |

```console
$ rpm -qf /etc/ssh/sshd_config
openssh-server-8.7p1-43.el9.x86_64
$ rpm -qc openssh-server
/etc/pam.d/sshd
/etc/ssh/sshd_config
/etc/sysconfig/sshd
```

Custom output with `--queryformat`:

```console
$ rpm -qa --qf '%{SIZE}\t%{NAME}\n' | sort -rn | head -3     # biggest packages
```

---

## Installing package files with rpm

```console
$ sudo rpm -ivh package.rpm     # install  (-v verbose, -h progress hashes)
$ sudo rpm -Uvh package.rpm     # upgrade or install
$ sudo rpm -Fvh package.rpm     # upgrade only if already installed (freshen)
$ sudo rpm -e package           # erase (remove)
```

`rpm` does **not** resolve dependencies. Prefer `sudo dnf install ./package.rpm`, which does.

---

## Verifying installed packages

```console
$ sudo rpm -V openssh-server
S.5....T.  c /etc/ssh/sshd_config
```

| Char | Meaning |
|------|---------|
| `S` | size differs |
| `M` | mode (permissions or type) differs |
| `5` | checksum differs (content changed) |
| `D` | device major/minor differs |
| `L` | symlink target differs |
| `U` / `G` | owner / group differs |
| `T` | modification time differs |
| `.` | test passed |
| `c` | the file is a config file |

`rpm -Va` verifies **every** package – useful after a suspected intrusion or a botched manual change. Repairing a modified binary: `dnf reinstall <pkg>`.

### Signatures and keys

```console
$ rpm -K package.rpm                      # check the signature of a file
package.rpm: digests signatures OK
$ sudo rpm --import https://repo.example.com/RPM-GPG-KEY-example
$ rpm -qa gpg-pubkey*                     # imported keys
```

---

## Repositories

Each file in `/etc/yum.repos.d/` may define several repositories:

```ini
# /etc/yum.repos.d/example.repo
[example]
name=Example Repository for EL9
baseurl=https://repo.example.com/el9/$basearch/
enabled=1
gpgcheck=1
gpgkey=https://repo.example.com/RPM-GPG-KEY-example

[example-debug]
name=Example debug packages
baseurl=https://repo.example.com/el9/debug/
enabled=0
```

| Key | Meaning |
|-----|---------|
| `[id]` | repository ID used on the command line |
| `baseurl` / `mirrorlist` / `metalink` | where packages are; `file:///mnt/iso` for a local DVD |
| `enabled` | 1 / 0 |
| `gpgcheck` | verify package signatures (keep 1) |
| `gpgkey` | URL or `file://` path of the key |
| `$basearch`, `$releasever` | variables filled in by dnf |

```console
$ dnf repolist                               # enabled repos
$ dnf repolist all                           # including disabled
$ dnf repoinfo appstream
$ sudo dnf config-manager --add-repo https://repo.example.com/example.repo   # from dnf-plugins-core
$ sudo dnf config-manager --set-enabled crb
$ sudo dnf config-manager --set-disabled example
$ sudo dnf --enablerepo=example-debug install foo     # enable for one command
```

### EPEL

```console
$ sudo dnf config-manager --set-enabled crb          # Rocky/Alma (RHEL: subscription-manager repos --enable codeready-builder-...)
$ sudo dnf install epel-release
$ dnf repolist
repo id        repo name
appstream      Rocky Linux 9 - AppStream
baseos         Rocky Linux 9 - BaseOS
crb            Rocky Linux 9 - CRB
epel           Extra Packages for Enterprise Linux 9 - x86_64
extras         Rocky Linux 9 - Extras
```

### A local repository from an ISO

```console
$ sudo mount -o loop /root/Rocky-9.iso /mnt/iso
$ sudo tee /etc/yum.repos.d/local.repo << 'EOF'
[local-baseos]
name=Local BaseOS
baseurl=file:///mnt/iso/BaseOS
gpgcheck=0
enabled=1
EOF
```

---

## History and rollback

Every transaction is recorded:

```console
$ sudo dnf history
ID     | Command line             | Date and time    | Action(s)      | Altered
-------------------------------------------------------------------------------
    12 | install httpd            | 2026-10-21 10:12 | Install        |    9
    11 | upgrade                  | 2026-10-20 08:00 | Upgrade        |   23
$ sudo dnf history info 12            # what exactly changed
$ sudo dnf history undo 12            # revert that transaction
$ sudo dnf history rollback 11        # go back to the state after transaction 11
```

---

## Groups and modules

```console
$ dnf group list
$ dnf group info "Development Tools"
$ sudo dnf group install "Development Tools"
$ sudo dnf group remove "Development Tools"
```

**Modules** (RHEL 8/9 AppStream) offer several versions (*streams*) of software. They're being phased out in RHEL 10 but still appear on RHEL 9:

```console
$ dnf module list nodejs
$ sudo dnf module enable nodejs:20
$ sudo dnf install nodejs
```

---

## Version locking

```console
$ sudo dnf install 'dnf-command(versionlock)'    # python3-dnf-plugin-versionlock
$ sudo dnf versionlock add openssh-server
$ dnf versionlock list
$ sudo dnf versionlock delete openssh-server
```

Or exclude packages from updates in a repo or in `/etc/dnf/dnf.conf`: `exclude=kernel*`.

---

## Logs and troubleshooting

| Need | Look at |
|------|---------|
| dnf actions | `/var/log/dnf.log`, `/var/log/dnf.rpm.log`, `dnf history` |
| "No match for argument" | wrong name; `dnf search`; repo disabled (`repolist all`); needs EPEL/CRB |
| GPG key errors | `rpm --import` the vendor key; check `gpgkey=` URL |
| Metadata errors | `dnf clean all && dnf makecache`; check network / proxy (`proxy=` in `dnf.conf`) |
| Corrupted RPM DB | `rpm --rebuilddb` |
| Config file changed by update | look for `*.rpmnew` / `*.rpmsave` |

Automatic updates: package `dnf-automatic`, configured in `/etc/dnf/automatic.conf`, enabled with `systemctl enable --now dnf-automatic.timer`.

---

## Try it yourself

1. Find which package provides `/usr/sbin/semanage` and install it.
2. List the configuration files and documentation of `chrony`.
3. Modify `/etc/chrony.conf`, then detect the change with `rpm -V chrony`.
4. Install `tree`, look at `dnf history`, then undo that transaction.
5. Add a repository file for a fake repo with `enabled=0`; confirm it appears only in `dnf repolist all`.

---

## Summary

| Task | Command |
|------|---------|
| Install / remove / upgrade | `dnf install`, `dnf remove`, `dnf upgrade` |
| Search / info | `dnf search`, `dnf info` |
| Provider of a file | `dnf provides /path` |
| Owner of an installed file | `rpm -qf /path` |
| Files / config / docs | `rpm -ql`, `rpm -qc`, `rpm -qd` |
| Verify | `rpm -V pkg`, `rpm -Va` |
| Repos | `/etc/yum.repos.d/*.repo`, `dnf repolist all`, `dnf config-manager` |
| History / undo | `dnf history`, `dnf history undo N` |
| Groups | `dnf group install "Name"` |

## Check yourself

1. What is the difference between `rpm -qf` and `dnf provides`?
2. What does `S.5....T.  c` mean in `rpm -V` output?
3. How do you enable a disabled repo for a single command?
4. After `dnf install httpd`, the web server is not reachable. Name two things you probably forgot.

<details>
<summary>Answers</summary>

1. `rpm -qf` queries the local database (installed packages only); `dnf provides` searches repository metadata, including packages not installed.
2. A config file whose size, checksum and modification time differ from the packaged version.
3. `dnf --enablerepo=<id> install …`.
4. `systemctl enable --now httpd` and opening the firewall (`firewall-cmd --add-service=http --permanent; firewall-cmd --reload`).

</details>

## Further reading

- `man 8 dnf`, `man 8 rpm`, `man 5 dnf.conf`, `man 5 yum.conf`
- [Red Hat: Managing software with the DNF tool (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_software_with_the_dnf_tool/index)
- [DNF command reference](https://dnf.readthedocs.io/en/latest/command_ref.html)
- [Rocky Linux docs: Software management](https://docs.rockylinux.org/books/admin_guide/13-softwares/)
- [EPEL documentation](https://docs.fedoraproject.org/en-US/epel/)

---

<!-- footer:start -->
[← Debian Packages: dpkg and APT](02-dpkg-apt.md) · [Module index](README.md) · [SUSE Packages: zypper →](04-zypper.md)  
**Practice now:** [Exercises – RPM Packages: rpm and dnf](../../exercises/03-packages-git/03-rpm-dnf.md)
<!-- footer:end -->
