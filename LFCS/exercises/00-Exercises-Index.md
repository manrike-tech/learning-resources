# LFCS Exercises – Index

> One exercise file per lesson, organised by course module. Exercises are graded ★ warm-up · ★★ practice · ★★★ challenge; each has a **Verify** step and a collapsible **Solution** – try first, then check.

## Lab environment

Use the two VMs from the [course setup](../course/00-Course-Index.md#lab-setup-do-this-first): `deb1` (Ubuntu Server) and `rhel1` (Rocky/Alma/RHEL), each with two extra 5 GB disks, on the same internal network. **Take a snapshot before every exercise session** and revert freely. Most files end with a *Cleanup* section.

## Exercise files

### Module 01 – [Foundations & Essential Commands](01-foundations/README.md)

Theory: [module 01](../theory/01-foundations/README.md) · Course lab: [01-Foundations.md](../course/01-Foundations.md)

| # | Exercises | Count |
|---|-----------|-------|
| 1 | [The Linux Filesystem Hierarchy](01-foundations/01-filesystem-hierarchy.md) | 8 |
| 2 | [Finding Files](01-foundations/02-finding-files.md) | 8 |
| 3 | [Working with Text](01-foundations/03-working-with-text.md) | 7 |
| 4 | [Redirection, Pipes and Archives](01-foundations/04-redirection-pipes-archives.md) | 8 |
| 5 | [The User Environment](01-foundations/05-user-environment.md) | 8 |
| 6 | [Finding Help Offline](01-foundations/06-finding-help.md) | 7 |

### Module 02 – [Users, Groups, Permissions & LDAP](02-users-groups-permissions/README.md)

Theory: [module 02](../theory/02-users-groups-permissions/README.md) · Course lab: [02-Users-Groups-Permissions.md](../course/02-Users-Groups-Permissions.md)

| # | Exercises | Count |
|---|-----------|-------|
| 1 | [User Accounts](02-users-groups-permissions/01-user-accounts.md) | 10 |
| 2 | [Groups](02-users-groups-permissions/02-groups.md) | 7 |
| 3 | [sudo and su](02-users-groups-permissions/03-sudo.md) | 9 |
| 4 | [File Permissions and Ownership](02-users-groups-permissions/04-permissions-ownership.md) | 10 |
| 5 | [Access Control Lists](02-users-groups-permissions/05-acls.md) | 7 |
| 6 | [Environment Profiles](02-users-groups-permissions/06-environment-profiles.md) | 8 |
| 7 | [User Resource Limits](02-users-groups-permissions/07-resource-limits.md) | 8 |
| 8 | [LDAP Clients with SSSD](02-users-groups-permissions/08-ldap-clients.md) | 5 |

### Module 03 – [Package Management & Git](03-packages-git/README.md)

Theory: [module 03](../theory/03-packages-git/README.md) · Course lab: [03-Packages-and-Git.md](../course/03-Packages-and-Git.md)

| # | Exercises | Count |
|---|-----------|-------|
| 1 | [Package Management Concepts](03-packages-git/01-package-management-concepts.md) | 7 |
| 2 | [dpkg and APT](03-packages-git/02-dpkg-apt.md) | 9 |
| 3 | [rpm and dnf](03-packages-git/03-rpm-dnf.md) | 8 |
| 4 | [zypper](03-packages-git/04-zypper.md) | 6 |
| 5 | [Git Fundamentals](03-packages-git/05-git-fundamentals.md) | 9 |

### Module 04 – [Processes, Services, Scheduling & Performance](04-processes-services/README.md)

Theory: [module 04](../theory/04-processes-services/README.md) · Course lab: [04-Processes-Services-Performance.md](../course/04-Processes-Services-Performance.md)

| # | Exercises | Count |
|---|-----------|-------|
| 1 | [Processes](04-processes-services/01-processes.md) | 8 |
| 2 | [Monitoring System Performance](04-processes-services/02-monitoring.md) | 8 |
| 3 | [Troubleshooting Disk Space](04-processes-services/03-disk-space.md) | 7 |
| 4 | [Managing Services with systemd](04-processes-services/04-systemd.md) | 9 |
| 5 | [Application and Service Constraints](04-processes-services/05-service-constraints.md) | 8 |
| 6 | [Scheduling Jobs](04-processes-services/06-scheduling-jobs.md) | 9 |

### Module 05 – [Storage](05-storage/README.md)

Theory: [module 05](../theory/05-storage/README.md) · Course lab: [05-Storage.md](../course/05-Storage.md)

| # | Exercises | Count |
|---|-----------|-------|
| 1 | [The VFS and Filesystem Types](05-storage/01-vfs-filesystem-types.md) | 6 |
| 2 | [Disk Partitioning](05-storage/02-partitioning.md) | 7 |
| 3 | [Creating, Mounting and Repairing Filesystems](05-storage/03-filesystems-mounting.md) | 8 |
| 4 | [Logical Volume Management](05-storage/04-lvm.md) | 10 |
| 5 | [Swap Space](05-storage/05-swap.md) | 8 |
| 6 | [NFS](05-storage/06-nfs.md) | 8 |
| 7 | [Network Block Devices](05-storage/07-network-block-devices.md) | 7 |
| 8 | [Filesystem Automounters](05-storage/08-automounters.md) | 7 |
| 9 | [Monitoring Storage Performance](05-storage/09-storage-performance.md) | 7 |

### Module 06 – [Kernel, Modules & Devices](06-kernel-devices/README.md)

Theory: [module 06](../theory/06-kernel-devices/README.md) · Course lab: [06-Kernel-Modules-Devices.md](../course/06-Kernel-Modules-Devices.md)

| # | Exercises | Count |
|---|-----------|-------|
| 1 | [The Kernel and Boot Parameters](06-kernel-devices/01-kernel-boot-parameters.md) | 8 |
| 2 | [Kernel Parameters with sysctl](06-kernel-devices/02-sysctl.md) | 8 |
| 3 | [Kernel Modules](06-kernel-devices/03-kernel-modules.md) | 7 |
| 4 | [Devices and udev](06-kernel-devices/04-devices-udev.md) | 8 |

### Module 07 – [Networking](07-networking/README.md)

Theory: [module 07](../theory/07-networking/README.md) · Course lab: [07-Networking.md](../course/07-Networking.md)

| # | Exercises | Count |
|---|-----------|-------|
| 1 | [Network Addresses: IPv4 and IPv6](07-networking/01-ip-addressing.md) | 7 |
| 2 | [Runtime Configuration with ip](07-networking/02-runtime-ip.md) | 7 |
| 3 | [Persistent Network Configuration](07-networking/03-persistent-config.md) | 7 |
| 4 | [Hostname and Name Resolution](07-networking/04-hostname-resolution.md) | 8 |
| 5 | [Static Routing](07-networking/05-static-routing.md) | 8 |
| 6 | [Bridges and Bonds](07-networking/06-bridges-bonds.md) | 7 |
| 7 | [Packet Filtering, Port Redirection and NAT](07-networking/07-firewalls-nat.md) | 9 |
| 8 | [The OpenSSH Server and Client](07-networking/08-openssh.md) | 10 |
| 9 | [Setting and Synchronizing System Time](07-networking/09-time-sync.md) | 7 |
| 10 | [Reverse Proxies and Load Balancers](07-networking/10-reverse-proxy.md) | 8 |
| 11 | [Monitoring and Troubleshooting the Network](07-networking/11-troubleshooting.md) | 8 |

### Module 08 – [Security, Virtualization & Containers](08-security-vms-containers/README.md)

Theory: [module 08](../theory/08-security-vms-containers/README.md) · Course lab: [08-Security-VMs-Containers.md](../course/08-Security-VMs-Containers.md)

| # | Exercises | Count |
|---|-----------|-------|
| 1 | [Linux Security Modules: DAC and MAC](08-security-vms-containers/01-lsm-dac-mac.md) | 6 |
| 2 | [SELinux](08-security-vms-containers/02-selinux.md) | 9 |
| 3 | [AppArmor](08-security-vms-containers/03-apparmor.md) | 7 |
| 4 | [SSL/TLS Certificates](08-security-vms-containers/04-ssl-certificates.md) | 9 |
| 5 | [Virtual Machines with libvirt](08-security-vms-containers/05-libvirt-vms.md) | 9 |
| 6 | [Containers: Docker and Podman](08-security-vms-containers/06-containers.md) | 11 |

### Module 09 – [Backup, Recovery & System Rescue](09-backup-rescue/README.md)

Theory: [module 09](../theory/09-backup-rescue/README.md) · Course lab: [09-Backup-Recovery-Rescue.md](../course/09-Backup-Recovery-Rescue.md)

| # | Exercises | Count |
|---|-----------|-------|
| 1 | [Backup Strategy](09-backup-rescue/01-backup-strategy.md) | 6 |
| 2 | [Backup Tools](09-backup-rescue/02-backup-tools.md) | 9 |
| 3 | [The Boot Process](09-backup-rescue/03-boot-process.md) | 7 |
| 4 | [System Rescue and Recovery](09-backup-rescue/04-system-rescue.md) | 10 |

**Total:** 59 exercise files, 466 exercises.

When every module is done, sit the timed **mock exam** in [Module 10 – Exam Preparation](../course/10-Exam-Prep.md#mock-exam-2-hours-20-tasks).
