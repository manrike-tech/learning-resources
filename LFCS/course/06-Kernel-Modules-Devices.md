# Module 06 - Kernel, Modules & Devices

**LFS207:** Ch. 24 Kernel Services and Configuration · Ch. 25 Kernel Modules · Ch. 26 Devices and udev
**Exam domain:** Operations & Deployment (25%)  
**Theory:** [Module 06 theory notes](../theory/06-kernel-devices/README.md) · **Exercises:** [Module 06 exercises](../exercises/06-kernel-devices/README.md)  

---

## Learning Objectives

- Read and set kernel parameters at runtime and persistently.
- Manage kernel modules and their options.
- Understand device files and write udev rules.

---

## Lesson 1 - Kernel Information & Boot Parameters (Ch. 24)

> **Theory:** [The Kernel and Boot Parameters](../theory/06-kernel-devices/01-kernel-boot-parameters.md) · **Exercises:** [practice tasks](../exercises/06-kernel-devices/01-kernel-boot-parameters.md)

```bash
uname -r; uname -a
cat /proc/cmdline            # parameters the kernel booted with
dmesg -T | tail; journalctl -k
ls /boot
```

Change boot parameters persistently:

```bash
# Debian/Ubuntu: edit GRUB_CMDLINE_LINUX in /etc/default/grub, then
update-grub
# RHEL family: edit /etc/default/grub then
grub2-mkconfig -o /boot/grub2/grub.cfg
# or (RHEL, BLS):
grubby --update-kernel=ALL --args="quiet"
grubby --info=DEFAULT
```

---

## Lesson 2 - sysctl: Kernel Parameters (Ch. 24)

> **Theory:** [Kernel Parameters with sysctl](../theory/06-kernel-devices/02-sysctl.md) · **Exercises:** [practice tasks](../exercises/06-kernel-devices/02-sysctl.md)

`/proc/sys/` exposes tunables; `net/ipv4/ip_forward` ↔ `net.ipv4.ip_forward`.

```bash
sysctl -a | grep forward
sysctl net.ipv4.ip_forward
cat /proc/sys/net/ipv4/ip_forward
```

**Non-persistent** (lost at reboot):

```bash
sysctl -w net.ipv4.ip_forward=1
echo 1 > /proc/sys/net/ipv4/ip_forward
```

**Persistent:**

```bash
echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/90-forward.conf
sysctl --system                     # reload all config files
sysctl -p /etc/sysctl.d/90-forward.conf
```

Common parameters: `net.ipv4.ip_forward`, `net.ipv6.conf.all.forwarding`, `vm.swappiness`, `fs.file-max`, `kernel.hostname`, `net.ipv4.icmp_echo_ignore_all`, `vm.dirty_ratio`.

> **Exam tip:** the competency explicitly says "persistent **and** non-persistent". Read each task carefully: do they want it to survive a reboot?

---

## Lesson 3 - Kernel Modules (Ch. 25)

> **Theory:** [Kernel Modules](../theory/06-kernel-devices/03-kernel-modules.md) · **Exercises:** [practice tasks](../exercises/06-kernel-devices/03-kernel-modules.md)

```bash
lsmod                         # loaded modules
modinfo e1000                 # info, parameters (parm:)
modprobe nbd                  # load with dependencies
modprobe -r nbd               # unload
modprobe nbd max_part=8       # with parameter
insmod / rmmod                # low-level, no dependencies (avoid)
ls /sys/module/nbd/parameters/
```

Persistent configuration:

```bash
# Load at boot
echo nbd > /etc/modules-load.d/nbd.conf
# Options
echo "options nbd max_part=8" > /etc/modprobe.d/nbd.conf
# Blacklist (prevent auto-load)
echo "blacklist pcspkr" > /etc/modprobe.d/blacklist-pcspkr.conf
# Hard block even manual/dependency loads:
echo "install pcspkr /bin/false" >> /etc/modprobe.d/blacklist-pcspkr.conf
```

After changing modules needed at early boot, rebuild initramfs: `update-initramfs -u` (Debian) / `dracut -f` (RHEL).

---

## Lesson 4 - Devices & udev (Ch. 26)

> **Theory:** [Devices and udev](../theory/06-kernel-devices/04-devices-udev.md) · **Exercises:** [practice tasks](../exercises/06-kernel-devices/04-devices-udev.md)

- Device files in `/dev`: **block** (`b`, disks) and **character** (`c`, terminals, serial). `ls -l /dev/sda` shows major, minor numbers.
- **udev** (`systemd-udevd`) creates device nodes dynamically, names them, sets permissions, creates symlinks (`/dev/disk/by-uuid/`, `by-id`, `by-path`).

```bash
udevadm info --query=all --name=/dev/vdb
udevadm info -a -n /dev/vdb        # attributes to match in rules
udevadm monitor                    # watch events (plug a USB)
udevadm control --reload
udevadm trigger
lsusb; lspci; lsblk
```

### udev rules

Files in `/etc/udev/rules.d/` (override `/usr/lib/udev/rules.d/`). Named `NN-name.rules`.

```
# /etc/udev/rules.d/99-mydisk.rules
# Symlink + permissions for a disk identified by serial
SUBSYSTEM=="block", ENV{ID_SERIAL}=="QEMU_HARDDISK_data01", SYMLINK+="datadisk", GROUP="disk", MODE="0660"

# Run a script when a USB storage device is added
ACTION=="add", SUBSYSTEM=="block", ENV{ID_BUS}=="usb", RUN+="/usr/local/bin/usb-added.sh"
```

`==` match · `=` assign · `+=` append.

---

## Lab 06

1. Enable IPv4 forwarding immediately **without** making it persistent; verify; reboot; verify it's off.
2. Make `vm.swappiness=10` persistent.
3. Ignore ICMP echo requests persistently; test with `ping` from the other VM; revert.
4. Load the `nbd` module with `max_part=4` at boot and verify the parameter in `/sys/module`.
5. Blacklist the `floppy` (or `pcspkr`) module.
6. Create a udev rule that adds the symlink `/dev/backupdisk` for your second extra disk.

## Self-check

- [ ] I can set a kernel parameter both ways and know where persistent files go.
- [ ] I know `modules-load.d` vs `modprobe.d`.
- [ ] I can find a device's attributes with `udevadm` and write a matching rule.
