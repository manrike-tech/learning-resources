# LFCS Course - Index

> Hands-on course for the **LFCS** exam, structured on the official **LFS207 – Linux System Administration Essentials** outline (34 chapters) and extended with the exam competencies LFS207 does not cover.
>
> This is a **study companion**, not a copy of LFS207: explanations, commands and labs are original. Use it alongside the official course material.

---

# Lab Setup (do this first)

| VM | Purpose | Resources |
|----|---------|-----------|
| `deb1` | Ubuntu Server LTS | 2 vCPU · 2 GB RAM · 20 GB disk + **2 extra 5 GB disks** |
| `rhel1` | Rocky Linux / AlmaLinux / Fedora | 2 vCPU · 2 GB RAM · 20 GB disk + **2 extra 5 GB disks** |

- Hypervisor: KVM/virt-manager, VirtualBox or VMware.
- Put both VMs on the **same internal network** (needed for SSH, NFS, NBD, routing and proxy labs).
- Take a **snapshot** after installation; revert whenever you break something.
- The exam is distribution-agnostic: practice every lab on **both** VMs.

---

# Modules

| # | Module (commands & lab) | Theory | Exercises | LFS207 chapters | Exam domain(s) |
|---|-------------------------|--------|-----------|-----------------|----------------|
| 01 | [Foundations & Essential Commands](01-Foundations.md) | [Theory](../theory/01-foundations/README.md) | [Exercises](../exercises/01-foundations/README.md) | 1, 2, 3 | Essential Commands |
| 02 | [Users, Groups, Permissions & LDAP](02-Users-Groups-Permissions.md) | [Theory](../theory/02-users-groups-permissions/README.md) | [Exercises](../exercises/02-users-groups-permissions/README.md) | 4, 5, 6, 29 | Users & Groups |
| 03 | [Package Management & Git](03-Packages-and-Git.md) | [Theory](../theory/03-packages-git/README.md) | [Exercises](../exercises/03-packages-git/README.md) | 7–13 | Ops & Deployment · Essential Commands |
| 04 | [Processes, Services, Scheduling & Performance](04-Processes-Services-Performance.md) | [Theory](../theory/04-processes-services/README.md) | [Exercises](../exercises/04-processes-services/README.md) | 14–17, 31 | Ops & Deployment · Essential Commands |
| 05 | [Storage](05-Storage.md) | [Theory](../theory/05-storage/README.md) | [Exercises](../exercises/05-storage/README.md) | 16 (swap), 19–23 + gaps | Storage |
| 06 | [Kernel, Modules, Devices](06-Kernel-Modules-Devices.md) | [Theory](../theory/06-kernel-devices/README.md) | [Exercises](../exercises/06-kernel-devices/README.md) | 24, 25, 26 | Ops & Deployment |
| 07 | [Networking](07-Networking.md) | [Theory](../theory/07-networking/README.md) | [Exercises](../exercises/07-networking/README.md) | 27, 28, 30 + gaps | Networking |
| 08 | [Security, Virtualization & Containers](08-Security-VMs-Containers.md) | [Theory](../theory/08-security-vms-containers/README.md) | [Exercises](../exercises/08-security-vms-containers/README.md) | 18, 33 + gaps | Ops & Deployment · Essential Commands |
| 09 | [Backup, Recovery & Rescue](09-Backup-Recovery-Rescue.md) | [Theory](../theory/09-backup-rescue/README.md) | [Exercises](../exercises/09-backup-rescue/README.md) | 32, 34 | Ops & Deployment |
| 10 | [Exam Preparation & Mock Exam](10-Exam-Prep.md) | — | Mock exam inside | — | All |

---

# LFS207 Chapter → Module Map

| Ch. | Title | Module |
|-----|-------|--------|
| 1 | Introduction | 01 |
| 2 | Linux Filesystem Tree Layout | 01 |
| 3 | User Environment | 01 |
| 4 | User Account Management | 02 |
| 5 | Group Management | 02 |
| 6 | File Permissions and Ownership | 02 |
| 7 | Package Management Systems | 03 |
| 8 | dpkg | 03 |
| 9 | APT | 03 |
| 10 | RPM | 03 |
| 11 | dnf and yum | 03 |
| 12 | zypper | 03 |
| 13 | GIT Fundamentals | 03 |
| 14 | Processes | 04 |
| 15 | Process Monitoring | 04 |
| 16 | Memory Monitoring, Usage and Configuring Swap | 04 / 05 |
| 17 | I/O Monitoring | 04 |
| 18 | Containers Overview | 08 |
| 19 | Linux Filesystems and the VFS | 05 |
| 20 | Disk Partitioning | 05 |
| 21 | Filesystem Features: Attributes, Creating, Checking, Usage, Mounting | 05 |
| 22 | The EXT4 Filesystem | 05 |
| 23 | Logical Volume Management (LVM) | 05 |
| 24 | Kernel Services and Configuration | 06 |
| 25 | Kernel Modules | 06 |
| 26 | Devices and udev | 06 |
| 27 | Network Addresses | 07 |
| 28 | Network Devices and Configuration | 07 |
| 29 | LDAP | 02 |
| 30 | Firewalls | 07 |
| 31 | System Init: systemd History and Customization | 04 |
| 32 | Backup and Recovery Methods | 09 |
| 33 | Linux Security Modules | 08 |
| 34 | System Rescue | 09 |

---

# Exam Competencies NOT in the LFS207 Outline (covered here)

| Competency | Module |
|------------|--------|
| Manage or schedule jobs (cron, at, timers) | 04 |
| Use remote filesystems and network block devices | 05 |
| Configure filesystem automounters | 05 |
| Configure user resource limits | 02 |
| Configure and manage ACLs | 02 |
| Set and synchronize system time | 07 |
| Configure the OpenSSH server and client | 07 |
| Configure static routing | 07 |
| Configure bridge and bonding devices | 07 |
| Implement reverse proxies and load balancers | 07 |
| Work with SSL certificates | 08 |
| Manage Virtual Machines (libvirt) | 08 |
| Configure container engines, create and manage containers (beyond overview) | 08 |

---

# How to Use Each Module

1. Read the matching LFS207 chapters.
2. For each lesson: read its **theory** page (linked under every lesson heading) – definitions, how it works, worked examples.
3. Work through the lesson's commands on your lab VMs, then do its **exercises** (★ → ★★★, with hidden solutions).
4. Complete the **Lab** without looking at the solution.
5. Answer the module **Self-check** and each theory page's *Check yourself* questions — if you can't do a task from memory, repeat it.
6. Tick the checklist in the roadmap.

---

# Theory Notes & Study Aids

| Document | Purpose |
|----------|---------|
| [Theory Index](../theory/00-Theory-Index.md) | One tutorial-style theory page per lesson (59): definitions, how it works, worked examples, pitfalls, self-check |
| [Exercises Index](../exercises/00-Exercises-Index.md) | One exercise file per lesson (59, ~460 exercises) with verification steps and hidden solutions |
| [Resources & Links](../resources/Resources-and-Links.md) | Official docs, books, practice sites, tools |
| [Glossary](../resources/Glossary.md) | Definitions of every key term |
| [Debian vs RHEL Differences](../resources/Distro-Differences.md) | Paths, packages and tools per distro family |
| [Troubleshooting Methodology](../resources/Troubleshooting-Methodology.md) | Repeatable process + symptom → cause table |
| [Persistence Checklist](../resources/Persistence-Checklist.md) | Runtime vs persistent config for every topic |

**Suggested pace:** 8–10 weeks at ~8 hours/week (LFS207 alone is 50–60 hours).
