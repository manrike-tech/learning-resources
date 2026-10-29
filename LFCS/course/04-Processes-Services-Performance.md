# Module 04 - Processes, Services, Scheduling & Performance

**LFS207:** Ch. 14 Processes · Ch. 15 Process Monitoring · Ch. 16 Memory Monitoring · Ch. 17 I/O Monitoring · Ch. 31 System Init: systemd
**Exam domain:** Operations & Deployment (25%) · Essential Commands (20%)  
**Theory:** [Module 04 theory notes](../theory/04-processes-services/README.md) · **Exercises:** [Module 04 exercises](../exercises/04-processes-services/README.md)  
**Gaps covered:** cron, at, systemd timers; application/service constraints

---

## Learning Objectives

- Inspect, prioritize and signal processes.
- Create, customize and troubleshoot systemd services.
- Schedule one-off and recurring jobs.
- Diagnose CPU, memory, I/O and disk-space problems.

---

## Lesson 1 - Processes (Ch. 14)

> **Theory:** [Processes](../theory/04-processes-services/01-processes.md) · **Exercises:** [practice tasks](../exercises/04-processes-services/01-processes.md)

```bash
ps aux; ps -ef --forest; ps -u alice
ps -eo pid,ppid,user,%cpu,%mem,stat,cmd --sort=-%cpu | head
pgrep -a nginx; pidof sshd
kill -15 PID     # SIGTERM (graceful, default)
kill -9 PID      # SIGKILL (force)
kill -1 PID      # SIGHUP (often: reload config)
pkill -u bob; killall nginx
nice -n 10 cmd; renice -n 5 -p PID   # -20 (highest) .. 19 (lowest)
cmd &; jobs; fg %1; bg %1; Ctrl+Z; nohup cmd &
```

Process states (`STAT`): **R** running · **S** sleeping · **D** uninterruptible (usually I/O) · **T** stopped · **Z** zombie.

---

## Lesson 2 - Monitoring (Ch. 15–17)

> **Theory:** [Monitoring System Performance](../theory/04-processes-services/02-monitoring.md) · **Exercises:** [practice tasks](../exercises/04-processes-services/02-monitoring.md)

| Resource | Tools |
|----------|-------|
| CPU / load | `top`, `htop`, `uptime`, `mpstat`, `sar -u` |
| Memory | `free -h`, `vmstat 1`, `/proc/meminfo`, `top` (sort `M`) |
| I/O | `iostat -xz 1`, `iotop`, `vmstat` (`wa`, `bi/bo`) |
| Files/ports in use | `lsof`, `lsof -i :80`, `ss -tulpn`, `fuser -v /mnt` |

`sysstat` package provides `iostat`, `mpstat`, `sar`, `pidstat`.

> Load average ≈ number of processes runnable or waiting on I/O. Compare it with `nproc`.

OOM killer events: `journalctl -k | grep -i "out of memory"`.

---

## Lesson 3 - Troubleshooting Disk Space (Gap)

> **Theory:** [Troubleshooting Disk Space](../theory/04-processes-services/03-disk-space.md) · **Exercises:** [practice tasks](../exercises/04-processes-services/03-disk-space.md)

```bash
df -h; df -i                          # space and inodes
du -sh /var/* | sort -h               # biggest directories
du -xh / --max-depth=1 2>/dev/null | sort -h
find / -xdev -size +500M -type f 2>/dev/null
lsof +L1                              # deleted files still held open
journalctl --disk-usage; journalctl --vacuum-size=200M
```

> "Disk full but du shows little" → a process holds a deleted file open. Restart the process (or truncate via `/proc/PID/fd/N`).
> "No space left" but `df -h` shows free space → check `df -i` (inodes exhausted).

---

## Lesson 4 - systemd (Ch. 31)

> **Theory:** [Managing Services with systemd](../theory/04-processes-services/04-systemd.md) · **Exercises:** [practice tasks](../exercises/04-processes-services/04-systemd.md)

```bash
systemctl status nginx
systemctl start|stop|restart|reload nginx
systemctl enable --now nginx; systemctl disable nginx
systemctl mask nginx; systemctl unmask nginx   # prevent any start
systemctl is-enabled nginx; systemctl is-active nginx
systemctl list-units --type=service --state=failed
systemctl list-unit-files --type=service
systemctl cat nginx; systemctl show nginx -p Restart
systemctl daemon-reload                         # after editing unit files
systemctl get-default; systemctl set-default multi-user.target
systemctl isolate rescue.target
systemd-analyze blame
```

Unit file locations (priority high → low): `/etc/systemd/system/` > `/run/systemd/system/` > `/usr/lib/systemd/system/` (Debian also `/lib/systemd/system/`).

### Create a service

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My App
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=svc_app
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/run.sh
Restart=on-failure
RestartSec=5
Environment=PORT=8080
LimitNOFILE=8192
MemoryMax=512M

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now myapp
```

### Customize a vendor unit (drop-in)

```bash
systemctl edit nginx        # creates /etc/systemd/system/nginx.service.d/override.conf
```
```ini
[Service]
Restart=always
```

### Logs

```bash
journalctl -u nginx -f; journalctl -u nginx --since "1 hour ago"
journalctl -p err -b        # errors this boot
journalctl -b -1            # previous boot (needs persistent journal)
```

Persistent journal: `mkdir -p /var/log/journal` (or `Storage=persistent` in `/etc/systemd/journald.conf`) then `systemctl restart systemd-journald`.

---

## Lesson 5 - Service Constraints (Gap)

> **Theory:** [Application and Service Constraints](../theory/04-processes-services/05-service-constraints.md) · **Exercises:** [practice tasks](../exercises/04-processes-services/05-service-constraints.md)

"Why won't the service start/work?" checklist:

1. `systemctl status` + `journalctl -u` → read the actual error.
2. Port already in use? `ss -tulpn | grep :80`.
3. Permissions/ownership on files the service user needs.
4. SELinux/AppArmor denials (`ausearch -m avc -ts recent`, `journalctl | grep apparmor`).
5. Limits: `LimitNOFILE`, `MemoryMax`, `ulimit`.
6. Config syntax: `nginx -t`, `sshd -t`, `apachectl configtest`.
7. Firewall blocks it from outside.

---

## Lesson 6 - Scheduling Jobs (Gap)

> **Theory:** [Scheduling Jobs: cron, at and systemd Timers](../theory/04-processes-services/06-scheduling-jobs.md) · **Exercises:** [practice tasks](../exercises/04-processes-services/06-scheduling-jobs.md)

### cron

```bash
crontab -e            # current user
crontab -l; crontab -r
crontab -u alice -e   # as root, for alice
```
```
# m  h  dom mon dow  command
30   2  *   *   *    /usr/local/bin/backup.sh
*/15 *  *   *   1-5  /usr/local/bin/check.sh
0    0  1   *   *    root /bin/cleanup.sh     # system crontab format (/etc/cron.d/) includes USER
```

System-wide: `/etc/crontab`, `/etc/cron.d/`, `/etc/cron.{hourly,daily,weekly,monthly}/`.
Access control: `/etc/cron.allow`, `/etc/cron.deny`.

### at (one-off)

```bash
echo "/usr/local/bin/task.sh" | at 23:00
at now + 2 hours; atq; atrm 3
```

### systemd timers

```ini
# /etc/systemd/system/backup.service
[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```
```ini
# /etc/systemd/system/backup.timer
[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true

[Install]
WantedBy=timers.target
```
```bash
systemctl daemon-reload && systemctl enable --now backup.timer
systemctl list-timers
systemd-analyze calendar "Mon..Fri *-*-* 08:00"
```

---

## Lab 04

1. Start `sleep 9999` with nice 15, then change it to 5 and finally terminate it gracefully.
2. Write `/etc/systemd/system/hello.service` that runs a script writing the date to `/var/log/hello.log` every 10 s (loop), runs as user `nobody`, restarts on failure, and is enabled at boot.
3. Using a drop-in, make `cron`/`crond` restart always.
4. Schedule `/usr/local/bin/report.sh` every weekday at 07:45 for user `ana` with cron, and the same with a systemd timer.
5. Fill `/tmp` with a 1 GB file held open by a `tail -f` process, delete it, and find the space with `lsof +L1`.
6. Find the 3 processes using the most memory and save them to `/root/topmem.txt`.

## Self-check

- [ ] I can write a unit file and a timer from memory.
- [ ] I know `enable` vs `start` vs `mask` and when `daemon-reload` is needed.
- [ ] I can read cron syntax and know the `/etc/cron.d` format includes a user field.
- [ ] I can find what is filling a disk in under 2 minutes.
