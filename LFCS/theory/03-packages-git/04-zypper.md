# SUSE Packages: zypper

<!-- nav:start -->
> **Module 03 · Package Management & Git** — lesson 4 of 5  
> [Module index](README.md) · [Course lesson (commands)](../../course/03-Packages-and-Git.md#lesson-4---zypper-ch-12) · [Exercises for this lesson](../../exercises/03-packages-git/04-zypper.md)
<!-- nav:end -->

openSUSE and SUSE Linux Enterprise (SLES) use RPM packages like Red Hat, but their high-level tool is **zypper**. LFS207 covers it so that you can work on any major distribution. The good news: if you understand [package management concepts](01-package-management-concepts.md) and [rpm](03-rpm-dnf.md), zypper is mostly new command names.

## What you will learn

- zypper commands for install, remove, update and search
- Repository management (`repos`, `addrepo`, `refresh`)
- Finding which package provides a file
- Patches vs updates, locks, and the difference between `update` and `dist-upgrade`
- How zypper maps to apt and dnf

---

## Key terms

| Term | Definition |
|------|-----------|
| **zypper** | Command-line front-end for **libzypp**, SUSE's package management library. |
| **Repository alias** | The short name of a repo (like a dnf repo ID). |
| **Refresh** | Downloading fresh repository metadata (`zypper refresh`). |
| **Patch** | A SUSE update unit that can contain several packages, classified as security/recommended/optional. |
| **Pattern** | A group of packages for a purpose (like dnf groups), e.g. `lamp_server`. |
| **Lock** | Prevents a package from being installed/changed. |
| **dup** | *Distribution upgrade* – the way to move between releases and to update openSUSE Tumbleweed. |

---

## Everyday commands

Most commands have a short alias (shown in parentheses).

```console
$ sudo zypper refresh                     # (ref) refresh metadata
$ zypper search nginx                     # (se) search
$ zypper info nginx                       # (if) details
$ sudo zypper install nginx               # (in) install with dependencies
$ sudo zypper install ./package.rpm       # local file with dependency resolution
$ sudo zypper remove nginx                # (rm) remove
$ sudo zypper remove --clean-deps nginx   # also remove now-unneeded deps
$ zypper list-updates                     # (lu) what can be updated
$ sudo zypper update                      # (up) update installed packages
$ sudo zypper dist-upgrade                # (dup) distribution upgrade / Tumbleweed updates
$ zypper search --installed-only ssh      # (se -i)
```

`-n` / `--non-interactive` answers prompts automatically (`zypper -n in tree`) – note it goes **before** the command.

### Search output

```console
$ zypper se -s nginx
S | Name   | Type    | Version        | Arch   | Repository
--+--------+---------+----------------+--------+-----------
i | nginx  | package | 1.21.5-150400  | x86_64 | Main Repository
```

`i` in the first column means installed; `v` means another version is installed.

---

## Which package provides a file?

```console
$ zypper what-provides /usr/bin/dig       # (wp)
$ zypper se --provides /usr/bin/dig
S | Name      | Summary
--+-----------+-------------------------
  | bind-utils | Utilities to query and test DNS
$ rpm -qf /usr/bin/dig                    # installed files: same rpm queries as on RHEL
```

All `rpm -q…` and `rpm -V` commands from the [RPM lesson](03-rpm-dnf.md) work identically on SUSE.

---

## Repositories

```console
$ zypper repos                            # (lr) list
$ zypper lr -u                            # with URIs
$ sudo zypper addrepo https://download.opensuse.org/repositories/home:/someone/openSUSE_Leap_15.6/ myrepo   # (ar) URL alias
$ sudo zypper addrepo -f URL alias        # -f enable autorefresh
$ sudo zypper modifyrepo -d myrepo        # (mr) disable  (-e enable)
$ sudo zypper removerepo myrepo           # (rr)
$ sudo zypper refresh myrepo
```

Repository files live in `/etc/zypp/repos.d/*.repo` (same INI style as dnf). The first time you refresh a repo with a new key, zypper asks whether to trust it.

---

## Patches, patterns and locks

```console
$ zypper list-patches                     # (lp) available patches
$ sudo zypper patch                       # install all needed patches
$ zypper patterns                         # (pt)
$ sudo zypper install -t pattern lamp_server
$ sudo zypper addlock openssh             # (al) prevent changes
$ zypper locks                            # (ll)
$ sudo zypper removelock openssh          # (rl)
```

---

## Command map: apt vs dnf vs zypper

| Task | apt | dnf | zypper |
|------|-----|-----|--------|
| Refresh metadata | `apt update` | `dnf makecache` | `zypper refresh` |
| Install | `apt install` | `dnf install` | `zypper install` |
| Remove | `apt remove` | `dnf remove` | `zypper remove` |
| Upgrade all | `apt upgrade` | `dnf upgrade` | `zypper update` |
| Search | `apt search` | `dnf search` | `zypper search` |
| Info | `apt show` | `dnf info` | `zypper info` |
| Provider of a file | `apt-file search` | `dnf provides` | `zypper what-provides` |
| List repos | `apt policy` | `dnf repolist` | `zypper repos` |
| Add repo | file in `sources.list.d` | `dnf config-manager --add-repo` | `zypper addrepo` |
| Lock version | `apt-mark hold` | `dnf versionlock add` | `zypper addlock` |
| Groups | (tasks/metapackages) | `dnf group install` | `zypper install -t pattern` |
| History | `/var/log/apt/history.log` | `dnf history` | `/var/log/zypp/history` |

---

## Try it yourself

If you have an openSUSE VM or container (`podman run -it --rm registry.opensuse.org/opensuse/leap:15.6`):

1. Refresh repositories and list them with URIs.
2. Search for `tree`, install it non-interactively, and find its files with `rpm -ql`.
3. Find which package provides `/usr/bin/dig`.
4. Add a lock on `tree`, try to remove it, then remove the lock.

---

## Summary

- zypper = SUSE's high-level tool on top of rpm.
- Same concepts: refresh, install, remove, update, search, repos, locks.
- `rpm` queries work exactly as on RHEL.

## Check yourself

1. Which zypper command corresponds to `dnf provides`?
2. How do you add a repository with alias `tools` from a URL?
3. Where do zypper repo files live?

<details>
<summary>Answers</summary>

1. `zypper what-provides` (or `zypper se --provides`).
2. `zypper addrepo <URL> tools`.
3. `/etc/zypp/repos.d/`.

</details>

## Further reading

- `man 8 zypper`
- [openSUSE: Zypper usage](https://en.opensuse.org/SDB:Zypper_usage)
- [SUSE documentation: Managing software with command line tools](https://documentation.suse.com/sles/15-SP6/html/SLES-all/cha-sw-cl.html)

---

<!-- footer:start -->
[← RPM Packages: rpm and dnf](03-rpm-dnf.md) · [Module index](README.md) · [Git Fundamentals →](05-git-fundamentals.md)  
**Practice now:** [Exercises – SUSE Packages: zypper](../../exercises/03-packages-git/04-zypper.md)
<!-- footer:end -->
