# LFCS - Resources & Links

> Curated, not exhaustive. Prefer **official docs** and **man pages** - they're what you'll have in the exam. Topic-specific links are at the end of each [theory note](../theory/00-Theory-Index.md).

---

# 1. Official Linux Foundation

| Resource | Link |
|----------|------|
| LFCS exam page (domains, competencies) | https://training.linuxfoundation.org/certification/linux-foundation-certified-sysadmin-lfcs/ |
| LFS207 course | https://training.linuxfoundation.org/training/linux-system-administration-essentials-lfs207/ |
| Free: Introduction to Linux (LFS101) | https://training.linuxfoundation.org/training/introduction-to-linux/ |
| Candidate Handbook | https://docs.linuxfoundation.org/tc-docs/certification/lf-handbook2 |
| LFCS/LFCE important instructions | https://docs.linuxfoundation.org/tc-docs/certification/instructions-lfcs-and-lfce |
| Exam simulator (2 sessions included) | https://killer.sh/ |

---

# 2. Distribution Documentation

| Distro | Link | Best for |
|--------|------|----------|
| Red Hat Enterprise Linux 9 | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9 | SELinux, firewalld, nmcli, LVM, storage, virtualization |
| Rocky Linux | https://docs.rockylinux.org/ | free RHEL-compatible guides and labs |
| Ubuntu Server | https://documentation.ubuntu.com/server/ | netplan, AppArmor, ufw, apt |
| Debian Administrator's Handbook (free book) | https://debian-handbook.info/browse/stable/ | apt, networking, services in depth |
| Fedora Docs | https://docs.fedoraproject.org/en-US/docs/ | latest systemd/NetworkManager behaviour |
| ArchWiki | https://wiki.archlinux.org/ | the best general Linux reference, distro-neutral explanations |

---

# 3. Reference Documentation

| Topic | Link |
|-------|------|
| Linux man pages online | https://man7.org/linux/man-pages/ |
| systemd man pages | https://www.freedesktop.org/software/systemd/man/latest/ |
| systemd project (design docs) | https://systemd.io/ |
| Linux kernel docs | https://docs.kernel.org/ |
| GNU Bash manual | https://www.gnu.org/software/bash/manual/ |
| GNU Coreutils | https://www.gnu.org/software/coreutils/manual/ |
| Filesystem Hierarchy Standard | https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html |
| The Linux Documentation Project (older but deep) | https://tldp.org/ |

---

# 4. Books

| Book | Why |
|------|-----|
| *The Linux Command Line* - W. Shotts ([free PDF](https://linuxcommand.org/tlcl.php)) | best shell foundation |
| *How Linux Works, 3rd ed.* - B. Ward (No Starch) | explains the internals behind every theory note |
| *UNIX and Linux System Administration Handbook, 5th ed.* - Nemeth et al. | the classic sysadmin reference |
| *Linux Bible* - C. Negus (Wiley) | broad, practical, covers both families |
| *Pro Git* ([free](https://git-scm.com/book/en/v2)) | Module 03 Git |
| *SELinux System Administration* - S. Vermeulen (Packt) | Module 08 deep dive |
| *Systems Performance, 2nd ed.* - B. Gregg | Module 04 performance, beyond the exam |

---

# 5. Video Courses & Free Learning

| Resource | Link |
|----------|------|
| Linux Journey (free, beginner-friendly) | https://linuxjourney.com/ |
| Linux Upskill Challenge (free month-long server course) | https://linuxupskillchallenge.org/ |
| Red Hat free RH124 overview / developer sandbox | https://www.redhat.com/en/services/training/rh124-red-hat-system-administration-i |
| KodeKloud LFCS course & labs (paid) | https://kodekloud.com/ |
| Learn Linux TV (YouTube) | https://www.youtube.com/@LearnLinuxTV |

---

# 6. Practice & Hands-On

| Resource | What |
|----------|------|
| [OverTheWire: Bandit](https://overthewire.org/wargames/bandit/) | shell skills as a game |
| [Killercoda](https://killercoda.com/) | free in-browser Linux scenarios (Ubuntu) |
| [SadServers](https://sadservers.com/) | "fix this broken server" troubleshooting challenges - excellent exam prep |
| [Learn Git Branching](https://learngitbranching.js.org/) | visual Git practice |
| [Subnetting Practice](https://subnettingpractice.com/) | subnet drills |
| [Vagrant boxes](https://portal.cloud.hashicorp.com/vagrant/discover) | spin up the lab VMs reproducibly (`generic/ubuntu2404`, `rockylinux/9`) |

---

# 7. Quick Tools

| Tool | Use |
|------|-----|
| [explainshell.com](https://explainshell.com/) | explains each part of a command line |
| [tldr pages](https://tldr.sh/) | example-first mini man pages (`tldr tar`) |
| [crontab.guru](https://crontab.guru/) | cron expressions |
| [regex101](https://regex101.com/) | test regular expressions |
| [ShellCheck](https://www.shellcheck.net/) | lint bash scripts |
| [Mozilla SSL Config Generator](https://ssl-config.mozilla.org/) | sane TLS configs for nginx |
| `systemd-analyze calendar '...'` | test timer expressions locally (no internet needed) |

---

# 8. Communities

- [r/linuxadmin](https://www.reddit.com/r/linuxadmin/) · [r/linuxquestions](https://www.reddit.com/r/linuxquestions/)
- [Unix & Linux Stack Exchange](https://unix.stackexchange.com/) · [Server Fault](https://serverfault.com/)
- [Linux Foundation community forums](https://forum.linuxfoundation.org/)

---

# 9. Inside the Exam

No internet - only what's on the machine. Practice finding answers with:

```bash
man -k <keyword>                 # which man page?
man 5 <config-file>              # file format
/usr/share/doc/<package>/        # examples, sample configs
<cmd> --help | grep -i <word>
systemctl cat <unit>             # see a working example unit
nmcli --help; nmcli con mod --help
ls /usr/share/doc/netplan*/examples 2>/dev/null   # netplan examples (Ubuntu)
```
