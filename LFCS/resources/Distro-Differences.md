# Debian/Ubuntu vs RHEL Family - Differences

> The LFCS exam lets you work on Ubuntu, but you must recognise both families. Most commands are identical; this table lists what is **not**. Verify on your two lab VMs (`deb1`, `rhel1`).

---

## Packages & Repositories

| Topic | Debian / Ubuntu | RHEL / Rocky / Alma / Fedora |
|-------|-----------------|------------------------------|
| Package format | `.deb` | `.rpm` |
| Low-level tool | `dpkg` | `rpm` |
| High-level tool | `apt` | `dnf` (`yum` alias) |
| Refresh metadata | `apt update` (required) | automatic (`dnf makecache`) |
| Which package owns a file | `dpkg -S /path` | `rpm -qf /path` |
| Which package provides a file (not installed) | `apt-file search /path` | `dnf provides /path` |
| List package files | `dpkg -L pkg` | `rpm -ql pkg` |
| Repo config | `/etc/apt/sources.list.d/*.sources` (deb822) or `*.list` | `/etc/yum.repos.d/*.repo` |
| Signing keys | `/etc/apt/keyrings/` + `Signed-By:` | `rpm --import`, `gpgkey=` |
| Extra repos | PPAs, `universe` | EPEL, CRB |
| Automatic updates | `unattended-upgrades` | `dnf-automatic` |

## Users & Authentication

| Topic | Debian / Ubuntu | RHEL family |
|-------|-----------------|-------------|
| Admin group for sudo | `sudo` | `wheel` |
| Interactive add-user helper | `adduser` (Perl wrapper, prompts) | `adduser` = symlink to `useradd` |
| Default shell for `useradd` | `/bin/sh` (set `-s /bin/bash`!) | `/bin/bash` |
| `useradd` creates home | no, unless `-m` | yes (`CREATE_HOME yes`) |
| Auth log | `/var/log/auth.log` | `/var/log/secure` |
| PAM configuration tool | `pam-auth-update` | `authselect` |
| System-wide bashrc | `/etc/bash.bashrc` | `/etc/bashrc` |

## Services & Logs

| Topic | Debian / Ubuntu | RHEL family |
|-------|-----------------|-------------|
| SSH service name | `ssh` (socket-activated on 24.04+) | `sshd` |
| Web server package/service | `apache2` / `nginx` | `httpd` / `nginx` |
| Apache config | `/etc/apache2/` (`sites-available` + `a2ensite`) | `/etc/httpd/conf.d/` |
| Nginx sites | `sites-available/` + symlink in `sites-enabled/`, also `conf.d/` | `/etc/nginx/conf.d/*.conf` |
| Cron service | `cron` | `crond` |
| Main syslog file | `/var/log/syslog` | `/var/log/messages` |
| Journal persistent by default | yes (Ubuntu) | varies - check `/var/log/journal` |
| Services auto-start after install | **yes** | **no** - `systemctl enable --now` |

## Networking

| Topic | Debian / Ubuntu | RHEL family |
|-------|-----------------|-------------|
| Persistent config | **netplan** YAML in `/etc/netplan/` | **NetworkManager** keyfiles in `/etc/NetworkManager/system-connections/` |
| Backend | `systemd-networkd` (server) | NetworkManager |
| CLI | `netplan try/apply`, `networkctl` | `nmcli`, `nmtui` |
| DNS resolver | `systemd-resolved` (stub `127.0.0.53`, `resolvectl`) | NetworkManager writes `/etc/resolv.conf` |
| Firewall front-end | `ufw` (often inactive by default) | `firewalld` (active by default) |
| Time sync | `systemd-timesyncd` or `chrony` (24.04+) | `chronyd` |
| chrony config | `/etc/chrony/chrony.conf` | `/etc/chrony.conf` |
| NFS server package | `nfs-kernel-server` | `nfs-utils` |
| NFS service | `nfs-kernel-server` / `nfs-server` | `nfs-server` |

## Security

| Topic | Debian / Ubuntu | RHEL family |
|-------|-----------------|-------------|
| MAC system | **AppArmor** | **SELinux** |
| Status | `aa-status` | `getenforce`, `sestatus` |
| Denial logs | `journalctl -k \| grep apparmor` | `ausearch -m AVC`, `/var/log/audit/audit.log` |
| Add trusted CA | `/usr/local/share/ca-certificates/*.crt` + `update-ca-certificates` | `/etc/pki/ca-trust/source/anchors/` + `update-ca-trust` |
| System CA bundle | `/etc/ssl/certs/ca-certificates.crt` | `/etc/pki/tls/certs/ca-bundle.crt` |
| Private keys location (convention) | `/etc/ssl/private/` | `/etc/pki/tls/private/` |

## Boot & Kernel

| Topic | Debian / Ubuntu | RHEL family |
|-------|-----------------|-------------|
| Regenerate GRUB config | `update-grub` | `grub2-mkconfig -o /boot/grub2/grub.cfg` (BIOS & UEFI on RHEL 9) or `grubby` |
| GRUB directory | `/boot/grub/` | `/boot/grub2/` |
| Rebuild initramfs | `update-initramfs -u` | `dracut -f` |
| initramfs file | `/boot/initrd.img-<ver>` | `/boot/initramfs-<ver>.img` |
| Root password reset trick | `init=/bin/bash` | `rd.break` (+ `/.autorelabel`) |
| Default filesystem | ext4 | XFS |

## Virtualization & Containers

| Topic | Debian / Ubuntu | RHEL family |
|-------|-----------------|-------------|
| libvirt packages | `qemu-kvm libvirt-daemon-system virtinst` | `qemu-kvm libvirt virt-install` |
| libvirt access group | `libvirt` | `libvirt` |
| Default container engine | Docker (`docker.io` or Docker CE) | Podman |
| Containers at boot | `--restart=always` | Quadlet units in `/etc/containers/systemd/` or `~/.config/containers/systemd/` |
