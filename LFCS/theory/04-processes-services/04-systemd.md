# Managing Services with systemd

<!-- nav:start -->
> **Module 04 · Processes, Services, Scheduling & Performance** — lesson 4 of 6  
> [Module index](README.md) · [Course lesson (commands)](../../course/04-Processes-Services-Performance.md#lesson-4---systemd-ch-31) · [Exercises for this lesson](../../exercises/04-processes-services/04-systemd.md)
<!-- nav:end -->

**systemd** is the init system and service manager of every major distribution. As PID 1 it starts the system at boot, launches and supervises services, restarts them when they fail, collects their logs and shuts everything down cleanly. In this lesson you will learn the systemd model (units, dependencies, targets), how to control services with `systemctl`, how to write your own service, how to customize vendor units safely, and how to read logs with `journalctl`.

## What you will learn

- What units are, the main unit types and where unit files live
- `systemctl`: start, stop, restart, reload, enable, disable, mask, status
- Writing a service unit: `[Unit]`, `[Service]`, `[Install]`
- Service types and restart policies
- Dependencies vs ordering (`Wants`/`Requires` vs `After`/`Before`)
- Drop-in overrides with `systemctl edit`
- Targets (the replacement for runlevels) and the boot process
- The journal: `journalctl` filters and persistent storage

---

## Key terms

| Term | Definition |
|------|-----------|
| **Unit** | An object systemd manages, described by a unit file: a service, socket, timer, mount, target… |
| **Unit file** | An INI-style text file defining a unit (`nginx.service`). |
| **Service** | A unit that runs and supervises one or more processes (a daemon or a one-shot task). |
| **Target** | A unit that groups other units; used as a synchronization point (e.g. `multi-user.target`). |
| **Enable** | Configure a unit to start automatically (at boot), by creating symlinks from its `[Install]` section. |
| **Start** | Run it **now**. Enable and start are independent. |
| **Mask** | Link the unit to `/dev/null` so it cannot be started at all, not even as a dependency. |
| **Drop-in** | A `*.conf` file in `<unit>.d/` that overrides or extends part of a unit file. |
| **Journal** | systemd's log store, managed by `systemd-journald`, read with `journalctl`. |
| **cgroup** | Kernel control group; systemd puts every service in its own cgroup to track and limit all its processes. |

---

## Units and unit files

| Type | Suffix | Represents |
|------|--------|-----------|
| Service | `.service` | a daemon or a task |
| Socket | `.socket` | a listening socket; starts the service on first connection |
| Timer | `.timer` | scheduled activation (see [Scheduling Jobs](06-scheduling-jobs.md)) |
| Target | `.target` | a group of units / a system state |
| Mount / Automount | `.mount` / `.automount` | a filesystem mount (generated from `/etc/fstab`) |
| Path | `.path` | watches a file or directory and starts a unit on change |
| Slice / Scope | `.slice` / `.scope` | cgroups for resource control |
| Device | `.device` | a kernel device (from udev) |

### Where unit files live

```text
/usr/lib/systemd/system/    ← installed by packages (Debian: also /lib/systemd/system/). Do NOT edit.
/run/systemd/system/        ← runtime units, gone after reboot
/etc/systemd/system/        ← YOUR units and overrides. Highest priority.
~/.config/systemd/user/     ← per-user units (systemctl --user)
```

A file in `/etc/systemd/system/` with the same name **replaces** the vendor file. A directory `/etc/systemd/system/<unit>.d/*.conf` **extends** it (drop-in).

```console
$ systemctl cat sshd           # shows the effective unit file plus all drop-ins
# /usr/lib/systemd/system/sshd.service
[Unit]
Description=OpenSSH server daemon
After=network.target sshd-keygen.target
...
$ systemctl show sshd -p Restart,ExecStart,MainPID   # resolved properties
```

---

## Controlling services with systemctl

```console
$ systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since Tue 2026-10-21 09:12:01 UTC; 1h 3min ago
   Main PID: 905 (nginx)
      Tasks: 3 (limit: 11099)
     Memory: 4.2M
        CPU: 62ms
     CGroup: /system.slice/nginx.service
             ├─905 "nginx: master process /usr/sbin/nginx"
             └─906 "nginx: worker process"
Oct 21 09:12:01 web01 systemd[1]: Started A high performance web server...
```

Reading it: **Loaded** tells you where the unit file is and whether it's *enabled*; **Active** whether it runs now; the last lines are recent journal entries.

| Command | Effect |
|---------|--------|
| `systemctl start nginx` | start now |
| `systemctl stop nginx` | stop now |
| `systemctl restart nginx` | stop + start |
| `systemctl reload nginx` | ask it to reload config without stopping (if supported) |
| `systemctl reload-or-restart nginx` | reload if possible, else restart |
| `systemctl enable nginx` | start at boot (creates symlinks) |
| `systemctl enable --now nginx` | enable **and** start |
| `systemctl disable --now nginx` | disable and stop |
| `systemctl mask nginx` | make it impossible to start |
| `systemctl unmask nginx` | undo mask |
| `systemctl is-active nginx` | prints `active`/`inactive`/`failed` (exit code usable in scripts) |
| `systemctl is-enabled nginx` | `enabled`/`disabled`/`masked`/`static` |
| `systemctl is-failed nginx` | is it failed? |
| `systemctl daemon-reload` | re-read unit files after you edit them |
| `systemctl kill -s HUP nginx` | send a signal to the service's processes |

### Listing

```console
$ systemctl list-units --type=service                 # loaded and active
$ systemctl list-units --type=service --state=failed  # failed ones (also: systemctl --failed)
$ systemctl list-unit-files --type=service            # all installed, with enabled state
$ systemctl list-dependencies multi-user.target
```

### What enable really does

```console
$ sudo systemctl enable nginx
Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /usr/lib/systemd/system/nginx.service.
```

It reads `[Install] WantedBy=multi-user.target` and creates a symlink in that target's `.wants/` directory. When `multi-user.target` starts at boot, it pulls in everything in `.wants/`. A unit without an `[Install]` section shows as `static`: it can't be enabled, it only runs when something else needs it.

```console
$ sudo systemctl mask nginx
Created symlink /etc/systemd/system/nginx.service → /dev/null.
$ sudo systemctl start nginx
Failed to start nginx.service: Unit nginx.service is masked.
```

---

## Writing a service unit

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Python web application
Documentation=https://example.lab/myapp
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
Environment=PORT=8080
EnvironmentFile=-/etc/myapp/env
ExecStartPre=/opt/myapp/check-config.sh
ExecStart=/usr/bin/python3 /opt/myapp/app.py
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
LimitNOFILE=8192
MemoryMax=512M

[Install]
WantedBy=multi-user.target
```

```console
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now myapp
$ systemctl status myapp
```

### [Unit] section

| Directive | Meaning |
|-----------|---------|
| `Description=` | human-readable name shown in status and logs |
| `After=` / `Before=` | **ordering**: start after/before these units (if both are being started) |
| `Wants=` | weak dependency: also start these; don't fail if they fail |
| `Requires=` | strong dependency: if it fails to start or is stopped, so is this unit |
| `BindsTo=` | even stronger: stop when the other stops |
| `Conflicts=` | cannot run together |
| `ConditionPathExists=` | skip silently if the condition is false |

> **Requirement does not imply order.** `Requires=postgresql.service` alone starts both **in parallel**. You almost always want `Requires=` (or `Wants=`) **plus** `After=` on the same unit.

### [Service] section

| Directive | Meaning |
|-----------|---------|
| `Type=` | how systemd knows the service has started (below) |
| `ExecStart=` | the command (absolute path) |
| `ExecStartPre=` / `ExecStartPost=` | commands before/after start |
| `ExecReload=` / `ExecStop=` | how to reload / stop (default stop: SIGTERM, then SIGKILL after `TimeoutStopSec`) |
| `User=` / `Group=` | run as an unprivileged account |
| `WorkingDirectory=` | current directory |
| `Environment=` / `EnvironmentFile=` | variables (`-` prefix: file is optional) |
| `Restart=` | `no`, `on-failure`, `on-abnormal`, `always` |
| `RestartSec=` | wait before restarting |
| `Limit*=`, `MemoryMax=`, `CPUQuota=` | resource limits (see [Resource Limits](../02-users-groups-permissions/07-resource-limits.md)) |
| `UMask=` | file creation mask |
| `ProtectSystem=`, `PrivateTmp=`, `NoNewPrivileges=` | hardening |

### Service types

| `Type=` | Use when | systemd considers it started… |
|---------|----------|-------------------------------|
| `simple` (default) | the program runs in the foreground | immediately after fork |
| `exec` | like simple, but reports exec failures | after the binary is executed |
| `forking` | traditional daemons that fork into the background | when the parent exits (set `PIDFile=`) |
| `oneshot` | a task that runs and exits (setup scripts) | when it exits; add `RemainAfterExit=yes` to stay "active" |
| `notify` | program supports sd_notify | when the program says it's ready |

A common mistake is `Type=simple` with a program that daemonizes itself: systemd thinks it exited and stops (or restarts) it. Run the program in the foreground (often a `-D`, `-f` or `--foreground` option) or use `Type=forking`.

### [Install] section

| Directive | Meaning |
|-----------|---------|
| `WantedBy=multi-user.target` | `enable` links it into the normal boot target |
| `WantedBy=graphical.target` | desktop systems |
| `Alias=` | extra names |

---

## Customizing vendor units: drop-ins

Never edit files in `/usr/lib/systemd/system/` – package updates overwrite them. Use a drop-in:

```console
$ sudo systemctl edit nginx
```

An editor opens `/etc/systemd/system/nginx.service.d/override.conf`. Add only what you change:

```ini
[Service]
Restart=always
RestartSec=3
```

`systemctl edit` runs `daemon-reload` for you when you save. Check the result with `systemctl cat nginx` and `systemctl show nginx -p Restart`. Then restart the service.

Two rules for drop-ins:

- **List settings** like `ExecStart=` must be **cleared** first with an empty assignment, otherwise you add a second value:
  ```ini
  [Service]
  ExecStart=
  ExecStart=/usr/sbin/nginx -g 'daemon off;' -c /etc/nginx/custom.conf
  ```
- `systemctl edit --full nginx` copies the whole unit to `/etc/systemd/system/` for you to edit – it then no longer follows vendor updates.

`systemctl revert nginx` removes all your overrides.

---

## Targets and boot

Targets group units into system states. They replace SysV runlevels:

| Runlevel | Target | State |
|----------|--------|-------|
| 0 | `poweroff.target` | shut down |
| 1 | `rescue.target` | single user, basic system |
| 3 | `multi-user.target` | normal server, no GUI |
| 5 | `graphical.target` | GUI |
| 6 | `reboot.target` | reboot |
| – | `emergency.target` | minimal shell, root read-only |

```console
$ systemctl get-default
multi-user.target
$ sudo systemctl set-default multi-user.target    # persistent default
$ sudo systemctl isolate rescue.target            # switch now (stops everything not needed)
$ systemctl list-units --type=target
```

Boot analysis:

```console
$ systemd-analyze                    # total boot time
Startup finished in 1.9s (kernel) + 3.2s (initrd) + 8.4s (userspace) = 13.5s
$ systemd-analyze blame | head -5    # slowest units
$ systemd-analyze critical-chain     # what delayed reaching the default target
$ systemd-analyze verify /etc/systemd/system/myapp.service   # lint a unit file
```

Power: `systemctl reboot`, `systemctl poweroff`, `systemctl suspend`.

---

## Logs with journalctl

`systemd-journald` captures the stdout/stderr of every service, kernel messages and syslog messages, with metadata (unit, PID, priority).

```console
$ journalctl -u nginx                      # one unit
$ journalctl -u nginx -f                   # follow (like tail -f)
$ journalctl -u nginx -n 50 --no-pager     # last 50 lines
$ journalctl -u nginx --since "1 hour ago"
$ journalctl --since "2026-10-21 09:00" --until "2026-10-21 10:00"
$ journalctl -p err -b                     # priority error and worse, current boot
$ journalctl -b -1                         # previous boot
$ journalctl --list-boots
$ journalctl -k                            # kernel messages (like dmesg)
$ journalctl _PID=905                      # by field
$ journalctl -u sshd -o json-pretty -n 1   # all fields of an entry
$ journalctl -xeu myapp                    # jump to end, with explanations – typical after a failed start
```

Priorities: `0 emerg`, `1 alert`, `2 crit`, `3 err`, `4 warning`, `5 notice`, `6 info`, `7 debug`.

### Persistent journal

On some systems the journal is kept only in RAM (`/run/log/journal`) and lost at reboot. Make it persistent:

```console
$ sudo mkdir -p /var/log/journal
$ sudo systemctl restart systemd-journald
```

or set `Storage=persistent` in `/etc/systemd/journald.conf`. Limit its size with `SystemMaxUse=500M`.

Many systems also run **rsyslog**, which writes classic text logs (`/var/log/syslog` on Debian, `/var/log/messages` and `/var/log/secure` on RHEL). Send your own messages with `logger -p user.warning "test message"`.

---

## Example: troubleshoot a failing service

```console
$ sudo systemctl start myapp
Job for myapp.service failed because the control process exited with error code.
See "systemctl status myapp.service" and "journalctl -xeu myapp.service" for details.
$ journalctl -xeu myapp --no-pager | tail -5
myapp.service: Failed to determine user credentials: No such process
myapp.service: Failed at step USER spawning /usr/bin/python3: No such process
```

The `User=myapp` account doesn't exist → `useradd -r -s /usr/sbin/nologin myapp`, then `systemctl restart myapp`. The full checklist is in [Service Constraints](05-service-constraints.md).

---

## Try it yourself

1. Look at `systemctl status` of `sshd`/`ssh` and identify the unit file path, enabled state and main PID.
2. Disable and stop `cron`/`crond`, check `is-enabled`, then `enable --now` it again. Look at the symlink created.
3. Mask `cron`/`crond`, try to start it, unmask it.
4. Write a `hello.service` that runs `/usr/local/bin/hello.sh` (a loop writing the date to a log every 10 s) as user `nobody`, restarts on failure, and starts at boot.
5. Use `systemctl edit` to set `Restart=always` on your service; check with `systemctl show`.
6. Kill the service's main process with `kill -9` and watch systemd restart it in `journalctl -u hello -f`.

---

## Common mistakes

- Editing unit files and forgetting `daemon-reload` (systemd warns: "unit file changed on disk").
- Confusing `enable` (boot) with `start` (now) – exam tasks usually need both.
- Editing `/usr/lib/systemd/system/…` instead of using a drop-in.
- Adding `ExecStart=` in a drop-in without clearing it first.
- `Type=simple` for a forking daemon.
- Relative paths in `ExecStart=` – they must be absolute.

---

## Summary

| Task | Command |
|------|---------|
| Status / logs | `systemctl status unit`, `journalctl -u unit` |
| Now / at boot | `start`/`stop`/`restart`/`reload` · `enable`/`disable` · `enable --now` |
| Prevent | `mask` / `unmask` |
| New unit | file in `/etc/systemd/system/` → `daemon-reload` → `enable --now` |
| Override | `systemctl edit unit` → restart; `systemctl cat unit` |
| Default target | `get-default`, `set-default`, `isolate` |
| Failed units | `systemctl --failed`, `journalctl -xeu unit` |
| Persistent journal | `mkdir /var/log/journal` or `Storage=persistent` |

## Check yourself

1. What is the difference between `systemctl enable` and `systemctl start`?
2. Why would `Requires=db.service` alone not guarantee the database is up before your app starts?
3. A drop-in contains `ExecStart=/new/path`. Why does the service fail with "has more than one ExecStart= setting"?
4. Where do you put a unit file you wrote yourself?
5. How do you see the logs of the previous boot?

<details>
<summary>Answers</summary>

1. `enable` creates the symlinks so the unit starts at boot; `start` runs it now.
2. `Requires=` only says both must run; without `After=db.service` they start in parallel.
3. `ExecStart=` is a list; you must clear it with an empty `ExecStart=` line before setting the new value.
4. `/etc/systemd/system/`.
5. `journalctl -b -1` (requires a persistent journal).

</details>

## Further reading

- `man systemctl`, `man systemd.unit`, `man systemd.service`, `man systemd.exec`, `man systemd.special`, `man journalctl`, `man systemd.directives`
- [systemd for Administrators (blog series by Lennart Poettering)](https://www.freedesktop.org/wiki/Software/systemd/)
- [ArchWiki: systemd](https://wiki.archlinux.org/title/Systemd) · [systemd/Journal](https://wiki.archlinux.org/title/Systemd/Journal)
- [Red Hat: Using systemd unit files to customize and optimize your system (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_systemd_unit_files_to_customize_and_optimize_your_system/index)
- [DigitalOcean: Understanding systemd units and unit files](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files)

---

<!-- footer:start -->
[← Troubleshooting Disk Space](03-disk-space.md) · [Module index](README.md) · [Application and Service Constraints →](05-service-constraints.md)  
**Practice now:** [Exercises – Managing Services with systemd](../../exercises/04-processes-services/04-systemd.md)
<!-- footer:end -->
