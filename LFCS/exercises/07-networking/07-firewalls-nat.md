# Exercises – Packet Filtering, Port Redirection and NAT

<!-- nav:start -->
> **Module 07 · Networking** — lesson 7 of 11  
> Read first: [Theory – Packet Filtering, Port Redirection and NAT](../../theory/07-networking/07-firewalls-nat.md) · [Course lesson (commands)](../../course/07-Networking.md#lesson-7---firewalls-port-redirection--nat-ch-30) · [Module exercises](README.md)
<!-- nav:end -->

> firewalld exercises on `rhel1`; ufw and nftables on `deb1` (use **one** manager at a time). Keep an SSH rule in place – or work from the console.
> Test services: `python3 -m http.server 80` / `8080` (run in a directory with an `index.html`).

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – firewalld inventory ★

Record the default zone, active zones with their interfaces, and everything allowed in the active zone.

<details>
<summary>Solution</summary>

```bash
firewall-cmd --get-default-zone; firewall-cmd --get-active-zones; firewall-cmd --list-all
```

</details>

---

## Exercise 2 – Runtime vs permanent ★★

1. Allow `http` **only at runtime**; confirm it works from `deb1`; run `firewall-cmd --reload`; test again.
2. Allow `http` and `https` **permanently**; confirm it's not active until you reload.

<details>
<summary>Solution</summary>

```bash
firewall-cmd --add-service=http; curl -I http://rhel1/      # works (from deb1)
firewall-cmd --reload                                       # rule gone
firewall-cmd --permanent --add-service={http,https}
firewall-cmd --list-services                                # not yet
firewall-cmd --reload; firewall-cmd --list-services
```

</details>

---

## Exercise 3 – Ports and sources ★★

1. Open `8080/tcp` permanently.
2. Allow MySQL (3306/tcp) **only** from `192.168.56.10` using a rich rule.
3. Put the subnet `192.168.100.0/24` in zone `trusted`.

<details>
<summary>Solution</summary>

```bash
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.10" port port="3306" protocol="tcp" accept'
firewall-cmd --permanent --zone=trusted --add-source=192.168.100.0/24
firewall-cmd --reload; firewall-cmd --list-all; firewall-cmd --zone=trusted --list-all
```

</details>

---

## Exercise 4 – Local port redirection ★★

> Redirect incoming TCP 8443 to local 443 (or 8080 → 80 for a plain HTTP test), persistently.

Test from `deb1` with `curl http://rhel1:8080/` while only port 80 has a listener.

<details>
<summary>Solution</summary>

```bash
firewall-cmd --permanent --add-forward-port=port=8080:proto=tcp:toport=80
firewall-cmd --reload; firewall-cmd --list-forward-ports
```

</details>

---

## Exercise 5 – Port forwarding to another host ★★★

On `rhel1`, forward TCP 2222 to `deb1:22`. From a third machine (or the hypervisor host), `ssh -p 2222 user@rhel1` should land on `deb1`.

<details>
<summary>Solution</summary>

```bash
firewall-cmd --permanent --add-forward-port=port=2222:proto=tcp:toport=22:toaddr=192.168.56.10
firewall-cmd --permanent --add-masquerade
firewall-cmd --reload
sysctl net.ipv4.ip_forward        # 1 (firewalld enables it with masquerade)
```

</details>

---

## Exercise 6 – ufw ★

On `deb1`: default deny incoming, allow SSH, allow HTTP only from `192.168.56.0/24`, rate-limit SSH, enable. List rules numbered and delete the HTTP rule.

<details>
<summary>Solution</summary>

```bash
ufw default deny incoming; ufw default allow outgoing
ufw allow OpenSSH
ufw allow from 192.168.56.0/24 to any port 80 proto tcp
ufw limit ssh
ufw enable; ufw status numbered
ufw delete <number>
```

</details>

---

## Exercise 7 – nftables host firewall ★★★

Disable ufw on `deb1`. Write `/etc/nftables.conf` that allows established/related, loopback, ICMP/ICMPv6, SSH and HTTP; rejects everything else on input; drops forwarding. Load it, make it persistent, test from `rhel1` (SSH and HTTP work, 8080 rejected).

<details>
<summary>Solution</summary>

```nft
#!/usr/sbin/nft -f
flush ruleset
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        ct state invalid drop
        iif "lo" accept
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept
        tcp dport { 22, 80 } accept
        reject
    }
    chain forward { type filter hook forward priority 0; policy drop; }
    chain output  { type filter hook output priority 0; policy accept; }
}
```

```bash
ufw disable
nft -c -f /etc/nftables.conf && nft -f /etc/nftables.conf
systemctl enable --now nftables
```

</details>

---

## Exercise 8 – NAT with nftables ★★★

Using the router setup from [Static Routing](05-static-routing.md) (namespace `backend` with `10.99.0.5`), make `deb1` masquerade traffic from `10.99.0.0/24` leaving `enp1s0`, and DNAT `deb1:8080` to `10.99.0.5:80`. Allow the forwarded traffic.

<details>
<summary>Solution</summary>

```nft
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat;
        iif "enp1s0" tcp dport 8080 dnat to 10.99.0.5:80
    }
    chain postrouting {
        type nat hook postrouting priority srcnat;
        oif "enp1s0" ip saddr 10.99.0.0/24 masquerade
    }
}
# in table inet filter, chain forward:
#   ct state established,related accept
#   iif "veth-r" oif "enp1s0" accept
#   iif "enp1s0" oif "veth-r" ip daddr 10.99.0.5 tcp dport 80 accept
```

Test: `ip netns exec backend python3 -m http.server 80 &` then from `rhel1`: `curl http://deb1:8080/`.

</details>

---

## Challenge – Read legacy rules ★★

Translate into nftables (use `iptables-translate` to check):

```text
iptables -A INPUT -p tcp -s 10.0.0.0/8 --dport 5432 -j ACCEPT
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

<details>
<summary>Solution</summary>

```nft
nft add rule ip filter INPUT ip saddr 10.0.0.0/8 tcp dport 5432 counter accept
nft add rule ip nat PREROUTING tcp dport 80 counter redirect to :8080
nft add rule ip nat POSTROUTING oifname "eth0" counter masquerade
```

</details>

---

<!-- footer:start -->
[← Exercises: Bridges and Bonds](06-bridges-bonds.md) · [Module exercises](README.md) · [Exercises: The OpenSSH Server and Client →](08-openssh.md)
<!-- footer:end -->
