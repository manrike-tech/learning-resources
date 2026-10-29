# Exercises – Hostname and Name Resolution

<!-- nav:start -->
> **Module 07 · Networking** — lesson 4 of 11  
> Read first: [Theory – Hostname and Name Resolution](../../theory/07-networking/04-hostname-resolution.md) · [Course lesson (commands)](../../course/07-Networking.md#lesson-4---hostname--name-resolution) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Hostnames ★

Set the static hostname of each VM to its FQDN (`deb1.example.lab`, `rhel1.example.lab`) and a pretty name. Verify with `hostnamectl`, `hostname -s` and `cat /etc/hostname`.

<details>
<summary>Solution</summary>

```bash
hostnamectl set-hostname rhel1.example.lab
hostnamectl set-hostname "Lab RHEL server" --pretty
hostnamectl; hostname -s; cat /etc/hostname
```

</details>

---

## Exercise 2 – /etc/hosts ★

Make each VM resolve the other by short name and FQDN. Also make `rhel1` known as `ldap.example.lab`.

**Verify:** `getent hosts rhel1 rhel1.example.lab ldap.example.lab` on `deb1`; `hostname -f` on each VM.

<details>
<summary>Solution</summary>

```text
192.168.56.10  deb1.example.lab   deb1
192.168.56.20  rhel1.example.lab  rhel1  ldap.example.lab
```

Add these lines to `/etc/hosts` on both VMs.

</details>

---

## Exercise 3 – getent vs dig ★★

Run `getent hosts rhel1` and `dig rhel1` / `dig +short rhel1.example.lab`. Explain the different results. Then check `/etc/nsswitch.conf` for the `hosts:` order.

<details>
<summary>Solution</summary>

`getent` uses NSS (`files` first → `/etc/hosts`); `dig` sends a DNS query to the configured server, which doesn't know the lab names.

</details>

---

## Exercise 4 – Which DNS servers? ★

On each VM, find the DNS servers actually used and where they were configured.

<details>
<summary>Solution</summary>

```bash
resolvectl status                 # Ubuntu (resolv.conf → 127.0.0.53)
nmcli dev show | grep DNS          # RHEL
ls -l /etc/resolv.conf; cat /etc/resolv.conf
```

</details>

---

## Exercise 5 – Override order ★★

Add a bogus entry `10.1.1.1 www.kernel.org` to `/etc/hosts`. Compare `getent hosts www.kernel.org`, `ping -c1 www.kernel.org` and `dig +short www.kernel.org`. Remove the entry.

<details>
<summary>Solution</summary>

`getent` and `ping` use `10.1.1.1` (files before dns); `dig` shows the real addresses.

</details>

---

## Exercise 6 – Persistent DNS and search domain ★★

1. On `deb1` (Netplan), set DNS `9.9.9.9` and search domain `example.lab` for the main interface.
2. On `rhel1` (nmcli), the same.
3. Prove `ping rhel1` would try `rhel1.example.lab` (look at `resolv.conf` / `resolvectl`).

<details>
<summary>Solution</summary>

Netplan `nameservers: {addresses: [9.9.9.9], search: [example.lab]}` + `netplan apply`; nmcli `nmcli con mod enp1s0 ipv4.dns 9.9.9.9 ipv4.dns-search example.lab && nmcli con up enp1s0`. `resolvectl domain` / `grep search /etc/resolv.conf`.

</details>

---

## Exercise 7 – DNS queries ★

Using `dig`, find: the A and AAAA records of `www.kernel.org`, the MX records of `gmail.com`, the reverse name of `1.1.1.1`, and query `example.com` directly at `8.8.8.8`.

<details>
<summary>Solution</summary>

```bash
dig +short A www.kernel.org; dig +short AAAA www.kernel.org
dig +short MX gmail.com
dig +short -x 1.1.1.1
dig @8.8.8.8 example.com
```

</details>

---

## Challenge – A tiny DNS server ★★★

On `rhel1`, install `dnsmasq`, make it serve the names in its `/etc/hosts` to the lab, open UDP/TCP 53, and point `deb1` at it as its only DNS server. Prove that `dig rhel1.example.lab` on `deb1` now works.

<details>
<summary>Solution</summary>

```bash
# rhel1
dnf install -y dnsmasq
cat > /etc/dnsmasq.d/lab.conf << 'EOF'
interface=enp1s0
bind-interfaces
domain=example.lab
expand-hosts
EOF
systemctl enable --now dnsmasq
firewall-cmd --permanent --add-service=dns && firewall-cmd --reload
# deb1: Netplan nameservers: addresses: [192.168.56.20]; netplan apply
dig +short rhel1.example.lab
```

</details>

---

<!-- footer:start -->
[← Exercises: Persistent Network Configuration: Netplan and NetworkManager](03-persistent-config.md) · [Module exercises](README.md) · [Exercises: Static Routing →](05-static-routing.md)
<!-- footer:end -->
