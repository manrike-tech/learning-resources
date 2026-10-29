# Exercises – SSL/TLS Certificates

<!-- nav:start -->
> **Module 08 · Security, Virtualization & Containers** — lesson 4 of 6  
> Read first: [Theory – Working with SSL/TLS Certificates](../../theory/08-security-vms-containers/04-ssl-certificates.md) · [Course lesson (commands)](../../course/08-Security-VMs-Containers.md#lesson-4---ssltls-certificates-gap) · [Module exercises](README.md)
<!-- nav:end -->

> Work in `/root/pki` on `rhel1`; add `192.168.56.20 web.example.lab` to `/etc/hosts` on both VMs.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Keys ★

Create an RSA 2048 key, an ECDSA P-256 key and an Ed25519 key. Show the type and size of each, set safe permissions.

<details>
<summary>Solution</summary>

```bash
mkdir -p /root/pki && cd /root/pki
openssl genrsa -out rsa.key 2048
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out ec.key
openssl genpkey -algorithm ed25519 -out ed.key
chmod 600 *.key
for k in *.key; do openssl pkey -in $k -noout -text | head -1; done
```

</details>

---

## Exercise 2 – CSR with SAN ★

Create a CSR for `web.example.lab` with SANs `web.example.lab`, `www.example.lab` and IP `192.168.56.20`, using `rsa.key`. Inspect and verify it.

<details>
<summary>Solution</summary>

```bash
openssl req -new -key rsa.key -out web.csr -subj "/O=Example Lab/CN=web.example.lab" \
  -addext "subjectAltName=DNS:web.example.lab,DNS:www.example.lab,IP:192.168.56.20"
openssl req -in web.csr -noout -text | grep -A1 -E 'Subject:|Alternative'
openssl req -in web.csr -noout -verify
```

</details>

---

## Exercise 3 – Self-signed (exam-style) ★★

> Create a self-signed certificate for `web.lab` valid **730 days** at `/etc/pki/tls/certs/web.crt`, key at `/etc/pki/tls/private/web.key`.

**Verify:** `openssl x509 -in /etc/pki/tls/certs/web.crt -noout -subject -dates -ext subjectAltName`.

<details>
<summary>Solution</summary>

```bash
openssl req -x509 -newkey rsa:2048 -nodes -days 730 \
  -keyout /etc/pki/tls/private/web.key -out /etc/pki/tls/certs/web.crt \
  -subj "/CN=web.lab" -addext "subjectAltName=DNS:web.lab"
chmod 600 /etc/pki/tls/private/web.key
```

</details>

---

## Exercise 4 – Lab CA ★★

Create a lab CA (`ca.key`, `ca.crt`, 10 years), sign `web.csr` from Exercise 2 with it (825 days, keeping the SANs), and verify the result.

<details>
<summary>Solution</summary>

```bash
openssl req -x509 -newkey rsa:4096 -nodes -days 3650 -keyout ca.key -out ca.crt \
  -subj "/CN=Example Lab CA" -addext "basicConstraints=critical,CA:TRUE" -addext "keyUsage=critical,keyCertSign,cRLSign"
cat > web.ext << 'EOF'
subjectAltName = DNS:web.example.lab, DNS:www.example.lab, IP:192.168.56.20
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
EOF
openssl x509 -req -in web.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out web.crt -days 825 -sha256 -extfile web.ext
openssl verify -CAfile ca.crt web.crt
```

</details>

---

## Exercise 5 – Trust the CA and use HTTPS ★★★

1. Trust `ca.crt` system-wide on **both** VMs.
2. Configure nginx (or httpd + mod_ssl) on `rhel1` with `web.crt`/`rsa.key` on port 443; open the firewall.
3. From `deb1`, `curl https://web.example.lab/` must succeed **without** `-k`.

<details>
<summary>Solution</summary>

```bash
# rhel1
cp ca.crt /etc/pki/ca-trust/source/anchors/example-lab-ca.crt && update-ca-trust
cp web.crt /etc/pki/tls/certs/; cp rsa.key /etc/pki/tls/private/web-example.key
cat > /etc/nginx/conf.d/web-ssl.conf << 'EOF'
server {
    listen 443 ssl;
    server_name web.example.lab;
    ssl_certificate     /etc/pki/tls/certs/web.crt;
    ssl_certificate_key /etc/pki/tls/private/web-example.key;
    root /usr/share/nginx/html;
}
EOF
nginx -t && systemctl reload nginx
firewall-cmd --permanent --add-service=https && firewall-cmd --reload
# deb1
scp rhel1:/root/pki/ca.crt /usr/local/share/ca-certificates/example-lab-ca.crt && update-ca-certificates
curl -I https://web.example.lab/
```

</details>

---

## Exercise 6 – Inspect a live server ★

Using `openssl s_client`, show the certificate chain, issuer, validity dates and verify result of `web.example.lab:443`, and of a public site (e.g. `www.kernel.org`).

<details>
<summary>Solution</summary>

```bash
openssl s_client -connect web.example.lab:443 -servername web.example.lab </dev/null 2>/dev/null | grep -E 's:|i:|Verify return'
openssl s_client -connect www.kernel.org:443 -servername www.kernel.org </dev/null 2>/dev/null | openssl x509 -noout -issuer -dates
```

</details>

---

## Exercise 7 – Key/cert mismatch ★★

Configure nginx with `web.crt` and `ec.key` (wrong key). Read the `nginx -t` error, prove the mismatch by comparing public keys, and fix it.

<details>
<summary>Solution</summary>

```bash
openssl x509 -in web.crt -noout -pubkey | sha256sum
openssl pkey -in ec.key -pubout | sha256sum          # different
openssl pkey -in rsa.key -pubout | sha256sum         # same as the cert
```

`nginx -t` → `SSL_CTX_use_PrivateKey(...) failed (... key values mismatch)`.

</details>

---

## Exercise 8 – Formats ★

Convert `web.crt` to DER and back; bundle key + cert + CA into `web.p12` (password `lab`); extract the certificate from the `.p12` again.

<details>
<summary>Solution</summary>

```bash
openssl x509 -in web.crt -outform der -out web.der
openssl x509 -in web.der -inform der -out web-again.pem
openssl pkcs12 -export -inkey rsa.key -in web.crt -certfile ca.crt -out web.p12 -passout pass:lab
openssl pkcs12 -in web.p12 -clcerts -nokeys -passin pass:lab | openssl x509 -noout -subject
```

</details>

---

## Challenge – Expiry monitoring ★★★

Write `/usr/local/bin/certcheck` that takes host names, connects to port 443 of each, and prints `OK`, `WARN` (< 30 days left) or `EXPIRED` with the expiry date. Schedule it daily with a systemd timer.

<details>
<summary>Solution</summary>

```bash
#!/bin/bash
for h in "$@"; do
  c=$(openssl s_client -connect "$h:443" -servername "$h" </dev/null 2>/dev/null | openssl x509 2>/dev/null) || { echo "ERROR   $h"; continue; }
  end=$(openssl x509 -noout -enddate <<<"$c" | cut -d= -f2)
  if ! openssl x509 -noout -checkend 0 <<<"$c" >/dev/null; then s=EXPIRED
  elif ! openssl x509 -noout -checkend 2592000 <<<"$c" >/dev/null; then s=WARN
  else s=OK; fi
  printf '%-8s %-30s %s\n' "$s" "$h" "$end"
done
```

</details>

---

<!-- footer:start -->
[← Exercises: AppArmor](03-apparmor.md) · [Module exercises](README.md) · [Exercises: Virtual Machines with libvirt →](05-libvirt-vms.md)
<!-- footer:end -->
