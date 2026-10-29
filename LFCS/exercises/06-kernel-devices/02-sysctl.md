# Exercises – Kernel Parameters with sysctl

<!-- nav:start -->
> **Module 06 · Kernel, Modules & Devices** — lesson 2 of 4  
> Read first: [Theory – Kernel Parameters with sysctl](../../theory/06-kernel-devices/02-sysctl.md) · [Course lesson (commands)](../../course/06-Kernel-Modules-Devices.md#lesson-2---sysctl-kernel-parameters-ch-24) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Three ways to read ★

Read `net.ipv4.ip_forward` with `sysctl`, with `sysctl -n` and from `/proc/sys`.

<details>
<summary>Solution</summary>

```bash
sysctl net.ipv4.ip_forward; sysctl -n net.ipv4.ip_forward; cat /proc/sys/net/ipv4/ip_forward
```

</details>

---

## Exercise 2 – Non-persistent ★

Enable IPv4 forwarding **without** making it persistent. Verify, reboot, verify it's off again.

<details>
<summary>Solution</summary>

```bash
sysctl -w net.ipv4.ip_forward=1; sysctl net.ipv4.ip_forward
reboot
sysctl net.ipv4.ip_forward     # 0 (unless a file or service sets it)
```

</details>

---

## Exercise 3 – Persistent ★

Make `vm.swappiness=10` persistent and active now.

**Verify:** `sysctl vm.swappiness` now and after reboot.

<details>
<summary>Solution</summary>

```bash
echo "vm.swappiness = 10" > /etc/sysctl.d/90-swappiness.conf
sysctl --system
```

</details>

---

## Exercise 4 – Ignore pings ★★

Make the system ignore ICMP echo requests persistently. Test with `ping` from the other VM. Revert completely.

<details>
<summary>Solution</summary>

```bash
echo "net.ipv4.icmp_echo_ignore_all = 1" > /etc/sysctl.d/90-noping.conf && sysctl --system
# from other VM: ping -c2 <this host>   → 100% packet loss
rm /etc/sysctl.d/90-noping.conf && sysctl -w net.ipv4.icmp_echo_ignore_all=0
```

</details>

---

## Exercise 5 – Mixed (exam-style) ★★

> Set `net.ipv4.ip_forward=1` persistently and `kernel.sysrq=0` **only** at runtime.

<details>
<summary>Solution</summary>

```bash
echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/90-fwd.conf
sysctl --system
sysctl -w kernel.sysrq=0           # after --system, so it isn't overwritten
sysctl net.ipv4.ip_forward kernel.sysrq
```

</details>

---

## Exercise 6 – Who wins? ★★

Create `/etc/sysctl.d/10-a.conf` with `vm.swappiness = 30` and `/etc/sysctl.d/95-b.conf` with `vm.swappiness = 40`. Predict the value after `sysctl --system`; check. Then find every file that mentions `swappiness`.

<details>
<summary>Solution</summary>

40 – files are applied in name order (90-swappiness.conf=10, then 95-b=40; 10-a applied first). Check:

```bash
sysctl --system | grep -B1 swappiness
grep -r swappiness /usr/lib/sysctl.d /etc/sysctl.d /etc/sysctl.conf
systemd-analyze cat-config sysctl.d | grep -B5 swappiness
rm /etc/sysctl.d/10-a.conf /etc/sysctl.d/95-b.conf
```

</details>

---

## Exercise 7 – Per-interface IPv6 ★★

Disable IPv6 only on your second network interface (e.g. `enp7s0`), persistently. Verify with `ip -6 addr show dev <if>`.

<details>
<summary>Solution</summary>

```bash
echo "net.ipv6.conf.enp7s0.disable_ipv6 = 1" > /etc/sysctl.d/90-noipv6-enp7s0.conf
sysctl --system; ip -6 addr show dev enp7s0
```

(Some network managers re-apply IPv6 settings when the interface comes up; with NetworkManager you can also use `nmcli con mod … ipv6.method disabled`.)

</details>

---

## Challenge – Module-dependent key ★★★

Persistently set `net.bridge.bridge-nf-call-iptables = 1` so that it survives a reboot even though the key only exists after `br_netfilter` is loaded.

<details>
<summary>Solution</summary>

```bash
echo br_netfilter > /etc/modules-load.d/br_netfilter.conf
modprobe br_netfilter
echo "net.bridge.bridge-nf-call-iptables = 1" > /etc/sysctl.d/90-bridge.conf
sysctl --system
reboot; lsmod | grep br_netfilter; sysctl net.bridge.bridge-nf-call-iptables
```

`systemd-modules-load` runs before `systemd-sysctl` at boot, so the key exists when the file is applied.

</details>

---

<!-- footer:start -->
[← Exercises: The Kernel and Boot Parameters](01-kernel-boot-parameters.md) · [Module exercises](README.md) · [Exercises: Kernel Modules →](03-kernel-modules.md)
<!-- footer:end -->
