# Exercises – Setting and Synchronizing System Time

<!-- nav:start -->
> **Module 07 · Networking** — lesson 9 of 11  
> Read first: [Theory – Setting and Synchronizing System Time](../../theory/07-networking/09-time-sync.md) · [Course lesson (commands)](../../course/07-Networking.md#lesson-9---time-synchronization-gap) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Status and time zone ★

1. Record `timedatectl` output of both VMs.
2. Set the time zone to `Europe/Madrid` (or yours). Compare `date` and `date -u`.
3. Show the current time in Tokyo without changing the system zone.

<details>
<summary>Solution</summary>

```bash
timedatectl
timedatectl set-timezone Europe/Madrid; date; date -u
TZ=Asia/Tokyo date
```

</details>

---

## Exercise 2 – Manual time and correction ★★

1. Disable NTP, set the clock 10 minutes in the past.
2. Re-enable NTP. Watch `chronyc tracking` – is the time stepped or slewed? Force a step.

<details>
<summary>Solution</summary>

```bash
timedatectl set-ntp false
timedatectl set-time "$(date -d '-10 min' '+%F %T')"
timedatectl set-ntp true
chronyc tracking | grep "System time"
chronyc makestep
```

With `makestep 1.0 3`, only the first updates after start may step; afterwards a big offset is slewed slowly unless you run `chronyc makestep`.

</details>

---

## Exercise 3 – Read chronyc ★

Run `chronyc sources -v` and `chronyc tracking`. Identify: the selected source, its stratum, the reach value and the current offset.

---

## Exercise 4 – NTP server (exam-style) ★★

> Make `deb1` an NTP server for the lab subnet and `rhel1` its client.

**Verify:** on `rhel1`, `chronyc sources` shows `^* 192.168.56.10`; on `deb1`, `chronyc clients` lists `rhel1`.

<details>
<summary>Solution</summary>

```bash
# deb1
apt install -y chrony
echo "allow 192.168.56.0/24" >> /etc/chrony/chrony.conf
systemctl restart chrony
ufw allow from 192.168.56.0/24 to any port 123 proto udp 2>/dev/null
# rhel1
sed -i 's/^pool /#pool /; s/^server /#server /' /etc/chrony.conf
echo "server 192.168.56.10 iburst" >> /etc/chrony.conf
systemctl restart chronyd
chronyc sources
# deb1
chronyc clients
```

</details>

---

## Exercise 5 – Isolated lab server ★★

Disconnect `deb1` from the internet (or block outgoing UDP 123). The client loses sync. Configure `deb1` to keep serving time anyway.

<details>
<summary>Solution</summary>

Add `local stratum 10` to `deb1`'s chrony.conf and restart. Clients then accept `deb1` as a stratum 10 source.

</details>

---

## Exercise 6 – systemd-timesyncd ★

On a VM without chrony (or a fresh Ubuntu container/VM), configure `systemd-timesyncd` to use `deb1`, and verify with `timedatectl timesync-status`.

<details>
<summary>Solution</summary>

```bash
mkdir -p /etc/systemd/timesyncd.conf.d
printf '[Time]\nNTP=192.168.56.10\n' > /etc/systemd/timesyncd.conf.d/lab.conf
systemctl restart systemd-timesyncd
timedatectl timesync-status
```

</details>

---

## Challenge – Hardware clock ★★

1. Show the RTC time and whether it's in UTC.
2. Explain what `rtcsync` does and what `hwclock --systohc` does.
3. Set the RTC to local time (`timedatectl set-local-rtc 1`), read the warning, revert.

<details>
<summary>Solution</summary>

`hwclock --show`, `timedatectl | grep RTC`. `rtcsync` lets the kernel copy system time to the RTC every 11 minutes; `hwclock --systohc` does it once now. Local-RTC mode produces a warning about DST problems – revert with `timedatectl set-local-rtc 0`.

</details>

---

<!-- footer:start -->
[← Exercises: The OpenSSH Server and Client](08-openssh.md) · [Module exercises](README.md) · [Exercises: Reverse Proxies and Load Balancers →](10-reverse-proxy.md)
<!-- footer:end -->
