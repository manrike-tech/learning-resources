# Linux Foundation Certified System Administrator (LFCS) - Study Roadmap

> **Objective:** Build the practical skills required to administer, configure, troubleshoot and secure Linux systems from the command line, and pass the performance-based **LFCS** exam, using the official **Linux System Administration Essentials (LFS207)** course as the backbone.

---

## Exam Domains

| Domain | Weight |
|--------|--------|
| Operations & Deployment | 25% |
| Networking | 25% |
| Storage | 20% |
| Essential Commands | 20% |
| Users & Groups | 10% |

> **Key insight:** LFS207 covers most of the exam, but **not all of it**. Phases marked **(Gap)** cover exam competencies that are absent or only briefly covered in the LFS207 outline and must be studied separately.

---

# Recommended Learning Path

## 1. Linux Foundations & Filesystem Layout

**LFS207:** Ch. 1 Introduction · Ch. 2 Linux Filesystem Tree Layout · Ch. 3 User Environment
**Exam domain:** Essential Commands

### Objectives

- Understand the Filesystem Hierarchy Standard (FHS): `/etc`, `/var`, `/usr`, `/proc`, `/sys`, `/dev`.
- Navigate, search and manipulate files efficiently (`find`, `grep`, `sed`, `awk`, `tar`).
- Use the shell environment: variables, `PATH`, aliases, history, redirection and pipes.
- Use `man`, `--help` and `/usr/share/doc` to find answers without the internet.

### Expected Outcome

Be fluent on the command line and able to locate any configuration file or log on an unfamiliar system.

---

## 2. Users, Groups & Permissions

**LFS207:** Ch. 4 User Account Management · Ch. 5 Group Management · Ch. 6 File Permissions and Ownership · Ch. 29 LDAP
**Exam domain:** Users & Groups (10%)

### Objectives

- Create, modify, lock and delete local users and groups.
- Manage password aging and `sudo` privileges.
- Apply standard permissions, special bits (SUID, SGID, sticky) and `umask`.
- Configure and manage **ACLs** (`setfacl`, `getfacl`).
- Manage personal and system-wide environment profiles.
- Configure user resource limits (`/etc/security/limits.conf`, `ulimit`).
- Configure the system to use **LDAP** user and group accounts (SSSD).

### Expected Outcome

Be able to implement a secure, least-privilege user model on a Linux server.

---

## 3. Software Package Management

**LFS207:** Ch. 7 Package Management Systems · Ch. 8 dpkg · Ch. 9 APT · Ch. 10 RPM · Ch. 11 dnf and yum · Ch. 12 zypper
**Exam domain:** Operations & Deployment

### Objectives

- Understand low-level (dpkg/rpm) vs high-level (apt/dnf/zypper) tools.
- Search, install, remove, upgrade and verify packages.
- Find which package owns a file and which package provides a command.
- Add, enable and disable repositories, and manage signing keys.

### Expected Outcome

Be able to maintain software on both Debian- and Red Hat-family systems.

---

## 4. Git Fundamentals

**LFS207:** Ch. 13 GIT Fundamentals
**Exam domain:** Essential Commands

### Objectives

- Initialize and clone repositories.
- Stage, commit, view history and diff changes.
- Create, switch and merge branches.
- Work with remotes: `fetch`, `pull`, `push`.

### Expected Outcome

Be able to perform basic Git operations required to manage configuration and scripts.

---

## 5. Processes, Services & Job Scheduling

**LFS207:** Ch. 14 Processes · Ch. 31 System Init: systemd History and Customization
**Exam domain:** Operations & Deployment · Essential Commands

### Objectives

- Understand process states, signals, priorities (`nice`, `renice`).
- Manage services with `systemctl` (start, enable, mask, status).
- Create and customize systemd units (drop-ins, `systemctl edit`).
- Read logs with `journalctl`.
- **(Gap)** Schedule jobs with `cron`, `at` and **systemd timers**.

### Expected Outcome

Be able to create, configure, schedule and troubleshoot services.

---

## 6. System Performance Monitoring

**LFS207:** Ch. 15 Process Monitoring · Ch. 16 Memory Monitoring, Usage and Configuring Swap · Ch. 17 I/O Monitoring
**Exam domain:** Essential Commands · Storage

### Objectives

- Monitor CPU, memory and load (`top`, `ps`, `free`, `vmstat`, `uptime`).
- Monitor storage performance (`iostat`, `iotop`).
- Configure and manage **swap** space.
- Troubleshoot disk space issues (`df`, `du`, deleted-but-open files with `lsof`).
- Determine application and service constraints (limits, ports, resources).

### Expected Outcome

Be able to diagnose why a system or service is slow, full or failing.

---

## 7. Filesystems, Partitions & LVM

**LFS207:** Ch. 19 Linux Filesystems and the VFS · Ch. 20 Disk Partitioning · Ch. 21 Filesystem Features · Ch. 22 The EXT4 Filesystem · Ch. 23 LVM
**Exam domain:** Storage (20%)

### Objectives

- Partition disks with `fdisk`, `gdisk`, `parted` (MBR and GPT).
- Create, check, repair, resize and mount filesystems (ext4, xfs).
- Configure persistent mounts in `/etc/fstab` using UUIDs.
- Create and manage LVM: PVs, VGs, LVs, extend and resize.

### Expected Outcome

Be able to provision, grow and repair local storage without data loss.

---

## 8. Remote Storage & Automounting (Gap)

**LFS207:** Not covered directly
**Exam domain:** Storage

### Objectives

- Export and mount **NFS** shares.
- Use network block devices (**NBD**) and understand iSCSI concepts.
- Configure filesystem automounters (**autofs**, systemd automount).

### Expected Outcome

Be able to consume and provide network storage that mounts on demand.

---

## 9. Kernel, Modules & Devices

**LFS207:** Ch. 24 Kernel Services and Configuration · Ch. 25 Kernel Modules · Ch. 26 Devices and udev
**Exam domain:** Operations & Deployment

### Objectives

- Configure kernel parameters with `sysctl`, persistent and non-persistent.
- Load, unload, configure and blacklist kernel modules.
- Understand udev and write simple udev rules.

### Expected Outcome

Be able to tune the kernel and manage hardware at runtime.

---

## 10. Networking

**LFS207:** Ch. 27 Network Addresses · Ch. 28 Network Devices and Configuration · Ch. 30 Firewalls
**Exam domain:** Networking (25%)

### Objectives

- Configure IPv4/IPv6 and hostname resolution (`ip`, `nmcli`, netplan, `/etc/hosts`, resolver).
- Monitor and troubleshoot networking (`ss`, `ping`, `tracepath`, `dig`, `tcpdump`).
- Configure packet filtering, **port redirection and NAT** (firewalld, ufw, nftables).
- **(Gap)** Configure static routing.
- **(Gap)** Configure **bridge** and **bonding** devices.
- **(Gap)** Configure the OpenSSH server and client (keys, `sshd_config`).
- **(Gap)** Set and synchronize time (chrony, `timedatectl`).
- **(Gap)** Implement **reverse proxies and load balancers** (Nginx).

### Expected Outcome

Be able to connect, route, protect and expose services on a Linux network.

---

## 11. Security: SELinux, AppArmor & SSL Certificates

**LFS207:** Ch. 33 Linux Security Modules
**Exam domain:** Operations & Deployment · Essential Commands

### Objectives

- Understand Mandatory Access Control (MAC).
- Create and enforce MAC using **SELinux**: modes, contexts, booleans, `restorecon`, `semanage`.
- Understand AppArmor profiles.
- **(Gap)** Work with **SSL certificates**: generate keys, CSRs and self-signed certs with `openssl`, inspect certificates.

### Expected Outcome

Be able to troubleshoot "permission denied" caused by MAC and secure services with TLS.

---

## 12. Virtualization & Containers

**LFS207:** Ch. 18 Containers Overview
**Exam domain:** Operations & Deployment

### Objectives

- **(Gap)** Manage virtual machines with **libvirt** (`virsh`, `virt-install`).
- Configure container engines (Docker/Podman).
- Create, run, inspect, stop and remove containers; manage images, ports and volumes.

### Expected Outcome

Be able to deploy applications as VMs or containers on a Linux host.

---

## 13. Backup, Recovery & System Rescue

**LFS207:** Ch. 32 Backup and Recovery Methods · Ch. 34 System Rescue
**Exam domain:** Operations & Deployment

### Objectives

- Back up and restore with `tar`, `rsync`, `dd`.
- Recover from hardware, OS or filesystem failures.
- Boot into rescue/emergency targets, repair GRUB, fix a broken `fstab`, reset a root password.

### Expected Outcome

Be able to bring a broken system back to a working state.

---

## 14. Exam Readiness

### Objectives

- Practice every competency on **two VMs** (Ubuntu + a RHEL-family distro).
- Complete both **Killer.sh** simulator sessions.
- Learn to verify every task (did it survive a reboot?).
- Train speed: 2 hours, many tasks.

### Expected Outcome

Pass the LFCS exam on the first attempt.

---

# Course Material in This Repository

| Document | Purpose |
|----------|---------|
| [Course Index](course/00-Course-Index.md) | Hands-on modules with lessons, labs and self-checks |
| [Theory Index](theory/00-Theory-Index.md) | One tutorial page per lesson: definitions, how it works, worked examples, pitfalls, self-check |
| [Exercises Index](exercises/00-Exercises-Index.md) | One exercise file per lesson, ★ to ★★★, with verification and hidden solutions |
| [Resources & Links](resources/Resources-and-Links.md) | Curated docs, books, practice sites and tools |
| [Glossary](resources/Glossary.md) · [Distro Differences](resources/Distro-Differences.md) · [Troubleshooting](resources/Troubleshooting-Methodology.md) · [Persistence Checklist](resources/Persistence-Checklist.md) | Study aids |

---

# Official Resources

## 1. LFCS Certification (domains, competencies, exam details)

https://training.linuxfoundation.org/certification/linux-foundation-certified-sysadmin-lfcs/

---

## 2. Linux System Administration Essentials (LFS207)

https://training.linuxfoundation.org/training/linux-system-administration-essentials-lfs207/

---

## 3. Official LFCS Learning Path (PDF)

https://training.linuxfoundation.org/wp-content/uploads/2024/10/LFCS.pdf

---

## 4. Free prerequisite course: Introduction to Linux (LFS101)

https://training.linuxfoundation.org/training/introduction-to-linux/

---

## 5. Candidate Handbook & LFCS Important Instructions

https://docs.linuxfoundation.org/tc-docs/certification/lf-handbook2

https://docs.linuxfoundation.org/tc-docs/certification/instructions-lfcs-and-lfce

---

## 6. Certification FAQ & Policies

https://training.linuxfoundation.org/about/faqs/certification-faq/

https://training.linuxfoundation.org/about/policies/

---

# Recommended Reading Order

- [ ] Linux Foundations & Filesystem Layout
- [ ] Users, Groups & Permissions
- [ ] Software Package Management
- [ ] Git Fundamentals
- [ ] Processes, Services & Job Scheduling
- [ ] System Performance Monitoring
- [ ] Filesystems, Partitions & LVM
- [ ] Remote Storage & Automounting
- [ ] Kernel, Modules & Devices
- [ ] Networking
- [ ] Security: SELinux, AppArmor & SSL Certificates
- [ ] Virtualization & Containers
- [ ] Backup, Recovery & System Rescue
- [ ] Exam Readiness

---

# Final Goal

Develop the hands-on skills required to install, configure, operate, secure and troubleshoot Linux systems on-premises or in the cloud, and earn the **Linux Foundation Certified System Administrator (LFCS)** certification as the foundation for future paths in Networking, DevOps and Kubernetes (CKA).
