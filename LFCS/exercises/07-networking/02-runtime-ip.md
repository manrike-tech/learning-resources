# Exercises – Runtime Configuration with ip

<!-- nav:start -->
> **Module 07 · Networking** — lesson 2 of 11  
> Read first: [Theory – Runtime Network Configuration with ip](../../theory/07-networking/02-runtime-ip.md) · [Course lesson (commands)](../../course/07-Networking.md#lesson-2---runtime-configuration-with-ip-ch-28) · [Module exercises](README.md)
<!-- nav:end -->

> Use a **spare** interface (add a second NIC to each VM on an isolated network, e.g. `enp7s0`) or a dummy interface (`ip link add dummy0 type dummy`). Never change the interface you're connected through.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Read everything ★

Save to `/root/net-now.txt`: brief link and address lists, IPv4 and IPv6 routes, neighbour table.

<details>
<summary>Solution</summary>

```bash
{ ip -br link; ip -br addr; ip route; ip -6 route; ip neigh; } > /root/net-now.txt
```

</details>

---

## Exercise 2 – Addresses ★

On the spare interface of `deb1`, add `192.168.100.10/24` and `fd00:100::10/64`; on `rhel1` add `.20` / `::20`. Bring the interfaces up and ping both ways over IPv4 and IPv6.

<details>
<summary>Solution</summary>

```bash
ip link set enp7s0 up
ip addr add 192.168.100.10/24 dev enp7s0
ip addr add fd00:100::10/64 dev enp7s0
ping -c2 192.168.100.20; ping -6 -c2 fd00:100::20
```

</details>

---

## Exercise 3 – Missing prefix ★★

Remove the IPv4 address and add `192.168.100.10` **without** a prefix. Look at `ip addr` and `ip route`. Can you ping `.20`? Explain and fix.

<details>
<summary>Solution</summary>

The address becomes `/32`; no connected route for 192.168.100.0/24 exists, so the kernel uses the default route. Fix: `ip addr del 192.168.100.10/32 dev enp7s0; ip addr add 192.168.100.10/24 dev enp7s0`.

</details>

---

## Exercise 4 – Routes ★★

1. On `rhel1`, create `dummy0` with `10.99.0.1/24` up.
2. On `deb1`, add a route to `10.99.0.0/24` via `192.168.100.20`. Verify with `ip route get 10.99.0.1` and ping it.
3. Replace the route so it uses a non-existent gateway `192.168.100.99`; observe; delete it.

<details>
<summary>Solution</summary>

```bash
# rhel1
ip link add dummy0 type dummy; ip addr add 10.99.0.1/24 dev dummy0; ip link set dummy0 up
# deb1
ip route add 10.99.0.0/24 via 192.168.100.20
ip route get 10.99.0.1; ping -c2 10.99.0.1
ip route replace 10.99.0.0/24 via 192.168.100.99
ping -c2 -W1 10.99.0.1; ip neigh show 192.168.100.99     # FAILED
ip route del 10.99.0.0/24
```

(The ping to 10.99.0.1 works without forwarding because the address belongs to rhel1 itself.)

</details>

---

## Exercise 5 – Link state and MTU ★

Set the spare interface down and up, then set MTU 1400. Prove the MTU with `ping -M do -s 1372` (works) and `-s 1373` (fails).

<details>
<summary>Solution</summary>

```bash
ip link set enp7s0 down; ip -br link show enp7s0; ip link set enp7s0 up
ip link set enp7s0 mtu 1400
ping -c1 -M do -s 1372 192.168.100.20     # 1372 + 28 header bytes = 1400
ping -c1 -M do -s 1373 192.168.100.20     # "message too long"
```

</details>

---

## Exercise 6 – Statistics and events ★

Run `ip monitor` in one terminal, then add/remove an address and bring a link down/up in another. Show RX/TX counters of the interface.

<details>
<summary>Solution</summary>

```bash
ip monitor address link route
ip -s link show enp7s0
```

</details>

---

## Challenge – Prove it's runtime only ★★

Reboot `deb1`. Which of your changes survived? Write the list into `/root/runtime-lost.txt` and explain why.

<details>
<summary>Solution</summary>

None of the `ip` changes survive (addresses, routes, MTU, dummy interface): they only exist in the running kernel. Persistent configuration needs Netplan/nmcli – next lesson.

</details>

---

<!-- footer:start -->
[← Exercises: Network Addresses: IPv4 and IPv6](01-ip-addressing.md) · [Module exercises](README.md) · [Exercises: Persistent Network Configuration: Netplan and NetworkManager →](03-persistent-config.md)
<!-- footer:end -->
