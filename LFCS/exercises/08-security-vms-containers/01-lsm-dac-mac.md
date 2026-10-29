# Exercises – Linux Security Modules: DAC and MAC

<!-- nav:start -->
> **Module 08 · Security, Virtualization & Containers** — lesson 1 of 6  
> Read first: [Theory – Linux Security Modules: DAC and MAC](../../theory/08-security-vms-containers/01-lsm-dac-mac.md) · [Course lesson (commands)](../../course/08-Security-VMs-Containers.md#lesson-1---linux-security-modules-ch-33) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Which LSMs? ★

On both VMs, list the active LSMs and identify the major MAC module.

<details>
<summary>Solution</summary>

```bash
cat /sys/kernel/security/lsm
getenforce 2>/dev/null; aa-status 2>/dev/null | head -2
```

</details>

---

## Exercise 2 – Contexts and profiles ★

1. On `rhel1`: show the SELinux context of your shell, of `sshd`, of `/etc/shadow` and of `/var/www` (install httpd/nginx if needed).
2. On `deb1`: list which running processes are confined by AppArmor.

<details>
<summary>Solution</summary>

```bash
id -Z; ps -eZ | grep sshd; ls -Z /etc/shadow; ls -dZ /var/www
aa-status | sed -n '/processes are in enforce mode/,/processes are in complain/p'
```

</details>

---

## Exercise 3 – DAC allows, MAC denies ★★

On `rhel1` (enforcing), make nginx serve `/var/www/html/test.html` that you **moved** from `/root`. Confirm `ls -l` permissions are fine (644, parent dirs 755) yet you get 403. Show the evidence that MAC denied it.

<details>
<summary>Solution</summary>

```bash
echo test > /root/test.html; mv /root/test.html /usr/share/nginx/html/
curl -I http://localhost/test.html           # 403
namei -l /usr/share/nginx/html/test.html     # DAC ok
ausearch -m AVC -ts recent | tail -3         # tcontext=…admin_home_t
restorecon -v /usr/share/nginx/html/test.html; curl -I http://localhost/test.html
```

</details>

---

## Exercise 4 – Root is confined too ★★

Run `nginx` (as root) and try to make it read `/etc/shadow` by symlinking it into the document root. What happens on `rhel1` (SELinux) and on `deb1` (AppArmor, if nginx has a profile)?

<details>
<summary>Solution</summary>

On RHEL: AVC denial (`httpd_t` may not read `shadow_t`) even though the master process is root; workers also run as `nginx` (DAC denies too). It illustrates that MAC confines root processes. On Ubuntu without an nginx profile, only DAC applies.

</details>

---

## Exercise 5 – Capabilities ★★

1. Show the file capabilities of `/usr/bin/ping`.
2. Copy `ping` to `/tmp/myping` (capabilities are not copied) and run it as a normal user. Then grant `cap_net_raw` and retry.

<details>
<summary>Solution</summary>

```bash
getcap /usr/bin/ping
cp /usr/bin/ping /tmp/myping
su - someuser -c '/tmp/myping -c1 127.0.0.1'     # may fail: Operation not permitted (depends on net.ipv4.ping_group_range)
setcap cap_net_raw+ep /tmp/myping; getcap /tmp/myping
su - someuser -c '/tmp/myping -c1 127.0.0.1'
```

</details>

---

## Challenge – Explain it ★★

Write in `/root/mac.txt`, in your own words (5–8 sentences): why MAC exists, in which order DAC and MAC are checked, and one concrete attack that SELinux/AppArmor would limit on a compromised web server.

---

<!-- footer:start -->
[Module exercises](README.md) · [Exercises: SELinux →](02-selinux.md)
<!-- footer:end -->
