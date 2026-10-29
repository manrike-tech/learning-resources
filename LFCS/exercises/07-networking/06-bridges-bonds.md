# Exercises – Bridges and Bonds

<!-- nav:start -->
> **Module 07 · Networking** — lesson 6 of 11  
> Read first: [Theory – Bridges and Bonds](../../theory/07-networking/06-bridges-bonds.md) · [Course lesson (commands)](../../course/07-Networking.md#lesson-6---bridges--bonds-gap) · [Module exercises](README.md)
<!-- nav:end -->

> Add **two extra NICs** to `rhel1` (e.g. `enp8s0`, `enp9s0`) and one to `deb1` (`enp7s0`) on the lab network in your hypervisor. Work from the VM console.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Runtime bridge ★

On `deb1`, create `br0` with `ip`, add `enp7s0` to it, give `br0` the address `192.168.56.31/24`. Ping it from `rhel1`. Show `bridge link`. Delete everything.

<details>
<summary>Solution</summary>

```bash
ip link add br0 type bridge
ip link set enp7s0 master br0; ip link set enp7s0 up
ip addr add 192.168.56.31/24 dev br0; ip link set br0 up
bridge link show; bridge fdb show br br0 | head
ip link del br0
```

</details>

---

## Exercise 2 – Persistent bridge with Netplan ★★

> Create bridge `br0` with an IP (`192.168.56.30/24`) using `enp7s0`, persistently.

**Verify:** after reboot, `ip -br addr show br0` and `bridge link`.

<details>
<summary>Solution</summary>

```yaml
# /etc/netplan/70-br0.yaml (mode 600)
network:
  version: 2
  ethernets:
    enp7s0: {}
  bridges:
    br0:
      interfaces: [enp7s0]
      addresses: [192.168.56.30/24]
      parameters:
        stp: false
```

`netplan try`.

</details>

---

## Exercise 3 – Persistent bond with nmcli ★★

> Create `bond0` (active-backup) using two spare NICs, IP `192.168.56.40/24`.

**Verify:** `cat /proc/net/bonding/bond0` shows both members `up` and the mode.

<details>
<summary>Solution</summary>

```bash
nmcli con add type bond ifname bond0 con-name bond0 \
  bond.options "mode=active-backup,miimon=100" \
  ipv4.method manual ipv4.addresses 192.168.56.40/24 ipv6.method disabled
nmcli con add type ethernet ifname enp8s0 con-name bond0-p1 port-type bond controller bond0
nmcli con add type ethernet ifname enp9s0 con-name bond0-p2 port-type bond controller bond0
nmcli con up bond0
cat /proc/net/bonding/bond0
```

(Older NetworkManager: `type bond-slave ifname enp8s0 master bond0`.)

</details>

---

## Exercise 4 – Failover test ★★

From `deb1`, run `ping 192.168.56.40` continuously. On `rhel1`, take the active member down (`ip link set <active> down` or disconnect it in the hypervisor). Record how many packets were lost and which member became active. Bring it back.

<details>
<summary>Solution</summary>

```bash
grep "Currently Active" /proc/net/bonding/bond0
nmcli dev disconnect enp8s0      # or ip link set enp8s0 down
grep -E "Currently Active|Link Failure" /proc/net/bonding/bond0
nmcli dev connect enp8s0
```

Usually 0–2 lost packets.

</details>

---

## Exercise 5 – Primary member ★

Make `enp8s0` the primary member so it becomes active again whenever it is up.

<details>
<summary>Solution</summary>

```bash
nmcli con mod bond0 bond.options "mode=active-backup,miimon=100,primary=enp8s0"
nmcli con up bond0
grep Primary /proc/net/bonding/bond0
```

</details>

---

## Exercise 6 – Modes ★

Explain in `/root/bond-modes.txt` which mode you'd use: (a) two switches without stacking, no switch config possible; (b) a switch with LACP configured; (c) a lab where you want simple round-robin. What is required on the switch for (b)?

<details>
<summary>Solution</summary>

(a) `active-backup` · (b) `802.3ad` (LACP enabled on the switch ports, same LAG) · (c) `balance-rr` (needs static aggregation on the switch to work well).

</details>

---

## Challenge – Bond under a bridge ★★★

On `rhel1`, delete `bond0`'s IP configuration and put `bond0` into a bridge `br1` that holds `192.168.56.41/24` – all persistent with nmcli. Verify connectivity and failover still work.

<details>
<summary>Solution</summary>

```bash
nmcli con add type bridge ifname br1 con-name br1 ipv4.method manual ipv4.addresses 192.168.56.41/24 bridge.stp no
nmcli con mod bond0 ipv4.method disabled ipv6.method disabled port-type bridge controller br1
nmcli con up bond0; nmcli con up br1
bridge link; cat /proc/net/bonding/bond0
```

</details>

---

## Cleanup

```bash
nmcli con delete br1 bond0 bond0-p1 bond0-p2 2>/dev/null
rm -f /etc/netplan/70-br0.yaml && netplan apply
```

---

<!-- footer:start -->
[← Exercises: Static Routing](05-static-routing.md) · [Module exercises](README.md) · [Exercises: Packet Filtering, Port Redirection and NAT →](07-firewalls-nat.md)
<!-- footer:end -->
