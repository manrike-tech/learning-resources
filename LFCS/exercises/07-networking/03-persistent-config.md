# Exercises – Persistent Network Configuration

<!-- nav:start -->
> **Module 07 · Networking** — lesson 3 of 11  
> Read first: [Theory – Persistent Network Configuration: Netplan and NetworkManager](../../theory/07-networking/03-persistent-config.md) · [Course lesson (commands)](../../course/07-Networking.md#lesson-3---persistent-configuration) · [Module exercises](README.md)
<!-- nav:end -->

> Work on the spare interface (`enp7s0`) unless you have console access. Lab network `192.168.100.0/24`, IPv6 `fd00:100::/64`.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Who manages what? ★

For each VM, find which network service manages each interface and where its configuration is stored.

<details>
<summary>Solution</summary>

```bash
systemctl is-active NetworkManager systemd-networkd
ls -l /etc/netplan/ 2>/dev/null; networkctl list 2>/dev/null
nmcli device status 2>/dev/null; ls /etc/NetworkManager/system-connections/ 2>/dev/null
```

</details>

---

## Exercise 2 – Netplan static (deb1) ★★

Give `enp7s0` the static addresses `192.168.100.10/24` and `fd00:100::10/64`, DNS server `192.168.100.20`, search domain `example.lab`, no DHCP. Apply with rollback protection.

**Verify:** `ip -br addr show enp7s0`, `resolvectl status enp7s0`; reboot and check again.

<details>
<summary>Solution</summary>

```bash
cat > /etc/netplan/60-lab.yaml << 'EOF'
network:
  version: 2
  ethernets:
    enp7s0:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.100.10/24
        - fd00:100::10/64
      nameservers:
        addresses: [192.168.100.20]
        search: [example.lab]
EOF
chmod 600 /etc/netplan/60-lab.yaml
netplan generate && netplan try
```

</details>

---

## Exercise 3 – nmcli static (rhel1) ★★

Create a connection profile `lab` on `enp7s0` with `192.168.100.20/24`, `fd00:100::20/64`, DNS `192.168.100.10`, search `example.lab`. Activate it.

**Verify:** `nmcli con show lab | grep -E 'ipv[46]\.(method|addresses|dns)'`, `ip -br addr show enp7s0`; reboot.

<details>
<summary>Solution</summary>

```bash
nmcli con add type ethernet ifname enp7s0 con-name lab \
  ipv4.method manual ipv4.addresses 192.168.100.20/24 ipv4.dns 192.168.100.10 ipv4.dns-search example.lab \
  ipv6.method manual ipv6.addresses fd00:100::20/64
nmcli con up lab
```

</details>

---

## Exercise 4 – Modify, don't forget to apply ★

Change the `lab` profile's IPv4 address to `192.168.100.21/24`. Check `ip addr` before and after `nmcli con up lab`. Then change it back.

<details>
<summary>Solution</summary>

```bash
nmcli con mod lab ipv4.addresses 192.168.100.21/24
ip -br addr show enp7s0      # still .20
nmcli con up lab
ip -br addr show enp7s0      # .21
nmcli con mod lab ipv4.addresses 192.168.100.20/24 && nmcli con up lab
```

</details>

---

## Exercise 5 – Inspect the keyfile ★

Show the keyfile of `lab` and identify each setting. Edit the file by hand to add `dns-search=lab.local;` and make NetworkManager use it.

<details>
<summary>Solution</summary>

```bash
cat /etc/NetworkManager/system-connections/lab.nmconnection
sed -i 's/^dns-search=example.lab;/dns-search=example.lab;lab.local;/' /etc/NetworkManager/system-connections/lab.nmconnection
nmcli con reload && nmcli con up lab
nmcli -g ipv4.dns-search con show lab
```

</details>

---

## Exercise 6 – DHCP profile and switching ★★

Create a second profile `lab-dhcp` for the same interface with DHCP and `autoconnect no`. Switch between `lab` and `lab-dhcp`. Which one is used after reboot?

<details>
<summary>Solution</summary>

```bash
nmcli con add type ethernet ifname enp7s0 con-name lab-dhcp ipv4.method auto connection.autoconnect no
nmcli con up lab-dhcp; nmcli con up lab
```

`lab` (autoconnect yes) is activated at boot.

</details>

---

## Challenge – Exam-style ★★★

> On `rhel1`, configure the lab NIC persistently with IPv4 `192.168.56.20/24` and IPv6 `fd00:56::20/64`, default gateway `192.168.56.1`, DNS `192.168.56.1`, and an additional route to `10.50.0.0/16` via `192.168.56.1`.
> Do the equivalent on `deb1` with Netplan (`.10`).

(Apply on the main lab interface only from the VM console.)

<details>
<summary>Solution</summary>

```bash
# rhel1
nmcli con mod enp1s0 ipv4.method manual ipv4.addresses 192.168.56.20/24 ipv4.gateway 192.168.56.1 \
  ipv4.dns 192.168.56.1 +ipv4.routes "10.50.0.0/16 192.168.56.1" \
  ipv6.method manual ipv6.addresses fd00:56::20/64
nmcli con up enp1s0
```

```yaml
# deb1: /etc/netplan/60-static.yaml
network:
  version: 2
  ethernets:
    enp1s0:
      dhcp4: false
      addresses: [192.168.56.10/24, "fd00:56::10/64"]
      routes:
        - to: default
          via: 192.168.56.1
        - to: 10.50.0.0/16
          via: 192.168.56.1
      nameservers:
        addresses: [192.168.56.1]
```

</details>

---

<!-- footer:start -->
[← Exercises: Runtime Network Configuration with ip](02-runtime-ip.md) · [Module exercises](README.md) · [Exercises: Hostname and Name Resolution →](04-hostname-resolution.md)
<!-- footer:end -->
