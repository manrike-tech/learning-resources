# Exercises – Scheduling Jobs

<!-- nav:start -->
> **Module 04 · Processes, Services, Scheduling & Performance** — lesson 6 of 6  
> Read first: [Theory – Scheduling Jobs: cron, at and systemd Timers](../../theory/04-processes-services/06-scheduling-jobs.md) · [Course lesson (commands)](../../course/04-Processes-Services-Performance.md#lesson-6---scheduling-jobs-gap) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Setup

```bash
useradd -m -s /bin/bash ana 2>/dev/null; useradd -m -s /bin/bash bob 2>/dev/null
cat > /usr/local/bin/report.sh << 'EOF'
#!/bin/bash
echo "report $(date +%F_%T) by $(whoami)" >> /tmp/report.log
EOF
chmod 755 /usr/local/bin/report.sh
```

---

## Exercise 1 – Read cron syntax ★

Translate into words: `*/20 6-18 * * 1-5`, `0 3 1,15 * *`, `@reboot`, `5 0 * 8 *`. Then write lines for: every day at 23:59; every Sunday at 03:10; every 2 hours on the hour.

<details>
<summary>Solution</summary>

- every 20 minutes from 06:00 to 18:40 on weekdays; 03:00 on the 1st and 15th; once at boot; 00:05 every day in August.
- `59 23 * * *` · `10 3 * * 0` · `0 */2 * * *`

</details>

---

## Exercise 2 – User crontab ★

As `ana`, schedule `report.sh` every minute. After 2 minutes check `/tmp/report.log` and the cron log. Then remove only that line.

<details>
<summary>Solution</summary>

```bash
su - ana -c '(crontab -l 2>/dev/null; echo "* * * * * /usr/local/bin/report.sh") | crontab -'
sleep 125; cat /tmp/report.log
journalctl -u cron --since "-3min" 2>/dev/null || grep CROND /var/log/cron | tail
su - ana -c 'crontab -l | grep -v report.sh | crontab -'
```

</details>

---

## Exercise 3 – Exam-style cron ★★

> Run `/usr/local/bin/report.sh` every weekday at 07:45 as user `ana`.

Do it twice: once with `crontab -u`, once with a file in `/etc/cron.d/`.

<details>
<summary>Solution</summary>

```bash
crontab -u ana -e        # 45 7 * * 1-5 /usr/local/bin/report.sh
echo '45 7 * * 1-5 ana /usr/local/bin/report.sh' > /etc/cron.d/report
chmod 644 /etc/cron.d/report
```

</details>

---

## Exercise 4 – The cron environment ★★

Add a job `* * * * * echo "PATH=$PATH SHELL=$SHELL" >> /tmp/cronenv.txt` and another `* * * * * date +%F >> /tmp/crondate.txt`. Explain the results and fix the second.

<details>
<summary>Solution</summary>

`cronenv.txt` shows the minimal PATH (`/usr/bin:/bin`) and `/bin/sh`. The second job fails/truncates because `%` is special in crontab: write `date +\%F`.

</details>

---

## Exercise 5 – Access control ★

Forbid `bob` from using cron; confirm `crontab -e` as bob fails; root can still create a crontab for bob.

<details>
<summary>Solution</summary>

```bash
echo bob >> /etc/cron.deny
su - bob -c 'crontab -l'     # "You (bob) are not allowed to use this program"
crontab -u bob -l            # root still can
```

</details>

---

## Exercise 6 – at ★

1. Make sure `atd` runs. Schedule `report.sh` in 2 minutes and a reboot-warning `wall` message at 23:00 today.
2. List the jobs, show the content of one, delete the 23:00 job.

<details>
<summary>Solution</summary>

```bash
systemctl enable --now atd
echo /usr/local/bin/report.sh | at now + 2 minutes
echo 'wall "Reboot in 10 minutes"' | at 23:00
atq; at -c <id> | tail -3; atrm <id>
```

</details>

---

## Exercise 7 – systemd timer ★★

Create `report.service` + `report.timer` to run `report.sh` as `ana` every weekday at 07:45, catching up missed runs.

**Verify:** `systemctl list-timers report.timer`, `systemd-analyze calendar "Mon..Fri *-*-* 07:45"`, and `systemctl start report.service` writes a line by `ana`.

<details>
<summary>Solution</summary>

```ini
# /etc/systemd/system/report.service
[Unit]
Description=Daily report
[Service]
Type=oneshot
User=ana
ExecStart=/usr/local/bin/report.sh
```
```ini
# /etc/systemd/system/report.timer
[Unit]
Description=Weekday report timer
[Timer]
OnCalendar=Mon..Fri *-*-* 07:45:00
Persistent=true
[Install]
WantedBy=timers.target
```
```bash
systemctl daemon-reload && systemctl enable --now report.timer
```

</details>

---

## Exercise 8 – Monotonic timer ★★

Make `report.service` also run 2 minutes after every boot and then every 30 minutes, using a second timer `report-interval.timer`.

<details>
<summary>Solution</summary>

```ini
# /etc/systemd/system/report-interval.timer
[Timer]
Unit=report.service
OnBootSec=2min
OnUnitActiveSec=30min
[Install]
WantedBy=timers.target
```

</details>

---

## Challenge – Migrate cron to timers ★★★

List every job in `/etc/crontab`, `/etc/cron.d/` and all user crontabs on the system into `/root/all-cron-jobs.txt`. Pick one and convert it to a systemd timer with identical schedule, then disable the cron version.

<details>
<summary>Solution</summary>

```bash
{
  grep -Ev '^\s*(#|$)' /etc/crontab
  grep -Ev '^\s*(#|$)' /etc/cron.d/* 2>/dev/null
  for u in $(cut -d: -f1 /etc/passwd); do crontab -l -u "$u" 2>/dev/null | sed "s/^/$u: /"; done
} > /root/all-cron-jobs.txt
```

For the conversion, map each cron field to `OnCalendar=` (e.g. `45 7 * * 1-5` → `Mon..Fri *-*-* 07:45`) and check with `systemd-analyze calendar`.

</details>

---

## Cleanup

```bash
rm -f /etc/cron.d/report /tmp/report.log /tmp/cronenv.txt /tmp/crondate.txt
sed -i '/^bob$/d' /etc/cron.deny
systemctl disable --now report.timer report-interval.timer 2>/dev/null
rm -f /etc/systemd/system/report*.{service,timer}; systemctl daemon-reload
crontab -r -u ana 2>/dev/null
```

---

<!-- footer:start -->
[← Exercises: Application and Service Constraints](05-service-constraints.md) · [Module exercises](README.md)
<!-- footer:end -->
