# Exercises – The Kernel and Boot Parameters

<!-- nav:start -->
> **Module 06 · Kernel, Modules & Devices** — lesson 1 of 4  
> Read first: [Theory – The Kernel and Boot Parameters](../../theory/06-kernel-devices/01-kernel-boot-parameters.md) · [Course lesson (commands)](../../course/06-Kernel-Modules-Devices.md#lesson-1---kernel-information--boot-parameters-ch-24) · [Module exercises](README.md)
<!-- nav:end -->

> Take a VM snapshot before changing GRUB.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Know your kernel ★

Save to `/root/kernel.txt`: running kernel release, installed kernel packages, current command line, and the default boot kernel.

<details>
<summary>Solution</summary>

```bash
{ uname -r; cat /proc/cmdline; } > /root/kernel.txt
rpm -q kernel >> /root/kernel.txt; grubby --default-kernel >> /root/kernel.txt        # RHEL
dpkg -l 'linux-image*' | grep ^ii >> /root/kernel.txt                                 # Debian/Ubuntu
```

</details>

---

## Exercise 2 – Built-in or module? ★

Determine from the kernel config whether ext4, XFS, NBD and the bridge driver are built in or modules.

<details>
<summary>Solution</summary>

```bash
grep -E 'CONFIG_(EXT4_FS|XFS_FS|BLK_DEV_NBD|BRIDGE)=' /boot/config-$(uname -r)
```

</details>

---

## Exercise 3 – Kernel messages ★

1. Show only kernel warnings and errors of this boot with readable timestamps.
2. Find the message where the root filesystem was mounted.

<details>
<summary>Solution</summary>

```bash
dmesg -T -l err,warn
journalctl -k | grep -iE 'EXT4-fs|XFS.*mount'
```

</details>

---

## Exercise 4 – Inspect the initramfs ★★

List the initramfs content of the running kernel and check whether the virtio block driver (or your disk driver) and LVM tools are included.

<details>
<summary>Solution</summary>

```bash
lsinitrd | grep -E 'virtio_blk|lvm'                               # RHEL
lsinitramfs /boot/initrd.img-$(uname -r) | grep -E 'virtio_blk|lvm'   # Debian/Ubuntu
```

</details>

---

## Exercise 5 – One-time boot parameter ★★

Reboot, interrupt GRUB, and boot once with `systemd.unit=multi-user.target` and without `quiet`. After boot, verify with `/proc/cmdline`. Reboot normally and verify it's gone.

<details>
<summary>Solution</summary>

At the GRUB menu: `e` → go to the `linux` line → remove `quiet`, append `systemd.unit=multi-user.target` → `Ctrl+X`. Then `cat /proc/cmdline`.

</details>

---

## Exercise 6 – Persistent parameter on RHEL ★★

On `rhel1`, add the harmless parameter `audit=1` to all installed kernels persistently (and to future kernels), and remove `rhgb`. Verify with `grubby --info=ALL` and after a reboot.

> Avoid practising with `net.ifnames=0`: it renames NICs and can break your network configuration.

<details>
<summary>Solution</summary>

```bash
grubby --update-kernel=ALL --args="audit=1" --remove-args="rhgb"
grubby --info=ALL | grep args
sed -i 's/^GRUB_CMDLINE_LINUX="\(.*\)"/GRUB_CMDLINE_LINUX="\1 audit=1"/' /etc/default/grub   # future kernels
reboot; cat /proc/cmdline
```

</details>

---

## Exercise 7 – Persistent parameter on Ubuntu ★★

On `deb1`, make GRUB show the menu for 10 seconds and add `audit=1` to all boot entries. Verify in `/boot/grub/grub.cfg` and after reboot.

<details>
<summary>Solution</summary>

```bash
sed -i 's/^GRUB_TIMEOUT=.*/GRUB_TIMEOUT=10/; s/^GRUB_TIMEOUT_STYLE=.*/GRUB_TIMEOUT_STYLE=menu/' /etc/default/grub
sed -i 's/^GRUB_CMDLINE_LINUX="\(.*\)"/GRUB_CMDLINE_LINUX="\1 audit=1"/' /etc/default/grub
update-grub
grep -m2 'audit=1' /boot/grub/grub.cfg
```

</details>

---

## Challenge – Boot an older kernel by default ★★★

If two kernels are installed, make the **older** one the default on RHEL (with grubby) and on Ubuntu (with `GRUB_DEFAULT` pointing to the "Advanced options" submenu entry). Reboot, verify with `uname -r`, then restore the newest as default.

<details>
<summary>Solution</summary>

```bash
# RHEL
grubby --info=ALL | grep ^kernel
grubby --set-default=/boot/vmlinuz-<older-version>
# Ubuntu: find menu entry IDs
grep -E "submenu|menuentry '" /boot/grub/grub.cfg | cut -d"'" -f2 | head
# set e.g. GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.8.0-41-generic"
update-grub
```

</details>

---

<!-- footer:start -->
[Module exercises](README.md) · [Exercises: Kernel Parameters with sysctl →](02-sysctl.md)
<!-- footer:end -->
