# LFCS Theory – Index

> One page per lesson. Each page is a self-contained tutorial: **what you will learn**, **key terms** with definitions, how the thing works (with diagrams), **worked examples with real output**, *try it yourself* steps, common mistakes, a summary table, *check yourself* questions with answers, and curated further reading.

The course modules in [`../course/`](../course/00-Course-Index.md) remain the compact *command reference and lab*; the theory pages explain the *why*, and the [exercises](../exercises/00-Exercises-Index.md) make you practise.

## How to study a lesson

1. Read the theory page (30–45 minutes). Type the examples on your lab VMs as you go.
2. Skim the matching **course lesson** (link at the top of each page) – it's the cheat sheet you'll come back to.
3. Do the **exercises** without opening the solutions. Every exercise has a *Verify* step.
4. Answer *Check yourself*. If you can't, re-read that section.
5. At the end of a module, do its **Lab** and **Self-check** in the course module.

## Lessons

### Module 01 – [Foundations & Essential Commands](01-foundations/README.md)

Course: [01-Foundations.md](../course/01-Foundations.md) · Exercises: [module 01](../exercises/01-foundations/README.md)

1. [The Linux Filesystem Hierarchy](01-foundations/01-filesystem-hierarchy.md) · [exercises](../exercises/01-foundations/01-filesystem-hierarchy.md)
2. [Finding Files](01-foundations/02-finding-files.md) · [exercises](../exercises/01-foundations/02-finding-files.md)
3. [Working with Text](01-foundations/03-working-with-text.md) · [exercises](../exercises/01-foundations/03-working-with-text.md)
4. [Redirection, Pipes and Archives](01-foundations/04-redirection-pipes-archives.md) · [exercises](../exercises/01-foundations/04-redirection-pipes-archives.md)
5. [The User Environment](01-foundations/05-user-environment.md) · [exercises](../exercises/01-foundations/05-user-environment.md)
6. [Finding Help Offline](01-foundations/06-finding-help.md) · [exercises](../exercises/01-foundations/06-finding-help.md)

### Module 02 – [Users, Groups, Permissions & LDAP](02-users-groups-permissions/README.md)

Course: [02-Users-Groups-Permissions.md](../course/02-Users-Groups-Permissions.md) · Exercises: [module 02](../exercises/02-users-groups-permissions/README.md)

1. [User Accounts](02-users-groups-permissions/01-user-accounts.md) · [exercises](../exercises/02-users-groups-permissions/01-user-accounts.md)
2. [Groups](02-users-groups-permissions/02-groups.md) · [exercises](../exercises/02-users-groups-permissions/02-groups.md)
3. [Privilege Escalation with sudo and su](02-users-groups-permissions/03-sudo.md) · [exercises](../exercises/02-users-groups-permissions/03-sudo.md)
4. [File Permissions and Ownership](02-users-groups-permissions/04-permissions-ownership.md) · [exercises](../exercises/02-users-groups-permissions/04-permissions-ownership.md)
5. [Access Control Lists (ACLs)](02-users-groups-permissions/05-acls.md) · [exercises](../exercises/02-users-groups-permissions/05-acls.md)
6. [Personal and System-Wide Environment Profiles](02-users-groups-permissions/06-environment-profiles.md) · [exercises](../exercises/02-users-groups-permissions/06-environment-profiles.md)
7. [User Resource Limits](02-users-groups-permissions/07-resource-limits.md) · [exercises](../exercises/02-users-groups-permissions/07-resource-limits.md)
8. [Using LDAP for Users and Groups](02-users-groups-permissions/08-ldap-clients.md) · [exercises](../exercises/02-users-groups-permissions/08-ldap-clients.md)

### Module 03 – [Package Management & Git](03-packages-git/README.md)

Course: [03-Packages-and-Git.md](../course/03-Packages-and-Git.md) · Exercises: [module 03](../exercises/03-packages-git/README.md)

1. [Package Management Concepts](03-packages-git/01-package-management-concepts.md) · [exercises](../exercises/03-packages-git/01-package-management-concepts.md)
2. [Debian Packages: dpkg and APT](03-packages-git/02-dpkg-apt.md) · [exercises](../exercises/03-packages-git/02-dpkg-apt.md)
3. [RPM Packages: rpm and dnf](03-packages-git/03-rpm-dnf.md) · [exercises](../exercises/03-packages-git/03-rpm-dnf.md)
4. [SUSE Packages: zypper](03-packages-git/04-zypper.md) · [exercises](../exercises/03-packages-git/04-zypper.md)
5. [Git Fundamentals](03-packages-git/05-git-fundamentals.md) · [exercises](../exercises/03-packages-git/05-git-fundamentals.md)

### Module 04 – [Processes, Services, Scheduling & Performance](04-processes-services/README.md)

Course: [04-Processes-Services-Performance.md](../course/04-Processes-Services-Performance.md) · Exercises: [module 04](../exercises/04-processes-services/README.md)

1. [Processes](04-processes-services/01-processes.md) · [exercises](../exercises/04-processes-services/01-processes.md)
2. [Monitoring System Performance](04-processes-services/02-monitoring.md) · [exercises](../exercises/04-processes-services/02-monitoring.md)
3. [Troubleshooting Disk Space](04-processes-services/03-disk-space.md) · [exercises](../exercises/04-processes-services/03-disk-space.md)
4. [Managing Services with systemd](04-processes-services/04-systemd.md) · [exercises](../exercises/04-processes-services/04-systemd.md)
5. [Application and Service Constraints](04-processes-services/05-service-constraints.md) · [exercises](../exercises/04-processes-services/05-service-constraints.md)
6. [Scheduling Jobs: cron, at and systemd Timers](04-processes-services/06-scheduling-jobs.md) · [exercises](../exercises/04-processes-services/06-scheduling-jobs.md)

### Module 05 – [Storage](05-storage/README.md)

Course: [05-Storage.md](../course/05-Storage.md) · Exercises: [module 05](../exercises/05-storage/README.md)

1. [The VFS and Filesystem Types](05-storage/01-vfs-filesystem-types.md) · [exercises](../exercises/05-storage/01-vfs-filesystem-types.md)
2. [Disk Partitioning](05-storage/02-partitioning.md) · [exercises](../exercises/05-storage/02-partitioning.md)
3. [Creating, Mounting and Repairing Filesystems](05-storage/03-filesystems-mounting.md) · [exercises](../exercises/05-storage/03-filesystems-mounting.md)
4. [Logical Volume Management (LVM)](05-storage/04-lvm.md) · [exercises](../exercises/05-storage/04-lvm.md)
5. [Swap Space](05-storage/05-swap.md) · [exercises](../exercises/05-storage/05-swap.md)
6. [NFS: Network File System](05-storage/06-nfs.md) · [exercises](../exercises/05-storage/06-nfs.md)
7. [Network Block Devices and iSCSI](05-storage/07-network-block-devices.md) · [exercises](../exercises/05-storage/07-network-block-devices.md)
8. [Filesystem Automounters](05-storage/08-automounters.md) · [exercises](../exercises/05-storage/08-automounters.md)
9. [Monitoring Storage Performance](05-storage/09-storage-performance.md) · [exercises](../exercises/05-storage/09-storage-performance.md)

### Module 06 – [Kernel, Modules & Devices](06-kernel-devices/README.md)

Course: [06-Kernel-Modules-Devices.md](../course/06-Kernel-Modules-Devices.md) · Exercises: [module 06](../exercises/06-kernel-devices/README.md)

1. [The Kernel and Boot Parameters](06-kernel-devices/01-kernel-boot-parameters.md) · [exercises](../exercises/06-kernel-devices/01-kernel-boot-parameters.md)
2. [Kernel Parameters with sysctl](06-kernel-devices/02-sysctl.md) · [exercises](../exercises/06-kernel-devices/02-sysctl.md)
3. [Kernel Modules](06-kernel-devices/03-kernel-modules.md) · [exercises](../exercises/06-kernel-devices/03-kernel-modules.md)
4. [Devices and udev](06-kernel-devices/04-devices-udev.md) · [exercises](../exercises/06-kernel-devices/04-devices-udev.md)

### Module 07 – [Networking](07-networking/README.md)

Course: [07-Networking.md](../course/07-Networking.md) · Exercises: [module 07](../exercises/07-networking/README.md)

1. [Network Addresses: IPv4 and IPv6](07-networking/01-ip-addressing.md) · [exercises](../exercises/07-networking/01-ip-addressing.md)
2. [Runtime Network Configuration with ip](07-networking/02-runtime-ip.md) · [exercises](../exercises/07-networking/02-runtime-ip.md)
3. [Persistent Network Configuration: Netplan and NetworkManager](07-networking/03-persistent-config.md) · [exercises](../exercises/07-networking/03-persistent-config.md)
4. [Hostname and Name Resolution](07-networking/04-hostname-resolution.md) · [exercises](../exercises/07-networking/04-hostname-resolution.md)
5. [Static Routing](07-networking/05-static-routing.md) · [exercises](../exercises/07-networking/05-static-routing.md)
6. [Bridges and Bonds](07-networking/06-bridges-bonds.md) · [exercises](../exercises/07-networking/06-bridges-bonds.md)
7. [Packet Filtering, Port Redirection and NAT](07-networking/07-firewalls-nat.md) · [exercises](../exercises/07-networking/07-firewalls-nat.md)
8. [The OpenSSH Server and Client](07-networking/08-openssh.md) · [exercises](../exercises/07-networking/08-openssh.md)
9. [Setting and Synchronizing System Time](07-networking/09-time-sync.md) · [exercises](../exercises/07-networking/09-time-sync.md)
10. [Reverse Proxies and Load Balancers](07-networking/10-reverse-proxy.md) · [exercises](../exercises/07-networking/10-reverse-proxy.md)
11. [Monitoring and Troubleshooting the Network](07-networking/11-troubleshooting.md) · [exercises](../exercises/07-networking/11-troubleshooting.md)

### Module 08 – [Security, Virtualization & Containers](08-security-vms-containers/README.md)

Course: [08-Security-VMs-Containers.md](../course/08-Security-VMs-Containers.md) · Exercises: [module 08](../exercises/08-security-vms-containers/README.md)

1. [Linux Security Modules: DAC and MAC](08-security-vms-containers/01-lsm-dac-mac.md) · [exercises](../exercises/08-security-vms-containers/01-lsm-dac-mac.md)
2. [SELinux](08-security-vms-containers/02-selinux.md) · [exercises](../exercises/08-security-vms-containers/02-selinux.md)
3. [AppArmor](08-security-vms-containers/03-apparmor.md) · [exercises](../exercises/08-security-vms-containers/03-apparmor.md)
4. [Working with SSL/TLS Certificates](08-security-vms-containers/04-ssl-certificates.md) · [exercises](../exercises/08-security-vms-containers/04-ssl-certificates.md)
5. [Virtual Machines with libvirt](08-security-vms-containers/05-libvirt-vms.md) · [exercises](../exercises/08-security-vms-containers/05-libvirt-vms.md)
6. [Containers: Docker and Podman](08-security-vms-containers/06-containers.md) · [exercises](../exercises/08-security-vms-containers/06-containers.md)

### Module 09 – [Backup, Recovery & System Rescue](09-backup-rescue/README.md)

Course: [09-Backup-Recovery-Rescue.md](../course/09-Backup-Recovery-Rescue.md) · Exercises: [module 09](../exercises/09-backup-rescue/README.md)

1. [Backup Strategy](09-backup-rescue/01-backup-strategy.md) · [exercises](../exercises/09-backup-rescue/01-backup-strategy.md)
2. [Backup Tools: tar, rsync, dd and Friends](09-backup-rescue/02-backup-tools.md) · [exercises](../exercises/09-backup-rescue/02-backup-tools.md)
3. [The Boot Process – and Where It Breaks](09-backup-rescue/03-boot-process.md) · [exercises](../exercises/09-backup-rescue/03-boot-process.md)
4. [System Rescue and Recovery](09-backup-rescue/04-system-rescue.md) · [exercises](../exercises/09-backup-rescue/04-system-rescue.md)

## Study aids

| Document | Use it for |
|----------|-----------|
| [Resources & Links](../resources/Resources-and-Links.md) | Official docs, books, practice sites, tools |
| [Glossary](../resources/Glossary.md) | Short definitions of every term |
| [Debian vs RHEL Differences](../resources/Distro-Differences.md) | Paths, packages and tools per family |
| [Troubleshooting Methodology](../resources/Troubleshooting-Methodology.md) | A repeatable method for "it doesn't work" |
| [Persistence Checklist](../resources/Persistence-Checklist.md) | Runtime vs persistent configuration – the #1 exam trap |
