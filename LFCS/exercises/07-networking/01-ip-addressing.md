# Exercises – Network Addresses: IPv4 and IPv6

<!-- nav:start -->
> **Module 07 · Networking** — lesson 1 of 11  
> Read first: [Theory – Network Addresses: IPv4 and IPv6](../../theory/07-networking/01-ip-addressing.md) · [Course lesson (commands)](../../course/07-Networking.md#lesson-1---addresses-ch-27) · [Module exercises](README.md)
<!-- nav:end -->

> Do the calculations **by hand** first, then check with `ipcalc` or Python.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Your addresses ★

For each interface of both VMs, record MAC, IPv4 with prefix, IPv6 global and link-local addresses, and the default gateway.

<details>
<summary>Solution</summary>

```bash
ip -br link; ip -br addr; ip route | grep default; ip -6 route | grep default
```

</details>

---

## Exercise 2 – Subnet calculations ★★

For each address give network, broadcast, first and last host, and number of usable hosts:

1. `192.168.56.77/26`
2. `10.20.30.40/22`
3. `172.16.5.130/25`
4. `172.20.13.200/20`
5. `192.168.1.9/30`

<details>
<summary>Solution</summary>

| # | Network | Broadcast | Hosts | Usable |
|---|---------|-----------|-------|--------|
| 1 | 192.168.56.64 | 192.168.56.127 | .65 – .126 | 62 |
| 2 | 10.20.28.0 | 10.20.31.255 | 10.20.28.1 – 10.20.31.254 | 1022 |
| 3 | 172.16.5.128 | 172.16.5.255 | .129 – .254 | 126 |
| 4 | 172.20.0.0 | 172.20.15.255 | 172.20.0.1 – 172.20.15.254 | 4094 |
| 5 | 192.168.1.8 | 192.168.1.11 | .9 – .10 | 2 |

Check: `python3 -c "import ipaddress as i; n=i.ip_interface('10.20.30.40/22').network; print(n, n.broadcast_address, n.num_addresses-2)"`

</details>

---

## Exercise 3 – Same subnet? ★★

Can these pairs talk without a router?

1. `192.168.56.70/26` and `192.168.56.130/26`
2. `10.0.5.1/16` and `10.0.200.9/16`
3. `192.168.10.5/23` and `192.168.11.200/23`
4. `172.16.0.10/28` and `172.16.0.20/28`

<details>
<summary>Solution</summary>

1. No (64–127 vs 128–191). 2. Yes. 3. Yes (192.168.10.0–11.255). 4. No (0–15 vs 16–31).

</details>

---

## Exercise 4 – Subnetting a network ★★★

Split `192.168.100.0/24` into 4 equal subnets; list each network with its usable range. Then into subnets of at least 50 hosts – what prefix do you need?

<details>
<summary>Solution</summary>

/26 each: `.0/26` (.1–.62), `.64/26` (.65–.126), `.128/26` (.129–.190), `.192/26` (.193–.254). For ≥ 50 hosts you need 64 addresses → /26 as well.

</details>

---

## Exercise 5 – IPv6 notation ★

Shorten:
1. `2001:0db8:0000:0000:0001:0000:0000:0010`
2. `fd00:0056:0000:0000:0000:0000:0000:0020`
3. `fe80:0000:0000:0000:5054:00ff:fe3a:1f9c`

Expand `2001:db8::1:0:0:10` fully.

<details>
<summary>Solution</summary>

1. `2001:db8::1:0:0:10` (only one `::`; leftmost longest run)
2. `fd00:56::20`
3. `fe80::5054:ff:fe3a:1f9c`
4. `2001:0db8:0000:0000:0001:0000:0000:0010`

</details>

---

## Exercise 6 – ARP and NDP ★★

1. Flush the neighbour cache, ping the other VM by IPv4, and show the new entry.
2. Ping the other VM's IPv6 link-local address and show the NDP entry.

<details>
<summary>Solution</summary>

```bash
ip neigh flush dev enp1s0
ping -c2 192.168.56.20; ip neigh show 192.168.56.20
ping -6 -c2 fe80::<peer-id>%enp1s0; ip -6 neigh
```

</details>

---

## Challenge – Identify the ranges ★

Classify each: `10.255.1.1`, `172.32.0.1`, `169.254.10.10`, `100.64.0.5`, `fd12:3456::1`, `fe80::1`, `2001:4860::8888`, `127.0.1.1`, `192.0.2.10`.

<details>
<summary>Solution</summary>

private · public (172.16/12 ends at 172.31) · link-local · CGNAT · IPv6 ULA · IPv6 link-local · IPv6 global · loopback · documentation (TEST-NET-1)

</details>

---

<!-- footer:start -->
[Module exercises](README.md) · [Exercises: Runtime Network Configuration with ip →](02-runtime-ip.md)
<!-- footer:end -->
