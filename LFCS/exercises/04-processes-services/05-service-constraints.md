# Exercises – Application and Service Constraints

<!-- nav:start -->
> **Module 04 · Processes, Services, Scheduling & Performance** — lesson 5 of 6  
> Read first: [Theory – Application and Service Constraints](../../theory/04-processes-services/05-service-constraints.md) · [Course lesson (commands)](../../course/04-Processes-Services-Performance.md#lesson-5---service-constraints-gap) · [Module exercises](README.md)
<!-- nav:end -->

> Each exercise breaks nginx in one way. Install it first (`apt install -y nginx` / `dnf install -y nginx && systemctl enable --now nginx`). Diagnose **before** opening the solution, using the checklist from the theory lesson.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Syntax error ★

Introduce a typo in the nginx configuration (`server_nmae`) and restart. Find and fix it.

<details>
<summary>Solution</summary>

`systemctl status nginx` → failed; `nginx -t` names the file and line. Fix the directive, `nginx -t`, `systemctl restart nginx`.

</details>

---

## Exercise 2 – Port conflict ★

Stop nginx, run `python3 -m http.server 80 &` as root, then start nginx.

<details>
<summary>Solution</summary>

`journalctl -u nginx` → `bind() to 0.0.0.0:80 failed (98: Address already in use)`. `ss -tlnp | grep :80` shows python3. Kill it, start nginx.

</details>

---

## Exercise 3 – Local only ★★

Change nginx to `listen 127.0.0.1:80;`. Test with `curl` locally and from the other VM. Explain using `ss`.

<details>
<summary>Solution</summary>

`ss -tlnp` shows `127.0.0.1:80`; remote connections are refused regardless of the firewall. Restore `listen 80;`.

</details>

---

## Exercise 4 – File permissions ★★

Point nginx's document root to `/srv/site` with an `index.html`, and set `chmod 700 /srv`. Request the page.

<details>
<summary>Solution</summary>

403 Forbidden. Error log: `open() "/srv/site/index.html" failed (13: Permission denied)`. `sudo -u www-data cat /srv/site/index.html` (RHEL: `nginx`) fails; `namei -l /srv/site/index.html` shows `/srv` without `x` for others. Fix: `chmod 755 /srv`.

(On RHEL, also SELinux: see Exercise 6.)

</details>

---

## Exercise 5 – systemd sandboxing ★★

Add a drop-in to nginx with `ProtectHome=yes`, set the document root to `/home/webuser/site` (readable by everyone), and restart.

<details>
<summary>Solution</summary>

403/404 although `namei -l` looks fine. `systemctl show nginx -p ProtectHome` → yes: `/home` is invisible to the service. Remove the drop-in (or move the site to `/srv`).

</details>

---

## Exercise 6 – SELinux port and labels (RHEL) ★★★

1. Configure nginx to listen on port 8099 and restart.
2. Then serve `/srv/site` (created with `mkdir` and `mv` of a file from `/root`).

<details>
<summary>Solution</summary>

1. `bind() to 0.0.0.0:8099 failed (13: Permission denied)`; `ausearch -m AVC -ts recent` → `name_bind` denied. Fix: `semanage port -a -t http_port_t -p tcp 8099`.
2. 403; AVC shows `tcontext=…admin_home_t` or `default_t`. Fix: `semanage fcontext -a -t httpd_sys_content_t "/srv/site(/.*)?"; restorecon -Rv /srv/site`.

</details>

---

## Exercise 7 – Open files limit ★★

Set `LimitNOFILE=64` for nginx and `worker_connections 1024;` in nginx.conf; restart and generate parallel connections (`ab -n 2000 -c 200 http://localhost/` from `apache2-utils`/`httpd-tools`). Find the evidence and fix it.

<details>
<summary>Solution</summary>

Error log: `accept4() failed (24: Too many open files)`. `cat /proc/<worker pid>/limits` shows 64. Raise `LimitNOFILE` in the drop-in (e.g. 65536) and restart.

</details>

---

## Challenge – Ticket simulation ★★★

Ask a partner to break nginx with **two** of the problems above at once (or write a script that picks two at random). You must restore service in under 10 minutes, writing each step and its evidence in `/root/ticket.txt`.

<details>
<summary>Solution</summary>

Follow the checklist strictly: status/journal → config test → user/files → ports (`ss`) → limits → sandboxing (`systemctl show -p Protect*`) → MAC → firewall. Fix one layer at a time and re-test after each fix.

</details>

---

<!-- footer:start -->
[← Exercises: Managing Services with systemd](04-systemd.md) · [Module exercises](README.md) · [Exercises: Scheduling Jobs: cron, at and systemd Timers →](06-scheduling-jobs.md)
<!-- footer:end -->
