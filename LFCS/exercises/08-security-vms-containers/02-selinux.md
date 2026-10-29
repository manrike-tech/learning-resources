# Exercises – SELinux

<!-- nav:start -->
> **Module 08 · Security, Virtualization & Containers** — lesson 2 of 6  
> Read first: [Theory – SELinux](../../theory/08-security-vms-containers/02-selinux.md) · [Course lesson (commands)](../../course/08-Security-VMs-Containers.md#lesson-2---selinux) · [Module exercises](README.md)
<!-- nav:end -->

> Run on `rhel1` in **enforcing** mode. Install: `dnf install -y httpd policycoreutils-python-utils setroubleshoot-server`; `systemctl enable --now httpd`; `firewall-cmd --permanent --add-service=http; firewall-cmd --reload`.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Modes ★

1. Show the current and configured mode.
2. Switch to permissive at runtime, verify, switch back.
3. Make permissive the persistent mode, reboot, check, and restore enforcing.

<details>
<summary>Solution</summary>

```bash
getenforce; sestatus | grep -E 'Current|config'
setenforce 0; getenforce; setenforce 1
sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config; reboot
getenforce
sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config; setenforce 1
```

</details>

---

## Exercise 2 – mv vs cp ★

Create `/root/a.html` and `/root/b.html`. `mv` the first and `cp` the second into `/var/www/html/`. Compare their contexts and fetch both with curl. Fix the broken one.

<details>
<summary>Solution</summary>

```bash
echo A > /root/a.html; echo B > /root/b.html
mv /root/a.html /var/www/html/; cp /root/b.html /var/www/html/
ls -Z /var/www/html/
curl -s -o /dev/null -w '%{http_code}\n' http://localhost/a.html     # 403
restorecon -v /var/www/html/a.html
```

</details>

---

## Exercise 3 – Custom document root (exam-style) ★★

> Serve `/srv/web` with httpd, SELinux enforcing, persistently labelled.

Configure `DocumentRoot "/srv/web"` (with a matching `<Directory>` block allowing access) and an `index.html`.

**Verify:** `curl http://localhost/` returns your page; `semanage fcontext -l -C` shows your rule; `restorecon -Rvn /srv/web` reports nothing to change.

<details>
<summary>Solution</summary>

```bash
mkdir -p /srv/web && echo "srv web" > /srv/web/index.html
sed -i 's|^DocumentRoot "/var/www/html"|DocumentRoot "/srv/web"|' /etc/httpd/conf/httpd.conf
cat > /etc/httpd/conf.d/srvweb.conf << 'EOF'
<Directory "/srv/web">
    Require all granted
</Directory>
EOF
semanage fcontext -a -t httpd_sys_content_t "/srv/web(/.*)?"
restorecon -Rv /srv/web
systemctl restart httpd; curl http://localhost/
```

</details>

---

## Exercise 4 – chcon is temporary ★★

Label `/srv/test` with `chcon -t httpd_sys_content_t`, check `ls -Z`, then run `restorecon -Rv /srv/test`. What happened? Why?

<details>
<summary>Solution</summary>

`restorecon` reverts it to `var_t` because the policy has no rule for `/srv/test`; `chcon` doesn't change the policy.

</details>

---

## Exercise 5 – Non-standard port ★★

Make httpd listen on port 8888 (`Listen 8888`). Read the failure, fix it with SELinux and the firewall.

<details>
<summary>Solution</summary>

```bash
echo "Listen 8888" > /etc/httpd/conf.d/port8888.conf
systemctl restart httpd          # fails
journalctl -xeu httpd | grep -i 8888; ausearch -m AVC -ts recent | grep name_bind
semanage port -a -t http_port_t -p tcp 8888
firewall-cmd --permanent --add-port=8888/tcp && firewall-cmd --reload
systemctl restart httpd; curl -I http://localhost:8888/
```

</details>

---

## Exercise 6 – Booleans ★★

1. Configure httpd as a reverse proxy to `http://127.0.0.1:3000` (`ProxyPass "/app" "http://127.0.0.1:3000/"`) with a back-end `python3 -m http.server 3000`.
2. Request `/app` – read the AVC and use `audit2why` to find the fix. Apply it persistently.

<details>
<summary>Solution</summary>

```bash
echo 'ProxyPass "/app" "http://127.0.0.1:3000/"' > /etc/httpd/conf.d/proxy.conf
systemctl restart httpd; (cd /tmp && python3 -m http.server 3000 &)
curl -I http://localhost/app        # 503
ausearch -m AVC -ts recent | audit2why
setsebool -P httpd_can_network_connect on
curl -I http://localhost/app        # 200
```

</details>

---

## Exercise 7 – User home directories ★★

Allow httpd to serve `~ana/public_html` (`UserDir public_html` in `/etc/httpd/conf.d/userdir.conf`). Find the boolean and labels needed.

<details>
<summary>Solution</summary>

```bash
setsebool -P httpd_enable_homedirs on
chmod 711 /home/ana; mkdir -p /home/ana/public_html; echo hi > /home/ana/public_html/index.html
chmod 755 /home/ana/public_html; restorecon -Rv /home/ana/public_html     # httpd_user_content_t
```

</details>

---

## Exercise 8 – sealert ★

Generate a denial (e.g. revert Exercise 5's port rule) and read the analysis with `sealert -a /var/log/audit/audit.log` and `journalctl -t setroubleshoot`.

---

## Challenge – Custom module (last resort) ★★★

Write a tiny service that runs as `httpd_t` and writes to `/srv/web/upload.log` (or make httpd's CGI write there). Fix it **first** properly with the correct file type (`httpd_sys_rw_content_t`). Then revert that, and instead create a policy module with `audit2allow`; review the `.te` file and explain why the label fix is better. Remove the module.

<details>
<summary>Solution</summary>

```bash
semanage fcontext -a -t httpd_sys_rw_content_t "/srv/web/upload.log"; restorecon -v /srv/web/upload.log
# alternative:
ausearch -m AVC -ts recent --raw | audit2allow -M mywebwrite
cat mywebwrite.te          # e.g. allow httpd_t httpd_sys_content_t:file write;  ← ALL web content becomes writable!
semodule -i mywebwrite.pp; semodule -r mywebwrite
```

The module grants write to every file of that type; the label limits write access to one file.

</details>

---

## Cleanup

```bash
semanage port -d -t http_port_t -p tcp 8888; semanage fcontext -d "/srv/web(/.*)?"
rm -f /etc/httpd/conf.d/{port8888,proxy,srvweb}.conf
setsebool -P httpd_can_network_connect off; setsebool -P httpd_enable_homedirs off
```

---

<!-- footer:start -->
[← Exercises: Linux Security Modules: DAC and MAC](01-lsm-dac-mac.md) · [Module exercises](README.md) · [Exercises: AppArmor →](03-apparmor.md)
<!-- footer:end -->
