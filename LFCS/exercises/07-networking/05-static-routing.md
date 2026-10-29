# Exercises – Static Routing

<!-- nav:start -->
> **Module 07 · Networking** — lesson 5 of 11  
> Read first: [Theory – Static Routing](../../theory/07-networking/05-static-routing.md) · [Course lesson (commands)](../../course/07-Networking.md#lesson-5---static-routing-gap) · [Module exercises](README.md)
<!-- nav:end -->

> Topology: `rhel1` (192.168.56.20) is the client; `deb1` (192.168.56.10) is the router to a back-end network `10.99.0.0/24`, simulated with a dummy interface or a network namespace.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Read the routing table ★

Explain every line of `ip route` on both VMs (type of route, gateway, interface, metric, protocol).

---

## Exercise 2 – Longest prefix ★★

On `rhel1` add runtime routes: `10.0.0.0/8 via 192.168.56.10` and `10.99.0.0/24 via 192.168.56.1`. Predict and check with `ip route get` which gateway is used for `10.99.0.5`, `10.5.5.5` and `8.8.8.8`. Remove the routes.

<details>
<summary>Solution</summary>

`10.99.0.5` → 192.168.56.1 (/24 wins) · `10.5.5.5` → 192.168.56.10 (/8) · `8.8.8.8` → default gateway.

</details>

---

## Exercise 3 – Build the back-end network ★★

On `deb1`, create a network namespace acting as a server on `10.99.0.0/24`:

```bash
ip netns add backend
ip link add veth-r type veth peer name veth-b
ip link set veth-b netns backend
ip addr add 10.99.0.1/24 dev veth-r; ip link set veth-r up
ip netns exec backend ip addr add 10.99.0.5/24 dev veth-b
ip netns exec backend ip link set veth-b up
ip netns exec backend ip link set lo up
ip netns exec backend ip route add default via 10.99.0.1
ip netns exec backend ping -c1 10.99.0.1
```

Then enable forwarding on `deb1` persistently.

<details>
<summary>Solution</summary>

```bash
echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/90-router.conf && sysctl --system
```

</details>

---

## Exercise 4 – Persistent route on the client ★★

> Add a persistent static route on `rhel1` to `10.99.0.0/24` via `deb1`.

**Verify:** `ping 10.99.0.5` from `rhel1` works; route present after reboot.

<details>
<summary>Solution</summary>

```bash
nmcli con mod enp1s0 +ipv4.routes "10.99.0.0/24 192.168.56.10"
nmcli con up enp1s0
ip route get 10.99.0.5; ping -c2 10.99.0.5
```

If ping fails, check `deb1`'s firewall (ufw/nftables FORWARD policy).

</details>

---

## Exercise 5 – The return route ★★★

Remove the namespace's default route (`ip netns exec backend ip route del default`). Ping again from `rhel1` and watch with `tcpdump -ni veth-r icmp` on `deb1`. Explain. Fix it in two different ways.

<details>
<summary>Solution</summary>

Requests reach 10.99.0.5, but it has no route back to 192.168.56.0/24, so replies are never sent. Fixes:

1. Return route: `ip netns exec backend ip route add 192.168.56.0/24 via 10.99.0.1`.
2. Masquerade on the router: `nft add table ip nat; nft add chain ip nat post '{ type nat hook postrouting priority 100; }'; nft add rule ip nat post oif "veth-r" masquerade`.

</details>

---

## Exercise 6 – Path tools ★

From `rhel1`, run `tracepath -n 10.99.0.5` and `mtr -n -c 10 --report 10.99.0.5`. Identify each hop.

---

## Exercise 7 – Netplan route ★★

On `deb1`, add a persistent route to `172.30.0.0/16` via `192.168.56.20` with metric 50 using Netplan. Verify with `ip route` (metric shown).

<details>
<summary>Solution</summary>

```yaml
      routes:
        - to: 172.30.0.0/16
          via: 192.168.56.20
          metric: 50
```

`netplan try`, then `ip route | grep 172.30`.

</details>

---

## Challenge – Route metrics and failover ★★★

On `rhel1`, add two routes to `10.99.0.0/24`: via `deb1` metric 100 and via a non-existent `192.168.56.250` metric 50. Which is used? Does Linux automatically fail over when the preferred gateway is dead? Explain.

<details>
<summary>Solution</summary>

The metric-50 route is used, and traffic fails: static routes don't detect dead gateways (no health check). Only a link-down on the route's interface removes it. Dynamic routing (OSPF/BGP with FRR) or monitoring scripts are needed for failover.

</details>

---

## Cleanup

```bash
# deb1
ip netns del backend; rm /etc/sysctl.d/90-router.conf
# rhel1
nmcli con mod enp1s0 -ipv4.routes "10.99.0.0/24 192.168.56.10" && nmcli con up enp1s0
```

---

<!-- footer:start -->
[← Exercises: Hostname and Name Resolution](04-hostname-resolution.md) · [Module exercises](README.md) · [Exercises: Bridges and Bonds →](06-bridges-bonds.md)
<!-- footer:end -->
