# Exercises – Kernel Modules

<!-- nav:start -->
> **Module 06 · Kernel, Modules & Devices** — lesson 3 of 4  
> Read first: [Theory – Kernel Modules](../../theory/06-kernel-devices/03-kernel-modules.md) · [Course lesson (commands)](../../course/06-Kernel-Modules-Devices.md#lesson-3---kernel-modules-ch-25) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Explore ★

1. Count loaded modules and find the one with the highest "Used by" count.
2. Show the description and parameters of `nbd` and the dependencies of `nfs` (or `xfs`).
3. Which driver does your network card use?

<details>
<summary>Solution</summary>

```bash
lsmod | tail -n +2 | wc -l
lsmod | tail -n +2 | sort -k3 -n | tail -1
modinfo -F description nbd; modinfo -p nbd; modinfo -F depends nfs
lspci -k | grep -A3 -i ethernet
```

</details>

---

## Exercise 2 – Load and unload ★

Load `nbd` with `max_part=4`, verify the parameter, unload it.

<details>
<summary>Solution</summary>

```bash
modprobe nbd max_part=4
cat /sys/module/nbd/parameters/max_part
modprobe -r nbd; lsmod | grep nbd
```

</details>

---

## Exercise 3 – Persistent load with options ★★

> Load the `nbd` module with `max_part=4` at boot and verify the parameter in `/sys/module`.

<details>
<summary>Solution</summary>

```bash
echo nbd > /etc/modules-load.d/nbd.conf
echo "options nbd max_part=4" > /etc/modprobe.d/nbd.conf
reboot
lsmod | grep nbd; cat /sys/module/nbd/parameters/max_part
```

</details>

---

## Exercise 4 – Dependencies ★★

Use a dry run to see which modules `modprobe nfs` would load. Load it, then try `modprobe -r sunrpc`. Explain the result.

<details>
<summary>Solution</summary>

```bash
modprobe -n -v nfs
modprobe nfs
modprobe -r sunrpc     # FATAL: Module sunrpc is in use (nfs depends on it)
modprobe -r nfs        # removes nfs and unused dependencies
```

</details>

---

## Exercise 5 – Blacklist vs install ★★

1. Blacklist `pcspkr` (or `floppy`). Reboot and confirm it isn't loaded.
2. Run `modprobe pcspkr`. Does it load? Why?
3. Hard-block it so manual loading fails.

<details>
<summary>Solution</summary>

```bash
echo "blacklist pcspkr" > /etc/modprobe.d/blacklist-pcspkr.conf
dracut -f 2>/dev/null || update-initramfs -u
reboot; lsmod | grep pcspkr               # not loaded
modprobe pcspkr; lsmod | grep pcspkr      # loads: blacklist only stops alias-based auto-loading
modprobe -r pcspkr
echo "install pcspkr /bin/false" >> /etc/modprobe.d/blacklist-pcspkr.conf
modprobe pcspkr                           # error
```

</details>

---

## Exercise 6 – Effective configuration ★

Show all `options`, `blacklist` and `install` directives that modprobe currently uses that come from your files.

<details>
<summary>Solution</summary>

```bash
modprobe -c | grep -E '^(options|blacklist|install) (nbd|pcspkr)'
```

</details>

---

## Challenge – USB storage lockdown ★★★

Security policy: USB mass storage must never work on this server, but other USB devices (keyboards) must. Implement it, rebuild the initramfs, and prove that `modprobe usb-storage` fails. Then document how you'd undo it.

<details>
<summary>Solution</summary>

```bash
cat > /etc/modprobe.d/usb-storage.conf << 'EOF'
blacklist usb-storage
blacklist uas
install usb-storage /bin/false
install uas /bin/false
EOF
dracut -f 2>/dev/null || update-initramfs -u
modprobe -r uas usb-storage 2>/dev/null
modprobe usb-storage; echo "exit=$?"
```

Undo: delete the file, rebuild the initramfs, `modprobe usb-storage`.

</details>

---

<!-- footer:start -->
[← Exercises: Kernel Parameters with sysctl](02-sysctl.md) · [Module exercises](README.md) · [Exercises: Devices and udev →](04-devices-udev.md)
<!-- footer:end -->
