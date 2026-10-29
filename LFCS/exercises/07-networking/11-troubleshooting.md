# Exercises – Monitoring and Troubleshooting the Network

<!-- nav:start -->
> **Module 07 · Networking** — lesson 11 of 11  
> Read first: [Theory – Monitoring and Troubleshooting the Network](../../theory/07-networking/11-troubleshooting.md) · [Course lesson (commands)](../../course/07-Networking.md#lesson-11---monitoring--troubleshooting) · [Module exercises](README.md)
<!-- nav:end -->

> Most exercises are **break-fix**: one terminal breaks something (or ask a partner), the other diagnoses using the bottom-up method **without** looking at what was broken.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Baseline ★

Record a healthy baseline of `rhel1` in `/root/net-baseline.txt`: link state, addresses, routes, DNS servers, listening sockets, firewall rules.

<details>
<summary>Solution</summary>

```bash
{ ip -br link; ip -br addr; ip route; resolvectl status 2>/dev/null || cat /etc/resolv.conf
  ss -tulpn; firewall-cmd --list-all 2>/dev/null || nft list ruleset; } > /root/net-baseline.txt
```

</details>

---

## Exercise 2 – Refused vs timeout ★

On `rhel1`, run a server on 8080 and test from `deb1` with `nc -zv`. Then: (a) stop the server, (b) add a firewalld rich rule that **rejects** 8080, (c) one that **drops** 8080. Record the client-side message for each.

<details>
<summary>Solution</summary>

(a) Connection refused (RST, nothing listening) · (b) "No route to host" / refused (ICMP reject from firewalld) · (c) timed out.

```bash
firewall-cmd --add-rich-rule='rule port port="8080" protocol="tcp" reject'
firewall-cmd --add-rich-rule='rule port port="8080" protocol="tcp" drop'
```

</details>

---

## Exercise 3 – Listening on the wrong address ★

Start `python3 -m http.server 8080 --bind 127.0.0.1` on `rhel1`, open the firewall, and diagnose from `deb1` why it fails.

<details>
<summary>Solution</summary>

`ss -tlnp | grep 8080` shows `127.0.0.1:8080`. Restart with `--bind 0.0.0.0`.

</details>

---

## Exercise 4 – tcpdump the handshake ★★

Capture on `rhel1` (`tcpdump -ni enp1s0 tcp port 22 -c 6`) while connecting with SSH from `deb1`. Identify SYN, SYN-ACK and ACK. Then capture a connection attempt to a closed port and identify the RST.

---

## Exercise 5 – Break-fix round 1 ★★

The breaker runs **one** of these on `rhel1`; the fixer restores connectivity from `deb1` to `http://rhel1/` (nginx running):

1. `ip link set enp1s0 down` (from the console!)
2. `ip addr del 192.168.56.20/24 dev enp1s0; ip addr add 192.168.56.20/32 dev enp1s0`
3. `firewall-cmd --remove-service=http`
4. `systemctl stop nginx`
5. `sed -i 's/listen       80;/listen 127.0.0.1:80;/' /etc/nginx/nginx.conf && systemctl restart nginx`

<details>
<summary>Diagnosis hints</summary>

1. `ip -br link` – DOWN · 2. prefix /32, no connected route; `ping` from deb1 fails or replies don't return · 3. `nc -zv` refused/timeout; `firewall-cmd --list-services` · 4. refused; `systemctl status nginx` · 5. refused from deb1; `ss -tlnp` shows `127.0.0.1:80`.

</details>

---

## Exercise 6 – Break-fix round 2 (DNS and routing) ★★★

1. Remove `rhel1` from `deb1`'s `/etc/hosts` and set a wrong DNS server; `curl http://rhel1.example.lab/` fails – prove it's DNS.
2. On `rhel1`, replace the default route with a bogus gateway; from `deb1` (same subnet) things still work, but `rhel1` can't reach the internet – prove it with `ip route get 1.1.1.1` and `tracepath`.

---

## Exercise 7 – MTU problem ★★★

Set `mtu 1400` on `rhel1`'s interface only. Show that `ping -s 1472 -M do` from `deb1` fails while small pings work. Explain why such problems often appear as "SSH works but large downloads hang".

---

## Challenge – Write your own checklist ★★

Write a script `/usr/local/bin/netcheck HOST PORT` that prints, in order: local link status, default gateway reachability, DNS resolution of HOST (via getent), ping to HOST, TCP connect to HOST:PORT (with a 3 s timeout) – each line with OK/FAIL.

<details>
<summary>Solution</summary>

```bash
#!/bin/bash
h=$1; p=$2
chk() { if eval "$2" >/dev/null 2>&1; then echo "OK   $1"; else echo "FAIL $1"; fi; }
gw=$(ip route | awk '/^default/ {print $3; exit}')
chk "link up"          "ip -br link | grep -v '^lo' | grep -q ' UP '"
chk "gateway $gw"      "ping -c1 -W1 $gw"
chk "resolve $h"       "getent hosts $h"
chk "ping $h"          "ping -c1 -W1 $h"
chk "tcp $h:$p"        "timeout 3 bash -c '</dev/tcp/$h/$p'"
```

</details>

---

<!-- footer:start -->
[← Exercises: Reverse Proxies and Load Balancers](10-reverse-proxy.md) · [Module exercises](README.md)
<!-- footer:end -->
