# Module 03 - Package Management & Git

**LFS207:** Ch. 7 Package Management Systems · Ch. 8 dpkg · Ch. 9 APT · Ch. 10 RPM · Ch. 11 dnf and yum · Ch. 12 zypper · Ch. 13 GIT Fundamentals
**Exam domain:** Operations & Deployment · Essential Commands  
**Theory:** [Module 03 theory notes](../theory/03-packages-git/README.md) · **Exercises:** [Module 03 exercises](../exercises/03-packages-git/README.md)  

---

## Learning Objectives

- Use low-level and high-level package tools on each family.
- Query, verify and troubleshoot installed packages.
- Manage repositories.
- Perform basic Git operations.

---

## Lesson 1 - Package Management Concepts (Ch. 7)

> **Theory:** [Package Management Concepts](../theory/03-packages-git/01-package-management-concepts.md) · **Exercises:** [practice tasks](../exercises/03-packages-git/01-package-management-concepts.md)

| Layer | Debian/Ubuntu | RHEL/Fedora | openSUSE |
|-------|---------------|-------------|----------|
| Package format | `.deb` | `.rpm` | `.rpm` |
| Low-level (no dependency resolution) | `dpkg` | `rpm` | `rpm` |
| High-level (repos + dependencies) | `apt` | `dnf` (`yum`) | `zypper` |

Low-level tools install a single file; high-level tools fetch from repositories and resolve dependencies.

---

## Lesson 2 - dpkg & APT (Ch. 8–9)

> **Theory:** [Debian Packages: dpkg and APT](../theory/03-packages-git/02-dpkg-apt.md) · **Exercises:** [practice tasks](../exercises/03-packages-git/02-dpkg-apt.md)

```bash
dpkg -l | grep nginx            # list installed
dpkg -L nginx                   # files installed by package
dpkg -S /usr/sbin/nginx         # which package owns a file
dpkg -s nginx                   # status/info
dpkg -i pkg.deb; dpkg -r pkg; dpkg -P pkg   # install, remove, purge
dpkg --verify nginx             # check files against package

apt update                      # refresh indexes
apt upgrade; apt full-upgrade
apt install nginx; apt install ./local.deb
apt remove nginx; apt purge nginx; apt autoremove
apt search nginx; apt show nginx
apt-cache policy nginx          # candidate versions & repo
apt-mark hold nginx             # pin version
apt-file search bin/dig         # which package provides a file (apt install apt-file)
```

Repositories: `/etc/apt/sources.list`, `/etc/apt/sources.list.d/*.list` or `*.sources` (deb822 format). Keys in `/etc/apt/keyrings/`.

```
# /etc/apt/sources.list.d/example.sources
Types: deb
URIs: https://repo.example.com/apt
Suites: stable
Components: main
Signed-By: /etc/apt/keyrings/example.gpg
```

---

## Lesson 3 - RPM, dnf/yum (Ch. 10–11)

> **Theory:** [RPM Packages: rpm and dnf](../theory/03-packages-git/03-rpm-dnf.md) · **Exercises:** [practice tasks](../exercises/03-packages-git/03-rpm-dnf.md)

```bash
rpm -qa | grep httpd            # list installed
rpm -qi httpd                   # info
rpm -ql httpd                   # files
rpm -qf /usr/sbin/httpd         # owner of a file
rpm -qc httpd                   # config files
rpm -V httpd                    # verify (size, mode, md5…)
rpm -ivh pkg.rpm; rpm -e pkg
rpm --import KEY-URL

dnf install httpd; dnf remove httpd; dnf upgrade
dnf search nginx; dnf info nginx
dnf provides */dig              # which package provides a file
dnf list installed; dnf history; dnf history undo <id>
dnf repolist; dnf repolist all
dnf config-manager --add-repo URL          # dnf-plugins-core
dnf config-manager --set-enabled crb
dnf group list; dnf group install "Development Tools"
dnf reinstall pkg
```

Repo files: `/etc/yum.repos.d/*.repo`

```ini
[myrepo]
name=My Repo
baseurl=https://repo.example.com/el9/
enabled=1
gpgcheck=1
gpgkey=https://repo.example.com/RPM-GPG-KEY
```

---

## Lesson 4 - zypper (Ch. 12)

> **Theory:** [SUSE Packages: zypper](../theory/03-packages-git/04-zypper.md) · **Exercises:** [practice tasks](../exercises/03-packages-git/04-zypper.md)

```bash
zypper refresh; zypper install pkg; zypper remove pkg
zypper search pkg; zypper info pkg; zypper update
zypper repos; zypper addrepo URL alias; zypper removerepo alias
zypper what-provides /usr/bin/dig
```

---

## Lesson 5 - Git Fundamentals (Ch. 13)

> **Theory:** [Git Fundamentals](../theory/03-packages-git/05-git-fundamentals.md) · **Exercises:** [practice tasks](../exercises/03-packages-git/05-git-fundamentals.md)

```bash
git config --global user.name "Ana"
git config --global user.email ana@example.com

git init myrepo; cd myrepo
git clone https://github.com/user/repo.git
git status
git add file; git add .
git commit -m "message"
git log --oneline --graph
git diff; git diff --staged
git restore file                 # discard working changes
git restore --staged file        # unstage

git branch feature; git switch feature   # or: git checkout -b feature
git merge feature
git branch -d feature

git remote -v; git remote add origin URL
git fetch; git pull; git push -u origin main
git tag v1.0
git stash; git stash pop
```

> **Exam tip:** Git tasks are usually simple: clone, create branch, commit a file with a given message, merge, push to a local bare remote (`git init --bare /srv/repo.git`).

---

## Lab 03

1. On both VMs, find which package provides the `dig` command and install it.
2. Find which package owns `/etc/ssh/sshd_config` and list its config files.
3. Accidentally edit a file shipped by a package, then detect the change with `rpm -V` / `dpkg --verify`.
4. Hold the current version of `openssh-server` on Ubuntu.
5. Add a custom repo file (disabled) on `rhel1`; list all repos including disabled ones.
6. Create a bare repo `/srv/git/app.git`. Clone it, create branch `feature-x`, add `config.yml`, commit "Add config", merge into `main` and push.

## Self-check

- [ ] I can answer "which package owns/provides X?" on both families.
- [ ] I can add, enable and disable a repository.
- [ ] I can verify installed package integrity.
- [ ] I can clone, branch, commit, merge and push without notes.
