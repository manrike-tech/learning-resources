# Scheduling Jobs: cron, at and systemd Timers

<!-- nav:start -->
> **Module 04 · Processes, Services, Scheduling & Performance** — lesson 6 of 6  
> [Module index](README.md) · [Course lesson (commands)](../../course/04-Processes-Services-Performance.md#lesson-6---scheduling-jobs-gap) · [Exercises for this lesson](../../exercises/04-processes-services/06-scheduling-jobs.md)
<!-- nav:end -->

Backups at night, log cleanup every hour, a report every Monday morning, a one-off reboot at 23:00: administrators schedule work all the time. Linux offers three tools – **cron** for recurring jobs, **at** for one-time jobs, and **systemd timers**, the modern and more powerful alternative. In this lesson you will learn all three, their syntax and their pitfalls.

## What you will learn

- The cron daemon, user crontabs and system crontabs
- Cron time syntax, special strings and common examples
- Access control with `cron.allow` / `cron.deny`
- The cron environment and why "it works in my shell but not in cron"
- anacron for machines that are not always on
- One-time jobs with `at`
- systemd timers: realtime and monotonic, `Persistent=`, testing calendar expressions
- How to choose between them

---

## Key terms

| Term | Definition |
|------|-----------|
| **cron** | Daemon (`cron` on Debian, `crond` on RHEL) that runs commands on a schedule, checking every minute. |
| **crontab** | A table of scheduled jobs; also the command to edit it. |
| **User crontab** | Per-user jobs edited with `crontab -e`; no user field. Stored in `/var/spool/cron/`. |
| **System crontab** | `/etc/crontab` and `/etc/cron.d/*`; have an extra **user** field. |
| **anacron** | Runs daily/weekly/monthly jobs that were missed while the machine was off. |
| **at** | Runs a job **once** at a given time (daemon `atd`). |
| **systemd timer** | A `.timer` unit that activates a `.service` unit on a schedule. |
| **Realtime timer** | Fires at calendar times (`OnCalendar=`). |
| **Monotonic timer** | Fires relative to an event (`OnBootSec=`, `OnUnitActiveSec=`). |

---

## cron

### Time fields

```text
┌───────────── minute        (0–59)
│ ┌─────────── hour          (0–23)
│ │ ┌───────── day of month  (1–31)
│ │ │ ┌─────── month         (1–12 or jan–dec)
│ │ │ │ ┌───── day of week   (0–7, 0 and 7 = Sunday, or sun–sat)
│ │ │ │ │
* * * * *  command
```

| Syntax | Meaning | Example |
|--------|---------|---------|
| `*` | every value | `* * * * *` every minute |
| `a,b` | list | `0 8,12,18 * * *` at 08:00, 12:00, 18:00 |
| `a-b` | range | `0 9 * * 1-5` 09:00 on weekdays |
| `*/n` | every n | `*/15 * * * *` every 15 minutes |
| `a-b/n` | every n within range | `0 8-18/2 * * *` every 2 hours from 8 to 18 |

| Example | Runs |
|---------|------|
| `30 2 * * *` | every day at 02:30 |
| `0 0 1 * *` | at midnight on the 1st of every month |
| `45 7 * * 1-5` | weekdays at 07:45 |
| `10 3 * * 0` | Sundays at 03:10 |
| `0 */6 * * *` | every 6 hours, on the hour |
| `0 0 1 1 *` | once a year, Jan 1st |

> **Day-of-month and day-of-week:** if **both** are restricted (not `*`), the job runs when **either** matches. `0 0 1 * 1` runs on the 1st of the month **and** on every Monday.

Special strings: `@reboot`, `@hourly`, `@daily`, `@weekly`, `@monthly`, `@yearly`.

### User crontabs

```console
$ crontab -e                     # edit your crontab (uses $EDITOR)
$ crontab -l                     # list
$ crontab -r                     # remove ALL your jobs (careful!)
$ sudo crontab -u alice -e       # root edits alice's crontab
$ sudo crontab -u alice -l
```

```text
# m  h  dom mon dow  command
30   2  *   *   *    /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
*/5  *  *   *   *    /usr/local/bin/check-disk.sh
```

The files are stored in `/var/spool/cron/crontabs/<user>` (Debian) or `/var/spool/cron/<user>` (RHEL) – never edit them directly; `crontab -e` checks syntax and notifies the daemon.

### System crontabs

`/etc/crontab` and files in `/etc/cron.d/` have a **sixth field: the user** to run as.

```text
# /etc/cron.d/cleanup
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=admin@example.lab
# m  h  dom mon dow  user   command
0    4  *   *   *    root   /usr/local/bin/cleanup.sh
```

Drop-in directories for scripts (run via `run-parts`, by cron or anacron):

```text
/etc/cron.hourly/   /etc/cron.daily/   /etc/cron.weekly/   /etc/cron.monthly/
```

Scripts there must be executable and – on Debian – have names without dots (`backup`, not `backup.sh`). Test with `run-parts --test /etc/cron.daily`.

### Access control

| File | Effect |
|------|--------|
| `/etc/cron.allow` exists | **only** listed users may use `crontab` |
| only `/etc/cron.deny` exists | everyone **except** listed users |
| neither | Debian: everyone; RHEL: only root (an empty `cron.deny` exists by default, so everyone) |

Root can always use cron. The same logic applies to `at` with `/etc/at.allow` and `/etc/at.deny`.

### The cron environment

Cron runs commands with a **minimal environment**: `PATH=/usr/bin:/bin`, `SHELL=/bin/sh`, `HOME` of the user, no `.bashrc`. This explains most "works in my terminal, not in cron" problems.

- Use **absolute paths** for commands and files, or set `PATH=` at the top of the crontab.
- `%` has a special meaning (newline/stdin) in crontab lines – escape it: `date +\%F`.
- Output is mailed to the user (`MAILTO=`) if a mail system exists; otherwise redirect it to a log: `>> /var/log/job.log 2>&1`.
- Check whether jobs ran: `journalctl -u cron` (Debian) / `journalctl -u crond` or `/var/log/cron` (RHEL).

```console
$ sudo grep CRON /var/log/syslog | tail -2        # Debian/Ubuntu
Oct 21 02:30:01 web01 CRON[5521]: (root) CMD (/usr/local/bin/backup.sh >> /var/log/backup.log 2>&1)
```

---

## anacron

Cron simply skips jobs whose time passed while the machine was off. **anacron** runs the daily/weekly/monthly jobs later, once the system is up. It's configured in `/etc/anacrontab`:

```text
# period(days) delay(min) job-id        command
1              5          cron.daily    run-parts /etc/cron.daily
7              10         cron.weekly   run-parts /etc/cron.weekly
@monthly       15         cron.monthly  run-parts /etc/cron.monthly
```

Timestamps of the last runs are kept in `/var/spool/anacron/`. On RHEL, `/etc/cron.daily` etc. are run through anacron by default.

---

## at: one-time jobs

```console
$ sudo systemctl enable --now atd              # package "at"
$ echo "/usr/local/bin/report.sh" | at 23:00
job 3 at Tue Oct 21 23:00:00 2026
$ at now + 2 hours
at> systemctl restart nginx
at> <Ctrl+D>
$ at 09:00 tomorrow -f /root/task.sh           # run a script file
$ at 14:30 2026-12-24
$ atq                                          # list pending jobs (as root: everyone's)
3       Tue Oct 21 23:00:00 2026 a root
$ at -c 3 | tail -3                            # show the job's content
$ atrm 3                                       # delete job 3
$ batch                                        # run when load average drops below 1.5
```

Unlike cron, `at` saves the **current environment** and working directory with the job.

---

## systemd timers

A timer is a `.timer` unit that starts a `.service` unit (by default the service with the same name).

### Example: daily backup at 02:30

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Nightly backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
User=root
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup daily at 02:30

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true
RandomizedDelaySec=5min

[Install]
WantedBy=timers.target
```

```console
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now backup.timer      # enable the TIMER, not the service
$ systemctl list-timers
NEXT                        LEFT     LAST  PASSED  UNIT          ACTIVATES
Wed 2026-10-22 02:33:12 UTC 16h left -     -       backup.timer  backup.service
$ sudo systemctl start backup.service           # run the job now, to test it
$ journalctl -u backup.service                  # its output and exit status
```

### OnCalendar syntax

Format: `DayOfWeek Year-Month-Day Hour:Minute:Second`

| Expression | Meaning |
|-----------|---------|
| `daily` | `*-*-* 00:00:00` |
| `hourly`, `weekly`, `monthly` | shortcuts |
| `*-*-* 02:30:00` | every day 02:30 |
| `Mon..Fri *-*-* 07:45` | weekdays 07:45 |
| `Sun *-*-* 03:10` | Sundays 03:10 |
| `*-*-01 00:00` | 1st of each month |
| `*:0/15` | every 15 minutes |
| `*-*-* 08..18:00` | every hour from 8 to 18 |

Always test an expression:

```console
$ systemd-analyze calendar "Mon..Fri *-*-* 07:45"
  Original form: Mon..Fri *-*-* 07:45
Normalized form: Mon..Fri *-*-* 07:45:00
    Next elapse: Wed 2026-10-22 07:45:00 UTC
       From now: 21h left
```

### Monotonic timers

```ini
[Timer]
OnBootSec=10min          # 10 minutes after boot
OnUnitActiveSec=1h       # then every hour after the service last ran
```

### Useful [Timer] options

| Option | Meaning |
|--------|---------|
| `Persistent=true` | if the machine was off at the scheduled time, run at next boot (like anacron) |
| `RandomizedDelaySec=` | spread the start randomly (avoid all servers hitting a backup server at once) |
| `AccuracySec=` | allowed lateness (default 1 min) |
| `Unit=` | activate a service with a different name |

Quick one-off with a transient timer: `systemd-run --on-calendar="*-*-* 23:00" /usr/local/bin/task.sh` or `systemd-run --on-active=30m …`.

---

## Choosing a tool

| | cron | at | systemd timer |
|--|------|----|---------------|
| Recurring | ✔ | – | ✔ |
| One-time | – | ✔ | ✔ (`systemd-run --on-calendar`) |
| Missed runs caught up | anacron only | n/a | `Persistent=true` |
| Logging | syslog line; output by mail/redirect | mail | full output in the journal |
| Dependencies, limits, sandboxing | – | – | all systemd features |
| Simplicity | one line | one line | two files |

On the exam, use whatever the task asks for – it often names the tool explicitly.

---

## Try it yourself

1. Add a user crontab entry that writes `date` to `/tmp/cron-test.log` every minute (escape `%` if you format the date). Check it after 2 minutes and in the logs.
2. Create `/etc/cron.d/report` running `/usr/local/bin/report.sh` as user `alice` every weekday at 07:45.
3. Deny cron to user `bob` and test `crontab -e` as bob.
4. Schedule `touch /tmp/at-test` in 2 minutes with `at`; list and inspect it.
5. Convert the cron job from step 2 into a systemd timer; verify with `list-timers` and test with `systemd-analyze calendar`.

---

## Common mistakes

- Forgetting the user field in `/etc/cron.d/` files (or adding one in `crontab -e`).
- Relying on `PATH` or aliases in cron jobs.
- Unescaped `%` in crontab commands.
- Enabling the `.service` instead of the `.timer`.
- Forgetting `daemon-reload` after creating timer/service files.
- Expecting cron to run jobs missed while the server was off.

---

## Summary

| Task | Command / file |
|------|----------------|
| Edit user jobs | `crontab -e`, `crontab -u user -e` |
| System job (with user field) | `/etc/cron.d/name`, `/etc/crontab` |
| Daily/weekly scripts | `/etc/cron.daily/` etc. |
| Restrict cron | `/etc/cron.allow`, `/etc/cron.deny` |
| One-time job | `at TIME`, `atq`, `atrm` |
| Timer | `name.service` + `name.timer` → `enable --now name.timer` |
| Test calendar | `systemd-analyze calendar "…"` |
| List timers | `systemctl list-timers` |

## Check yourself

1. Write a cron line that runs `/opt/x.sh` every 10 minutes between 08:00 and 17:59 on weekdays.
2. What is the difference between a line in `crontab -e` and a line in `/etc/cron.d/x`?
3. A server was off at 02:30; how do you make a systemd timer still run the job after boot?
4. Why does `0 0 13 * 5` run more often than "Friday the 13th"?

<details>
<summary>Answers</summary>

1. `*/10 8-17 * * 1-5 /opt/x.sh`
2. `/etc/cron.d/` lines have a user field after the five time fields; user crontabs do not.
3. `Persistent=true` in the `[Timer]` section.
4. When both day-of-month and day-of-week are restricted, cron runs if **either** matches: every 13th **and** every Friday.

</details>

## Further reading

- `man 5 crontab`, `man 8 cron`, `man 5 anacrontab`, `man 1 at`, `man 5 systemd.timer`, `man 7 systemd.time`
- [crontab.guru](https://crontab.guru/) – cron expression editor
- [ArchWiki: cron](https://wiki.archlinux.org/title/Cron) · [systemd/Timers](https://wiki.archlinux.org/title/Systemd/Timers)

---

<!-- footer:start -->
[← Application and Service Constraints](05-service-constraints.md) · [Module index](README.md)  
**Practice now:** [Exercises – Scheduling Jobs: cron, at and systemd Timers](../../exercises/04-processes-services/06-scheduling-jobs.md)
<!-- footer:end -->
