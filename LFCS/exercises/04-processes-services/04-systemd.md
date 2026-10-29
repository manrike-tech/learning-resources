# Exercises – Managing Services with systemd

<!-- nav:start -->
> **Module 04 · Processes, Services, Scheduling & Performance** — lesson 4 of 6  
> Read first: [Theory – Managing Services with systemd](../../theory/04-processes-services/04-systemd.md) · [Course lesson (commands)](../../course/04-Processes-Services-Performance.md#lesson-4---systemd-ch-31) · [Module exercises](README.md)
<!-- nav:end -->

> The cron service is `cron` on Ubuntu and `crond` on RHEL; SSH is `ssh` on Ubuntu and `sshd` on RHEL. Adapt names below.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Read a status ★

For the SSH service, record in `/root/ssh-status.txt`: unit file path, enabled state, active state, main PID and memory used.

<details>
<summary>Solution</summary>

```bash
systemctl status sshd --no-pager > /root/ssh-status.txt     # ssh on Ubuntu
systemctl show sshd -p FragmentPath,UnitFileState,ActiveState,MainPID,MemoryCurrent >> /root/ssh-status.txt
```

</details>

---

## Exercise 2 – start/enable/mask ★

1. Stop and disable cron. Check `is-active` and `is-enabled`.
2. Enable and start it in one command; find the symlink created.
3. Mask it and try to start it. Unmask and start it.

<details>
<summary>Solution</summary>

```bash
systemctl disable --now crond; systemctl is-active crond; systemctl is-enabled crond
systemctl enable --now crond          # prints "Created symlink .../multi-user.target.wants/crond.service"
systemctl mask crond; systemctl start crond   # "Unit crond.service is masked."
systemctl unmask crond; systemctl start crond
```

</details>

---

## Exercise 3 – Write a service ★★

Create `hello.service` that:
- runs `/usr/local/bin/hello.sh`, a loop appending the date to `/var/log/hello.log` every 10 s,
- runs as user `nobody` (make the log writable for it),
- restarts on failure after 3 s,
- starts at boot.

**Verify:** `systemctl is-enabled hello` → enabled; `tail -2 /var/log/hello.log` shows new lines; `ps -o user= -C hello.sh` → nobody.

<details>
<summary>Solution</summary>

```bash
cat > /usr/local/bin/hello.sh << 'EOF'
#!/bin/bash
while true; do date >> /var/log/hello.log; sleep 10; done
EOF
chmod 755 /usr/local/bin/hello.sh
touch /var/log/hello.log && chown nobody /var/log/hello.log
cat > /etc/systemd/system/hello.service << 'EOF'
[Unit]
Description=Hello logger
After=network.target

[Service]
Type=simple
User=nobody
ExecStart=/usr/local/bin/hello.sh
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload && systemctl enable --now hello
```

</details>

---

## Exercise 4 – Restart behaviour ★★

1. Kill hello's main process with `kill -9`. Does systemd restart it? Watch `journalctl -u hello -f`.
2. Stop it with `kill -15`. Does it restart? Why?
3. Change the policy so it always restarts, using a drop-in.

<details>
<summary>Solution</summary>

1. Yes – killed by a signal other than a clean stop counts as failure for `on-failure`.
2. No – SIGTERM is considered a clean exit for `on-failure`.

```bash
systemctl edit hello      # add: [Service]\nRestart=always
systemctl show hello -p Restart
```

</details>

---

## Exercise 5 – Override a vendor unit ★★

Using a drop-in, make cron restart always and add the environment variable `MAILTO=root` to it. Show the effective unit with all drop-ins.

<details>
<summary>Solution</summary>

```bash
systemctl edit crond
# [Service]
# Restart=always
# Environment=MAILTO=root
systemctl cat crond
systemctl restart crond
systemctl show crond -p Restart -p Environment
```

</details>

---

## Exercise 6 – Fix a broken unit ★★

Create this unit, try to start it, read the errors and fix all three problems:

```ini
# /etc/systemd/system/broken.service
[Unit]
Description=Broken demo

[Service]
User=svcdemo
ExecStart=python3 -m http.server 8088

[Install]
WantedBy=multi-user.target
```

<details>
<summary>Solution</summary>

1. Missing `daemon-reload` after creating the file.
2. `ExecStart=` must be an absolute path: `/usr/bin/python3 -m http.server 8088` (older systemd versions; newer ones search PATH but absolute paths are still best practice).
3. User `svcdemo` doesn't exist: `useradd -r -s /usr/sbin/nologin svcdemo`.

`journalctl -xeu broken` shows "Failed to determine user credentials".

</details>

---

## Exercise 7 – Targets ★

1. Show the default target. Set it to `multi-user.target`.
2. List the units that `multi-user.target` wants.
3. Show the 5 slowest units at boot.

<details>
<summary>Solution</summary>

```bash
systemctl get-default; systemctl set-default multi-user.target
systemctl list-dependencies multi-user.target | head -20
systemd-analyze blame | head -5
```

</details>

---

## Exercise 8 – Journal ★★

1. Show only error-or-worse messages of the current boot.
2. Show hello's logs from the last 10 minutes.
3. Make the journal persistent; verify with `journalctl --list-boots` after a reboot.

<details>
<summary>Solution</summary>

```bash
journalctl -p err -b
journalctl -u hello --since "10 min ago"
mkdir -p /var/log/journal && systemctl restart systemd-journald
```

</details>

---

## Challenge – Ordered dependency ★★★

Create two services: `backend.service` (`/usr/bin/python3 -m http.server 9000`, runs in `/srv/backend`) and `frontend.service` (a script that runs `curl -fsS localhost:9000` every 5 s and logs the result). Configure them so that:

- starting `frontend` also starts `backend`, and `frontend` starts **after** it,
- stopping `backend` also stops `frontend`,
- only `frontend` needs to be enabled for both to start at boot.

<details>
<summary>Solution</summary>

```ini
# /etc/systemd/system/backend.service
[Unit]
Description=Backend
[Service]
WorkingDirectory=/srv/backend
ExecStart=/usr/bin/python3 -m http.server 9000
```
```ini
# /etc/systemd/system/frontend.service
[Unit]
Description=Frontend
Requires=backend.service
After=backend.service
[Service]
ExecStart=/bin/bash -c 'while true; do curl -fsS -o /dev/null localhost:9000 && echo ok || echo fail; sleep 5; done'
[Install]
WantedBy=multi-user.target
```

```bash
mkdir -p /srv/backend; systemctl daemon-reload; systemctl enable --now frontend
systemctl status backend; systemctl stop backend; systemctl is-active frontend   # inactive
```

</details>

---

## Cleanup

```bash
systemctl disable --now hello frontend backend broken 2>/dev/null
rm -rf /etc/systemd/system/{hello,broken,frontend,backend}.service /etc/systemd/system/hello.service.d
systemctl revert crond 2>/dev/null; systemctl daemon-reload
```

---

<!-- footer:start -->
[← Exercises: Troubleshooting Disk Space](03-disk-space.md) · [Module exercises](README.md) · [Exercises: Application and Service Constraints →](05-service-constraints.md)
<!-- footer:end -->
