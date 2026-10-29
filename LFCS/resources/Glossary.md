# LFCS - Glossary

> Short definitions of terms used across the course. The number in brackets is the **module** whose [theory pages](../theory/00-Theory-Index.md) explain it in depth.

---

| Term | Definition |
|------|-----------|
| **ACL** | Access Control List: extra permission entries for named users/groups beyond owner/group/other. [02] |
| **AppArmor** | Path-based MAC system used by Ubuntu and SUSE; confines programs with profiles. [08] |
| **ARP** | Address Resolution Protocol: finds the MAC address for an IPv4 address on the local network. [07] |
| **AVC** | Access Vector Cache: SELinux's decision cache; "AVC denied" messages are SELinux denials. [08] |
| **Block device** | Storage accessed in fixed-size blocks with random access (`/dev/sda`). [05] |
| **Bond** | Several NICs combined into one logical interface for redundancy or throughput. [07] |
| **Boolean (SELinux)** | Pre-defined on/off switch that changes SELinux policy behaviour. [08] |
| **Bridge** | Virtual layer-2 switch in the kernel; connects VMs/containers to a network. [07] |
| **CA** | Certificate Authority: entity that signs certificates, vouching for the key ↔ identity binding. [08] |
| **cgroup** | Control group: kernel feature limiting and accounting resources for a group of processes. [04][08] |
| **Character device** | Device accessed as a stream of bytes (`/dev/tty`, `/dev/null`). [01] |
| **chroot** | Changes the apparent root directory for a process; used in rescue. [09] |
| **CIDR** | Notation `address/prefix` (e.g. `10.0.0.0/8`) for IP networks. [07] |
| **Conntrack** | Kernel connection-tracking table that makes firewalls stateful and enables NAT. [07] |
| **Context (SELinux)** | Label `user:role:type:level` attached to processes and objects. [08] |
| **Copy-on-write (CoW)** | Data is shared until modified, then copied - used by fork, LVM snapshots, qcow2, overlayfs. [04][05] |
| **cron** | Daemon that runs scheduled recurring commands. [04] |
| **CSR** | Certificate Signing Request: public key + identity sent to a CA to be signed. [08] |
| **DAC** | Discretionary Access Control: owner-controlled permissions (rwx, ACLs). [08] |
| **Daemon** | Background service process (often name ends in `d`: `sshd`, `crond`). [04] |
| **dentry / directory entry** | Mapping of a name to an inode number inside a directory. [01] |
| **Distribution** | Linux kernel + userland + package manager + defaults (Ubuntu, RHEL…). [01] |
| **DNAT** | Destination NAT: rewrites the destination of incoming packets (port forwarding). [07] |
| **Drop-in** | A `.conf` file in `unit.d/` that overrides parts of a systemd unit. [04] |
| **Emergency target** | Minimal systemd state: root FS read-only, nothing else mounted. [09] |
| **Environment variable** | Exported variable inherited by child processes. [01] |
| **Extent** | Contiguous range of blocks; ext4/XFS use extents to map file data. LVM also uses extents (PE). [05] |
| **File descriptor (FD)** | Integer handle for an open file/socket/pipe; 0 stdin, 1 stdout, 2 stderr. [01] |
| **firewalld** | Zone-based firewall daemon (RHEL family) on top of nftables. [07] |
| **fork / exec** | System calls to duplicate a process and to replace its program. [04] |
| **fstab** | `/etc/fstab`: filesystems to mount at boot. [05] |
| **GID** | Group ID number. [02] |
| **Glob** | Shell wildcard pattern (`*.conf`) expanded to filenames. [01] |
| **GPT** | GUID Partition Table: modern partition scheme, >2 TiB, 128 partitions. [05] |
| **GRUB2** | The standard Linux bootloader. [09] |
| **Hard link** | Additional directory entry pointing to the same inode. [01] |
| **Host key** | SSH server's key pair identifying the server to clients. [07] |
| **Hypervisor** | Software that runs virtual machines (KVM, ESXi, Hyper-V). [08] |
| **Image (container)** | Read-only, layered template from which containers are created. [08] |
| **Incremental backup** | Backup of changes since the previous backup of any type. [09] |
| **initramfs** | Early-boot RAM filesystem with drivers/tools to mount the real root. [06][09] |
| **Inode** | On-disk structure holding a file's metadata and data location. [01][05] |
| **iSCSI** | SCSI block storage over TCP/IP (port 3260). [05] |
| **Journal (filesystem)** | Log of pending metadata changes allowing fast crash recovery. [05] |
| **Journal (systemd)** | Binary log store managed by `systemd-journald`, read with `journalctl`. [04] |
| **Kernel module** | Loadable piece of kernel code, usually a driver (`.ko`). [06] |
| **KVM** | Kernel-based Virtual Machine: Linux kernel module turning Linux into a hypervisor. [08] |
| **LDAP** | Lightweight Directory Access Protocol: queries a directory of users, groups, etc. [02] |
| **Load average** | Average number of runnable + uninterruptible tasks over 1/5/15 min. [04] |
| **LSM** | Linux Security Modules: kernel framework hosting SELinux, AppArmor, etc. [08] |
| **LV / VG / PV** | LVM Logical Volume / Volume Group / Physical Volume. [05] |
| **MAC (access control)** | Mandatory Access Control: system-wide policy root can't override. [08] |
| **MAC address** | 48-bit hardware address of a network interface. [07] |
| **Major / minor number** | Identify a device's driver and instance for a device file. [06] |
| **Mask (systemd)** | Link a unit to `/dev/null` so it can't be started at all. [04] |
| **Mask (ACL)** | ACL entry limiting max permissions of named users/groups and the owning group. [02] |
| **Masquerade** | SNAT to the outgoing interface's current IP. [07] |
| **MBR** | Master Boot Record: legacy partition scheme, 4 primaries, 2 TiB limit. [05] |
| **Modalias** | Device ID string used to auto-load the matching kernel module. [06] |
| **Mount point** | Directory where a filesystem is attached to the tree. [05] |
| **Namespace** | Kernel feature giving a process its own view of PIDs, network, mounts, etc. [08] |
| **NAT** | Network Address Translation: rewriting IPs/ports of packets. [07] |
| **NBD** | Network Block Device: exports a block device over TCP (port 10809). [05] |
| **NDP** | Neighbor Discovery Protocol: IPv6 replacement for ARP and more. [07] |
| **Netfilter** | Kernel packet-filtering framework with hooks used by nftables/iptables. [07] |
| **NetworkManager** | Network configuration daemon (default on RHEL, desktops), CLI `nmcli`. [07] |
| **NFS** | Network File System: file-level sharing over the network (port 2049). [05] |
| **nice** | Process CPU priority hint, -20 (high) to 19 (low). [04] |
| **NSS** | Name Service Switch: decides where to look up users, groups, hosts (`/etc/nsswitch.conf`). [02] |
| **NTP** | Network Time Protocol (UDP 123); implemented by chrony. [07] |
| **OOM killer** | Kernel mechanism that kills a process when memory is exhausted. [04] |
| **PAM** | Pluggable Authentication Modules: stack that authenticates users for each service. [02] |
| **Page cache** | RAM used to cache file data; reclaimable. [04] |
| **PE** | Physical Extent: LVM allocation unit. [05] |
| **PEM** | Base64 text format for certificates and keys. [08] |
| **PID 1** | First user-space process (systemd); ancestor of all others. [04] |
| **Pipe** | Connects one process's stdout to another's stdin. [01] |
| **Podman** | Daemonless, rootless-capable container engine (RHEL default). [08] |
| **Port (TCP/UDP)** | 16-bit number identifying a service endpoint on a host. [07] |
| **Profile (AppArmor)** | Set of path and capability rules confining a program. [08] |
| **qcow2** | QEMU copy-on-write disk image format. [08] |
| **QEMU** | Machine emulator providing virtual hardware for KVM guests. [08] |
| **Rescue target** | Single-user systemd state with local filesystems mounted. [09] |
| **Reverse proxy** | Server that accepts client requests on behalf of backend servers. [07] |
| **root_squash** | NFS option mapping client root to an unprivileged user. [05] |
| **Route / default gateway** | Rule telling the kernel where to send packets for a destination; `0.0.0.0/0` = default. [07] |
| **RPO / RTO** | Recovery Point / Recovery Time Objective. [09] |
| **SAN (certificate)** | Subject Alternative Name: hostnames/IPs a certificate is valid for. [08] |
| **SELinux** | Label-based MAC system (RHEL family). [08] |
| **SGID** | Special bit: run with file's group; on directories, new files inherit the group. [02] |
| **Signal** | Asynchronous notification to a process (`SIGTERM`, `SIGKILL`…). [04] |
| **Slice** | systemd cgroup grouping for resource control. [04] |
| **SNAT** | Source NAT: rewrites the source of outgoing packets. [07] |
| **Snapshot (LVM)** | Point-in-time CoW view of a logical volume. [05] |
| **SSSD** | System Security Services Daemon: connects NSS/PAM to LDAP/AD with caching. [02] |
| **Sticky bit** | On a directory, only owners can delete their files (`/tmp`). [02] |
| **SUID** | Special bit: executable runs with the file owner's UID. [02] |
| **Superblock** | Filesystem header with global metadata. [05] |
| **Swap** | Disk area used for memory pages evicted from RAM. [04][05] |
| **Symbolic link** | File containing a path to another file. [01] |
| **sysctl** | Interface to kernel parameters under `/proc/sys`. [06] |
| **sysfs** | `/sys`: virtual filesystem exposing the kernel device model. [06] |
| **System call** | Request from a user-space program to the kernel. [01][06] |
| **systemd** | Init system and service manager (PID 1). [04] |
| **Target (systemd)** | Unit grouping others, like a runlevel (`multi-user.target`). [04] |
| **Timer (systemd)** | Unit that activates another unit on a schedule. [04] |
| **TLS** | Transport Layer Security: encrypts and authenticates network connections. [08] |
| **Type enforcement** | SELinux model: rules allow process types to access object types. [08] |
| **udev** | Device manager creating `/dev` nodes and handling hotplug events. [06] |
| **UID** | User ID number; 0 = root. [02] |
| **umask** | Mask removing permission bits from newly created files. [02] |
| **Unit (systemd)** | Configuration object: service, socket, timer, mount, target… [04] |
| **USE method** | Check Utilization, Saturation, Errors for every resource. [04] |
| **User space / kernel space** | Unprivileged vs privileged CPU mode and memory regions. [06] |
| **UUID** | Universally unique ID; used to identify filesystems reliably. [05] |
| **VFS** | Virtual File System: kernel layer giving all filesystems one API. [05] |
| **virtio** | Paravirtualized device drivers for efficient VM I/O. [08] |
| **Volume (container)** | Persistent storage managed outside a container's writable layer. [08] |
| **XFS** | High-performance journaling filesystem; default on RHEL; cannot shrink. [05] |
| **Zombie** | Exited process whose parent hasn't collected its status yet. [04] |
| **Zone (firewalld)** | Trust level bound to interfaces/sources with its own allowed services. [07] |
