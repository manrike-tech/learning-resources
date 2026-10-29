# Module 10 - Exam Preparation & Mock Exam

**Exam domain:** All  
**Study aids:** [Persistence Checklist](../resources/Persistence-Checklist.md) · [Troubleshooting Methodology](../resources/Troubleshooting-Methodology.md) · [Distro Differences](../resources/Distro-Differences.md)

---

## Exam-Day Strategy

1. **Read the whole task**, note every requirement (names, paths, permissions, "persistent", "at boot").
2. **Check which host** the task wants. Tasks often say `ssh <host>` first — run it, and `exit` back when done.
3. **Skip and flag** anything that takes more than ~5 minutes; come back at the end.
4. **Verify** each task: `systemctl is-enabled`, `mount -a`, `getent`, `curl`, `cat` the output file.
5. **Persistence**: if it should survive reboot, it must be in a file (fstab, sysctl.d, netplan/nmcli, unit enabled, `--permanent`).
6. Use `man`, `--help` and `/usr/share/doc` — they are your only documentation.
7. Don't reboot unless necessary; if you must, make sure fstab is valid first.

## Speed Kit (memorize)

```bash
# Syntax checkers
sshd -t | nginx -t | findmnt --verify | visudo -c | named-checkconf | systemd-analyze verify unit
# "Where is the config?"
rpm -qc pkg | dpkg -L pkg | grep etc | systemctl cat svc
# "Did it work?"
systemctl status svc --no-pager; journalctl -u svc -n 30 --no-pager; ss -tulpn
```

---

## Competency Checklist (official domains)

### Operations & Deployment - 25%
- [ ] Configure kernel parameters, persistent and non-persistent → M06
- [ ] Diagnose, identify, manage, and troubleshoot processes and services → M04
- [ ] Manage or schedule jobs for executing commands → M04
- [ ] Search for, install, validate, and maintain software packages or repositories → M03
- [ ] Recover from hardware, operating system, or filesystem failures → M09
- [ ] Manage Virtual Machines (libvirt) → M08
- [ ] Configure container engines, create and manage containers → M08
- [ ] Create and enforce MAC using SELinux → M08

### Networking - 25%
- [ ] Configure IPv4 and IPv6 networking and hostname resolution → M07
- [ ] Set and synchronize system time using time servers → M07
- [ ] Monitor and troubleshoot networking → M07
- [ ] Configure the OpenSSH server and client → M07
- [ ] Configure packet filtering, port redirection, and NAT → M07
- [ ] Configure static routing → M07
- [ ] Configure bridge and bonding devices → M07
- [ ] Implement reverse proxies and load balancers → M07

### Storage - 20%
- [ ] Configure and manage LVM storage → M05
- [ ] Manage and configure the virtual file system → M05
- [ ] Create, manage, and troubleshoot filesystems → M05
- [ ] Use remote filesystems and network block devices → M05
- [ ] Configure and manage swap space → M05
- [ ] Configure filesystem automounters → M05
- [ ] Monitor storage performance → M05

### Essential Commands - 20%
- [ ] Basic Git operations → M03
- [ ] Create, configure, and troubleshoot services → M04
- [ ] Monitor and troubleshoot system performance and services → M04
- [ ] Determine application and service specific constraints → M04
- [ ] Troubleshoot diskspace issues → M04
- [ ] Work with SSL certificates → M08

### Users & Groups - 10%
- [ ] Create and manage local user and group accounts → M02
- [ ] Manage personal and system-wide environment profiles → M01 / M02
- [ ] Configure user resource limits → M02
- [ ] Configure and manage ACLs → M02
- [ ] Configure the system to use LDAP user and group accounts → M02

---

## Mock Exam (2 hours, 20 tasks)

> Start from fresh snapshots of `deb1` and `rhel1`. Don't look at the hints until time is up.

| # | Host | Task |
|---|------|------|
| 1 | deb1 | Find all files in `/usr/share` larger than 5 MB, save full paths sorted to `/opt/big.txt`. |
| 2 | deb1 | Create group `web` (GID 4000), user `deploy` (UID 4001, primary group `web`, shell bash, account expires 2027-06-30). |
| 3 | deb1 | Create `/srv/site` (group `web`, SGID). Give user `qa` read+execute on it and on all future files, via ACL. |
| 4 | deb1 | Limit user `deploy` to max 2048 open files (hard). |
| 5 | rhel1 | Install the package that provides `/usr/bin/dig`; list its documentation files into `/opt/dig-docs.txt`. |
| 6 | rhel1 | Clone `/srv/git/app.git`, create branch `hotfix`, commit `fix.txt` with message "Hotfix", push the branch. |
| 7 | rhel1 | Create service `pinger.service` that runs `/usr/local/bin/pinger.sh`, restarts always, starts at boot. |
| 8 | rhel1 | Run `/usr/local/bin/cleanup.sh` every Sunday at 03:10 as user `deploy`. |
| 9 | rhel1 | A process is consuming 100% CPU. Find it and lower its priority to 19 without killing it. |
| 10 | deb1 | `/var` is reported full. Find the cause and free the space without rebooting. |
| 11 | deb1 | Disk 1: VG `vgdata`, LV `lvapp` 1 GiB, ext4, mounted persistently on `/app`. Then grow it by 512 MiB. |
| 12 | deb1 | Add 256 MiB of swap from a file, persistent. |
| 13 | rhel1 | Export `/exports/pub` read-only to the lab subnet. On deb1, automount it at `/net/pub` with autofs. |
| 14 | rhel1 | Set `net.ipv4.ip_forward=1` persistently and `kernel.sysrq=0` **only** at runtime. |
| 15 | rhel1 | Static IPv4 `192.168.56.20/24` + IPv6 `fd00:56::20/64` on the lab NIC; route `10.50.0.0/16` via `192.168.56.1`, all persistent. |
| 16 | rhel1 | Bond `bond0` (active-backup) using two spare NICs, IP `192.168.56.40/24`. |
| 17 | rhel1 | Firewall: allow HTTPS; forward incoming TCP 8443 to local 443; persistent. |
| 18 | deb1 | Nginx on port 80 load-balancing to `rhel1:8081` and `rhel1:8082`. |
| 19 | rhel1 | Serve `/srv/web` with httpd on port 8888, SELinux enforcing, with a self-signed cert for `web.lab` valid 730 days at `/etc/pki/tls/certs/web.crt`. |
| 20 | rhel1 | Run container `cache` from `redis:alpine`, restart policy always, port 6379 published, data volume `redisdata` at `/data`. |

<details>
<summary><b>Hints (open after finishing)</b></summary>

1. `find /usr/share -type f -size +5M | sort > /opt/big.txt`
2. `groupadd -g 4000 web; useradd -m -u 4001 -g web -s /bin/bash -e 2027-06-30 deploy`
3. `chgrp web /srv/site; chmod g+s /srv/site; setfacl -m u:qa:rx /srv/site; setfacl -m d:u:qa:rx /srv/site`
4. `/etc/security/limits.d/deploy.conf` → `deploy hard nofile 2048`
5. `dnf provides /usr/bin/dig` → `bind-utils`; `rpm -qd bind-utils > /opt/dig-docs.txt`
6. `git clone …; git switch -c hotfix; … git commit -m "Hotfix"; git push -u origin hotfix`
7. Unit with `Restart=always`, `WantedBy=multi-user.target`; `daemon-reload`; `enable --now`
8. `crontab -u deploy -e` → `10 3 * * 0 /usr/local/bin/cleanup.sh`
9. `top` / `ps --sort=-%cpu`; `renice -n 19 -p PID`
10. `du -xh /var --max-depth=2 | sort -h`, `lsof +L1`, `journalctl --vacuum-size=`
11. `pvcreate; vgcreate; lvcreate -L 1G -n lvapp vgdata; mkfs.ext4`; fstab; `mount -a`; `lvextend -r -L +512M`
12. `fallocate`/`dd`, `chmod 600`, `mkswap`, `swapon`, fstab line
13. `/etc/exports` + `exportfs -ar` + firewall; `/etc/auto.master.d/net.autofs` → `/net /etc/auto.net2`; map: `pub -ro rhel1:/exports/pub`
14. `/etc/sysctl.d/90-fwd.conf` + `sysctl --system`; `sysctl -w kernel.sysrq=0`
15. `nmcli con mod … ipv4.method manual … ipv6.method manual … +ipv4.routes "10.50.0.0/16 192.168.56.1"; nmcli con up`
16. `nmcli con add type bond … bond.options "mode=active-backup,miimon=100"` + two ports
17. `firewall-cmd --add-service=https --add-forward-port=port=8443:proto=tcp:toport=443 --permanent; --reload`
18. `upstream` block + `proxy_pass`; `nginx -t`; reload
19. `semanage fcontext -a -t httpd_sys_content_t "/srv/web(/.*)?"; restorecon -Rv /srv/web; semanage port -a -t http_port_t -p tcp 8888`; `openssl req -x509 -days 730 …`; firewall
20. `docker/podman run -d --name cache --restart always -p 6379:6379 -v redisdata:/data redis:alpine`

</details>

---

## Final 2 Weeks Plan

| Day | Activity |
|-----|----------|
| 1–3 | Redo labs of your weakest 3 modules from scratch |
| 4 | Mock exam (above) - timed |
| 5 | Review mistakes; repeat failed tasks twice |
| 6 | **Killer.sh session 1** (36 h access) |
| 7–8 | Study every Killer.sh solution, even the ones you got right |
| 9–10 | Networking + Storage drills (50% of the exam) |
| 11 | **Killer.sh session 2** - aim for speed |
| 12 | Read the Candidate Handbook & LFCS Important Instructions; test your system (webcam, ID) |
| 13 | Light review, rest |
| 14 | **Exam** |
