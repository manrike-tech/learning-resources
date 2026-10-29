# Exercises – Devices and udev

<!-- nav:start -->
> **Module 06 · Kernel, Modules & Devices** — lesson 4 of 4  
> Read first: [Theory – Devices and udev](../../theory/06-kernel-devices/04-devices-udev.md) · [Course lesson (commands)](../../course/06-Kernel-Modules-Devices.md#lesson-4---devices--udev-ch-26) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Device files ★

1. Show type, major and minor of your first disk, its first partition, `/dev/null` and `/dev/tty1`.
2. Which driver owns your disk's major number?
3. Create your own null device `/tmp/mynull` and prove it works.

<details>
<summary>Solution</summary>

```bash
ls -l /dev/vda /dev/vda1 /dev/null /dev/tty1
grep -w 252 /proc/devices          # use the major number shown by ls -l (e.g. 252 → virtblk, 8 → sd)
mknod /tmp/mynull c 1 3 && chmod 666 /tmp/mynull
echo test > /tmp/mynull; cat /tmp/mynull; rm /tmp/mynull
```

</details>

---

## Exercise 2 – Persistent names ★

List every symlink under `/dev/disk/` that points to your second extra disk or its partitions.

<details>
<summary>Solution</summary>

```bash
find /dev/disk -lname '*vdc*' -printf '%p -> %l\n'
udevadm info --query=symlink --name=/dev/vdc
```

</details>

---

## Exercise 3 – Inspect with udevadm ★★

For the second extra disk, find: `ID_SERIAL` (or `ID_PATH`), `DEVTYPE`, its size attribute, and its parent's driver.

<details>
<summary>Solution</summary>

```bash
udevadm info --query=property --name=/dev/vdc | grep -E 'ID_SERIAL=|ID_PATH=|DEVTYPE='
udevadm info -a -n /dev/vdc | grep -E 'ATTR\{size\}|DRIVERS==' | head -3
```

(virtio disks get a serial only if the VM defines one, e.g. `<serial>data02</serial>` in libvirt; otherwise use `ID_PATH`.)

</details>

---

## Exercise 4 – Watch events ★

Run `udevadm monitor --udev --property` and, in another shell, attach and detach a loop device. Identify `ACTION`, `DEVNAME` and `SUBSYSTEM` in the output.

<details>
<summary>Solution</summary>

```bash
truncate -s 50M /root/x.img
L=$(losetup -f --show /root/x.img); losetup -d $L
```

</details>

---

## Exercise 5 – Exam-style symlink rule ★★

> Create a udev rule that adds the symlink `/dev/backupdisk` for your second extra disk, owned by group `disk` with mode `0660`.

**Verify:** `ls -l /dev/backupdisk` and `ls -l $(readlink -f /dev/backupdisk)`; still present after reboot.

<details>
<summary>Solution</summary>

```bash
udevadm info --query=property --name=/dev/vdc | grep -E 'ID_SERIAL=|ID_PATH='
cat > /etc/udev/rules.d/99-backupdisk.rules << 'EOF'
SUBSYSTEM=="block", ENV{DEVTYPE}=="disk", ENV{ID_PATH}=="pci-0000:00:09.0", SYMLINK+="backupdisk", GROUP="disk", MODE="0660"
EOF
udevadm control --reload
udevadm trigger --name-match=/dev/vdc
ls -l /dev/backupdisk
```

(Use the `ID_SERIAL` or `ID_PATH` value from your own system.)

</details>

---

## Exercise 6 – Run a script on add ★★

Log `"<date> added <kernel name>"` to `/var/log/blockdev.log` every time a **loop** block device is added. Test by attaching a loop device.

<details>
<summary>Solution</summary>

```bash
cat > /usr/local/bin/log-blockdev.sh << 'EOF'
#!/bin/sh
echo "$(date '+%F %T') added $1" >> /var/log/blockdev.log
EOF
chmod 755 /usr/local/bin/log-blockdev.sh
echo 'ACTION=="add", SUBSYSTEM=="block", KERNEL=="loop[0-9]*", ENV{DEVTYPE}=="disk", RUN+="/usr/local/bin/log-blockdev.sh %k"' \
  > /etc/udev/rules.d/99-log-loop.rules
udevadm control --reload
L=$(losetup -f --show /root/x.img); losetup -d $L; cat /var/log/blockdev.log
```

</details>

---

## Exercise 7 – Debug a broken rule ★★

This rule never matches. Find two mistakes with `udevadm test`, fix it.

```text
SUBSYSTEM="block", KERNEL=="vdc", ATTRS{size}=="10485760", ATTRS{driver}=="virtio_blk", SYMLINK+="mydisk"
```

<details>
<summary>Solution</summary>

1. `SUBSYSTEM="block"` uses `=` (assignment) instead of `==`.
2. `ATTRS{size}` (on the disk) and the driver (on the parent) belong to different devices; also the driver is matched with `DRIVERS==`, not `ATTRS{driver}`. Use `ATTR{size}` for the device itself:

```text
SUBSYSTEM=="block", KERNEL=="vdc", ATTR{size}=="10485760", SYMLINK+="mydisk"
```

`udevadm test /sys/class/block/vdc 2>&1 | grep -i mydisk`

</details>

---

## Challenge – Rename a network interface ★★★

On a VM with a second NIC, rename it to `lab0` based on its MAC address, persistently. Reboot and update the NetworkManager/netplan configuration for the new name.

<details>
<summary>Solution</summary>

```bash
ip -br link       # note the MAC of the 2nd NIC
cat > /etc/systemd/network/10-lab0.link << 'EOF'
[Match]
MACAddress=52:54:00:aa:bb:cc
[Link]
Name=lab0
EOF
# (or udev: SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="52:54:00:aa:bb:cc", NAME="lab0")
dracut -f 2>/dev/null || update-initramfs -u     # the .link file may be applied from the initramfs
reboot; ip -br link
```

</details>

---

## Cleanup

```bash
rm -f /etc/udev/rules.d/99-backupdisk.rules /etc/udev/rules.d/99-log-loop.rules /root/x.img
udevadm control --reload
```

---

<!-- footer:start -->
[← Exercises: Kernel Modules](03-kernel-modules.md) · [Module exercises](README.md)
<!-- footer:end -->
