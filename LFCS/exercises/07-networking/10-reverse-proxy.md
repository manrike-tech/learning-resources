# Exercises – Reverse Proxies and Load Balancers

<!-- nav:start -->
> **Module 07 · Networking** — lesson 10 of 11  
> Read first: [Theory – Reverse Proxies and Load Balancers](../../theory/07-networking/10-reverse-proxy.md) · [Course lesson (commands)](../../course/07-Networking.md#lesson-10---reverse-proxy--load-balancer-gap) · [Module exercises](README.md)
<!-- nav:end -->

> Proxy: `deb1` (Nginx). Back-ends: `rhel1`. Add `192.168.56.10 app.example.lab` to `/etc/hosts` on both VMs.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Setup – back-ends on rhel1

```bash
mkdir -p /srv/b1 /srv/b2
echo "backend 1" > /srv/b1/index.html; echo "backend 2" > /srv/b2/index.html
cd /srv/b1 && nohup python3 -m http.server 8081 >/dev/null 2>&1 &
cd /srv/b2 && nohup python3 -m http.server 8082 >/dev/null 2>&1 &
firewall-cmd --add-port={8081,8082}/tcp
```

---

## Exercise 1 – Single back-end ★

On `deb1`, publish `rhel1:8081` as `http://app.example.lab/` with the `Host`, `X-Real-IP` and `X-Forwarded-For` headers.

**Verify:** `curl http://app.example.lab/` → `backend 1`.

<details>
<summary>Solution</summary>

```nginx
# /etc/nginx/conf.d/app.conf
server {
    listen 80;
    server_name app.example.lab;
    location / {
        proxy_pass http://192.168.56.20:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
nginx -t && systemctl reload nginx
```

(On Ubuntu, remove the default site if it catches the request: `rm /etc/nginx/sites-enabled/default`.)

</details>

---

## Exercise 2 – Load balancer (exam-style) ★★

> Nginx on port 80 load-balancing to `rhel1:8081` and `rhel1:8082`.

**Verify:** `for i in 1 2 3 4; do curl -s http://app.example.lab/; done` alternates.

<details>
<summary>Solution</summary>

```nginx
upstream app_backend {
    server 192.168.56.20:8081;
    server 192.168.56.20:8082;
}
server {
    listen 80;
    server_name app.example.lab;
    location / {
        proxy_pass http://app_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

</details>

---

## Exercise 3 – Weights, backup, failure ★★

1. Give 8081 `weight=3`; count the distribution over 8 requests.
2. Make 8082 a `backup`; stop the 8081 server; test; start it again.

<details>
<summary>Solution</summary>

```bash
for i in $(seq 8); do curl -s http://app.example.lab/; done | sort | uniq -c
```

With `backup`, requests go only to 8081 until it stops; then 8082 answers.

</details>

---

## Exercise 4 – Path routing ★★

Route `/api/` to 8082 with the `/api` prefix stripped, and everything else to the upstream.

<details>
<summary>Solution</summary>

```nginx
location /api/ {
    proxy_pass http://192.168.56.20:8082/;
}
location / {
    proxy_pass http://app_backend;
}
```

`curl http://app.example.lab/api/` → `backend 2`.

</details>

---

## Exercise 5 – 502 Bad Gateway ★

Stop both back-ends and request the page. Read the error log and explain.

<details>
<summary>Solution</summary>

`tail /var/log/nginx/error.log` → `connect() failed (111: Connection refused) while connecting to upstream`. The proxy works, the back-ends don't.

</details>

---

## Exercise 6 – RHEL as proxy (SELinux) ★★★

Repeat Exercise 2 with Nginx on `rhel1` proxying to back-ends on `deb1`. Diagnose the 502 caused by SELinux and fix it persistently.

<details>
<summary>Solution</summary>

```bash
ausearch -m AVC -ts recent | grep name_connect
setsebool -P httpd_can_network_connect 1
firewall-cmd --permanent --add-service=http && firewall-cmd --reload
```

</details>

---

## Exercise 7 – HTTPS termination ★★★

Add HTTPS to `app.example.lab` on `deb1` with a self-signed certificate (see [SSL Certificates](../../theory/08-security-vms-containers/04-ssl-certificates.md)), redirect HTTP to HTTPS, and pass `X-Forwarded-Proto`.

<details>
<summary>Solution</summary>

```bash
mkdir -p /etc/nginx/ssl
openssl req -x509 -newkey rsa:2048 -nodes -days 365 -subj "/CN=app.example.lab" \
  -addext "subjectAltName=DNS:app.example.lab" \
  -keyout /etc/nginx/ssl/app.key -out /etc/nginx/ssl/app.crt
chmod 600 /etc/nginx/ssl/app.key
```

```nginx
server { listen 80; server_name app.example.lab; return 301 https://$host$request_uri; }
server {
    listen 443 ssl;
    server_name app.example.lab;
    ssl_certificate     /etc/nginx/ssl/app.crt;
    ssl_certificate_key /etc/nginx/ssl/app.key;
    location / {
        proxy_pass http://app_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

`curl -k https://app.example.lab/`; `curl -I http://app.example.lab/` → 301.

</details>

---

## Challenge – TCP load balancing ★★★

Load-balance SSH: connections to `deb1:2200` go round-robin to `rhel1:22` and `deb1:22` using the Nginx `stream` module.

<details>
<summary>Solution</summary>

```nginx
# top level of /etc/nginx/nginx.conf (Ubuntu: libnginx-mod-stream is usually installed)
stream {
    upstream ssh_pool {
        server 192.168.56.20:22;
        server 127.0.0.1:22;
    }
    server {
        listen 2200;
        proxy_pass ssh_pool;
    }
}
```

`ssh -p 2200 ana@deb1 hostname` several times (accept both host keys).

</details>

---

<!-- footer:start -->
[← Exercises: Setting and Synchronizing System Time](09-time-sync.md) · [Module exercises](README.md) · [Exercises: Monitoring and Troubleshooting the Network →](11-troubleshooting.md)
<!-- footer:end -->
