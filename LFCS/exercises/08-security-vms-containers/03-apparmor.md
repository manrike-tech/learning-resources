# Exercises – AppArmor

<!-- nav:start -->
> **Module 08 · Security, Virtualization & Containers** — lesson 3 of 6  
> Read first: [Theory – AppArmor](../../theory/08-security-vms-containers/03-apparmor.md) · [Course lesson (commands)](../../course/08-Security-VMs-Containers.md#lesson-3---apparmor-ubuntu) · [Module exercises](README.md)
<!-- nav:end -->

> Run on `deb1` (Ubuntu). Install: `apt install -y apparmor-utils apparmor-profiles apparmor-profiles-extra`.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Status ★

How many profiles are loaded, in enforce and complain mode? Which running processes are confined?

<details>
<summary>Solution</summary>

```bash
aa-status
ps auxZ | grep -v '^unconfined' | head
```

</details>

---

## Exercise 2 – Read a profile ★

Open the profile of a confined service (e.g. `/etc/apparmor.d/usr.sbin.chronyd`). Identify includes, capabilities, file rules and the `local/` include.

---

## Exercise 3 – Complain and enforce ★

Put the `chronyd` profile in complain mode, verify, and set it back to enforce.

<details>
<summary>Solution</summary>

```bash
aa-complain /usr/sbin/chronyd; aa-status | grep -A3 complain
aa-enforce /usr/sbin/chronyd
```

</details>

---

## Exercise 4 – Build a profile ★★

1. Create `/usr/local/bin/readit`:
   ```bash
   #!/bin/bash
   cat /etc/hostname
   cat /etc/os-release | head -1
   ```
2. Generate a profile with `aa-genprof /usr/local/bin/readit` (run the script in another terminal while it scans; allow both reads).
3. Change the script to also read `/etc/shadow` (run it with sudo). What happens in enforce mode? Find the log line.

<details>
<summary>Solution</summary>

```bash
chmod 755 /usr/local/bin/readit
aa-genprof /usr/local/bin/readit       # S to scan, Allow, F to finish
echo 'cat /etc/shadow' >> /usr/local/bin/readit
sudo /usr/local/bin/readit             # "Permission denied" for /etc/shadow even as root
journalctl -k | grep 'apparmor="DENIED"' | grep readit
```

</details>

---

## Exercise 5 – Local override ★★

Allow `readit` to also read `/srv/notes.txt` using the local include (create `local/usr.local.bin.readit` and add `#include <local/usr.local.bin.readit>` to the profile if missing). Reload and test.

<details>
<summary>Solution</summary>

```bash
echo "/srv/notes.txt r," >> /etc/apparmor.d/local/usr.local.bin.readit
grep -q 'local/usr.local.bin.readit' /etc/apparmor.d/usr.local.bin.readit || \
  sed -i 's|^}|  #include <local/usr.local.bin.readit>\n}|' /etc/apparmor.d/usr.local.bin.readit
apparmor_parser -r /etc/apparmor.d/usr.local.bin.readit
```

</details>

---

## Exercise 6 – Nginx with a new document root ★★★

If your nginx is confined (install `apparmor-profiles-extra` or write a basic profile with `aa-genprof /usr/sbin/nginx`), move the site to `/srv/site`, observe the denial, and fix it.

<details>
<summary>Solution</summary>

Find `apparmor="DENIED" … name="/srv/site/index.html"` in the kernel log, add `/srv/site/** r,` to `/etc/apparmor.d/local/usr.sbin.nginx`, reload with `apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx`, restart nginx.

</details>

---

## Challenge – Disable and re-enable a profile ★

Disable the `readit` profile persistently, show it's no longer loaded, then enable it again.

<details>
<summary>Solution</summary>

```bash
aa-disable /usr/local/bin/readit; ls -l /etc/apparmor.d/disable/; aa-status | grep readit
rm /etc/apparmor.d/disable/usr.local.bin.readit; apparmor_parser -r /etc/apparmor.d/usr.local.bin.readit
```

</details>

---

<!-- footer:start -->
[← Exercises: SELinux](02-selinux.md) · [Module exercises](README.md) · [Exercises: Working with SSL/TLS Certificates →](04-ssl-certificates.md)
<!-- footer:end -->
