# Persistence Checklist - Runtime vs Permanent

> The #1 way to lose points: a change that works now and disappears after reboot. The exam environment may be rebooted before grading. For every topic, know **both** columns - use the runtime command to test, the persistent method to finish.

---

| Topic | Runtime only (lost on reboot) | Persistent | Apply / verify |
|-------|-------------------------------|------------|----------------|
| Environment variable | `export VAR=x` | `/etc/profile.d/*.sh` (all) · `~/.bashrc` (one user) · `/etc/environment` | new login shell, `echo $VAR` |
| Alias | `alias ll='ls -l'` | `~/.bashrc` or `/etc/profile.d/` | new shell |
| umask | `umask 027` | `~/.bashrc`, `/etc/profile.d/`, `/etc/login.defs` (`UMASK`) | new login, `umask` |
| Resource limits | `ulimit -n 4096` | `/etc/security/limits.conf` or `limits.d/` · service: `LimitNOFILE=` | new login, `ulimit -a` / `systemctl show -p LimitNOFILE` |
| Service running | `systemctl start` | `systemctl enable` (or `enable --now`) | `systemctl is-enabled` / `is-active` |
| Unit change | - | `systemctl edit unit` (drop-in in `/etc/systemd/system/`) | `daemon-reload`, `systemctl cat unit` |
| Default boot target | `systemctl isolate x.target` | `systemctl set-default x.target` | `systemctl get-default` |
| Journal storage | - | `mkdir /var/log/journal` or `Storage=persistent` | `journalctl --list-boots` |
| Mount | `mount /dev/x /mnt` | `/etc/fstab` (UUID) or `.mount` unit | `systemctl daemon-reload && mount -a`, `findmnt` |
| Swap | `swapon /swapfile` | fstab `... none swap defaults 0 0` | `swapon -a`, `swapon --show` |
| Kernel parameter | `sysctl -w k=v` / `echo > /proc/sys/...` | `/etc/sysctl.d/99-x.conf` | `sysctl --system`, `sysctl k` |
| Kernel cmdline | edit in GRUB menu (`e`) | `/etc/default/grub` + regenerate · `grubby --args` | `cat /proc/cmdline` after reboot |
| Kernel module | `modprobe mod` | `/etc/modules-load.d/x.conf` | `lsmod \| grep mod` |
| Module option | `modprobe mod opt=v` | `/etc/modprobe.d/x.conf` (`options mod opt=v`) | `cat /sys/module/mod/parameters/opt` |
| Module blacklist | `modprobe -r mod` | `/etc/modprobe.d/` `blacklist` / `install mod /bin/false` (+ rebuild initramfs) | reboot, `lsmod` |
| udev rule | - | `/etc/udev/rules.d/NN-x.rules` | `udevadm control --reload && udevadm trigger` |
| IP address / route | `ip addr add` · `ip route add` | netplan YAML (`netplan apply`) · `nmcli con mod` (`nmcli con up`) | `ip a`, `ip r` |
| Hostname | `hostname x` | `hostnamectl set-hostname x` | `hostnamectl` |
| Name resolution | - | `/etc/hosts` · DNS in netplan/nmcli | `getent hosts name` |
| Firewall (firewalld) | `firewall-cmd --add-...` | `--permanent` + `--reload` (or `--runtime-to-permanent`) | `firewall-cmd --list-all` |
| Firewall (nftables) | `nft add rule ...` | `/etc/nftables.conf` (Debian) · `/etc/sysconfig/nftables.conf` (RHEL) + `enable nftables` | `nft list ruleset` |
| Firewall (ufw) | - (ufw rules are persistent) | `ufw allow ...`, `ufw enable` | `ufw status verbose` |
| Time zone / NTP | `date -s` | `timedatectl set-timezone` · `set-ntp true` · chrony config | `timedatectl`, `chronyc sources` |
| SELinux mode | `setenforce 0/1` | `/etc/selinux/config` | `getenforce` after reboot |
| SELinux boolean | `setsebool b on` | `setsebool -P b on` | `getsebool b` |
| SELinux file label | `chcon -t type file` | `semanage fcontext -a` + `restorecon -Rv` | `ls -Z` (survives relabel) |
| SELinux port | - | `semanage port -a -t type -p tcp N` | `semanage port -l \| grep N` |
| User group membership | `newgrp` (current shell) | `usermod -aG` | new login, `id user` |
| Container at boot | `podman/docker run` | `--restart=always` (Docker) · Quadlet/systemd unit (Podman) | reboot, `podman ps` / `docker ps` |
| VM at boot | `virsh start vm` | `virsh autostart vm` | `virsh list --autostart` |
| Scheduled job | `at` (one-off by design) | `crontab -e` · `/etc/cron.d/` · systemd `.timer` (`enable --now`) | `crontab -l`, `systemctl list-timers` |

---

## Final Pass Before Ending a Task

- [ ] Did I use a persistent method, not just a runtime command?
- [ ] Did I reload/apply (`daemon-reload`, `--reload`, `sysctl --system`, `mount -a`, `netplan apply`)?
- [ ] Is the service **enabled** as well as started?
- [ ] Would it survive a reboot? (When in doubt on a lab VM - reboot and check.)
