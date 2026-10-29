# Exercises – Swap Space

<!-- nav:start -->
> **Module 05 · Storage** — lesson 5 of 9  
> Read first: [Theory – Swap Space](../../theory/05-storage/05-swap.md) · [Course lesson (commands)](../../course/05-Storage.md#lesson-5---swap-ch-16) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Current state ★

Record total swap, used swap, swap devices with priorities and the current swappiness in `/root/swap.txt`.

<details>
<summary>Solution</summary>

```bash
{ free -h; swapon --show; cat /proc/sys/vm/swappiness; } > /root/swap.txt
```

</details>

---

## Exercise 2 – Swap partition ★

Use a 512 MiB partition of type "Linux swap" (e.g. `/dev/vdb2`), initialize it with label `SWAP2` and enable it permanently by UUID.

**Verify:** `swapoff -a && swapon -a && swapon --show` lists it.

<details>
<summary>Solution</summary>

```bash
mkswap -L SWAP2 /dev/vdb2
echo "UUID=$(blkid -s UUID -o value /dev/vdb2) none swap defaults 0 0" >> /etc/fstab
swapon -a; swapon --show
```

</details>

---

## Exercise 3 – Swap file ★★

> Add 256 MiB of swap from a file `/swapfile`, persistent.

<details>
<summary>Solution</summary>

```bash
dd if=/dev/zero of=/swapfile bs=1M count=256 status=progress   # (fallocate -l 256M also works on ext4)
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo "/swapfile none swap defaults 0 0" >> /etc/fstab
swapon --show
```

</details>

---

## Exercise 4 – Priorities ★★

Give the swap file priority 5 and the partition priority 10 (persistently). Verify the order of use.

<details>
<summary>Solution</summary>

Edit the fstab options to `defaults,pri=10` (partition) and `defaults,pri=5` (file), then:

```bash
swapoff -a && swapon -a && swapon --show     # PRIO column 10 and 5
```

</details>

---

## Exercise 5 – Swap on LVM and resize ★★

Create a 256 MiB swap LV, enable it, then grow it to 512 MiB.

<details>
<summary>Solution</summary>

```bash
lvcreate -n lvswap -L 256M vgapp && mkswap /dev/vgapp/lvswap && swapon /dev/vgapp/lvswap
swapoff /dev/vgapp/lvswap
lvextend -L 512M /dev/vgapp/lvswap
mkswap /dev/vgapp/lvswap && swapon /dev/vgapp/lvswap
swapon --show
```

(`mkswap` changes the UUID – update fstab if it uses UUID.)

</details>

---

## Exercise 6 – Swappiness ★

Set `vm.swappiness` to 10 now and persistently; confirm after `sysctl --system`.

<details>
<summary>Solution</summary>

```bash
sysctl -w vm.swappiness=10
echo "vm.swappiness = 10" > /etc/sysctl.d/90-swappiness.conf
sysctl --system | grep swappiness; cat /proc/sys/vm/swappiness
```

</details>

---

## Exercise 7 – Watch swapping ★★★

On a VM with ~2 GiB RAM, allocate more memory than available (`python3 -c 'a=bytearray(2500*1024*1024); input()'`) and watch `vmstat 1` and `swapon --show`. Which process uses swap? Stop it and watch swap usage afterwards.

<details>
<summary>Solution</summary>

`vmstat` shows `so` (and later `si`) values; `free` swap used grows. Find users: `grep VmSwap /proc/*/status | sort -k2 -n | tail`. After stopping, used swap may decrease only slowly – pages are swapped back in only when accessed.

</details>

---

## Challenge – Remove swap cleanly ★★

Remove the swap file and the swap LV completely: disable, remove from fstab, delete, and confirm nothing references them.

<details>
<summary>Solution</summary>

```bash
swapoff /swapfile && rm /swapfile
swapoff /dev/vgapp/lvswap && lvremove -y /dev/vgapp/lvswap
sed -i '\|/swapfile|d; \|lvswap|d' /etc/fstab
findmnt --verify; swapon --show
```

</details>

---

<!-- footer:start -->
[← Exercises: Logical Volume Management (LVM)](04-lvm.md) · [Module exercises](README.md) · [Exercises: NFS: Network File System →](06-nfs.md)
<!-- footer:end -->
