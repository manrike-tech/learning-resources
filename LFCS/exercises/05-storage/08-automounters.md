# Exercises – Filesystem Automounters

<!-- nav:start -->
> **Module 05 · Storage** — lesson 8 of 9  
> Read first: [Theory – Filesystem Automounters](../../theory/05-storage/08-automounters.md) · [Course lesson (commands)](../../course/05-Storage.md#lesson-8---automounters-gap) · [Module exercises](README.md)
<!-- nav:end -->

> Requires the NFS server from the [NFS exercises](06-nfs.md) (`rhel1:/exports/share`). Remove any static NFS line for these paths from the client's fstab first.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Indirect map ★

On the client, configure autofs so that `/mnt/auto/share` mounts `rhel1:/exports/share` on access, unmounting after 60 s idle. Use a drop-in in `/etc/auto.master.d/`.

**Verify:** `ls /mnt/auto/share` works; `findmnt /mnt/auto/share` shows nfs4; after ~60 s idle it disappears.

<details>
<summary>Solution</summary>

```bash
apt install -y autofs || dnf install -y autofs
echo "/mnt/auto /etc/auto.nfs --timeout=60" > /etc/auto.master.d/nfs.autofs
echo "share -rw,sync rhel1:/exports/share" > /etc/auto.nfs
systemctl enable --now autofs; systemctl reload autofs
ls /mnt/auto/share; findmnt /mnt/auto/share
```

</details>

---

## Exercise 2 – Browse mode ★

Make the key visible in `ls /mnt/auto` before it is mounted.

<details>
<summary>Solution</summary>

```bash
echo "/mnt/auto /etc/auto.nfs --timeout=60 --ghost" > /etc/auto.master.d/nfs.autofs
systemctl reload autofs; ls /mnt/auto
```

</details>

---

## Exercise 3 – Read-only and local entries ★★

Add two keys to the same map: `pub` → `rhel1:/exports/ro` read-only, and `img` → a local ext4 loop image (`-fstype=ext4,loop :/root/disk.img`).

<details>
<summary>Solution</summary>

```bash
truncate -s 100M /root/disk.img && mkfs.ext4 -q /root/disk.img
cat >> /etc/auto.nfs << 'EOF'
pub  -ro                rhel1:/exports/ro
img  -fstype=ext4,loop  :/root/disk.img
EOF
systemctl reload autofs; ls /mnt/auto/pub /mnt/auto/img
```

</details>

---

## Exercise 4 – Wildcard home directories ★★

On the server, create `/exports/home/{ana,luis}` and export `/exports/home`. On the client, configure `/home/remote/<user>` with a wildcard map.

**Verify:** `ls /home/remote/ana` and `ls /home/remote/luis` both mount.

<details>
<summary>Solution</summary>

```bash
# server
mkdir -p /exports/home/{ana,luis}
echo "/exports/home 192.168.56.0/24(rw,sync,no_subtree_check)" >> /etc/exports.d/share.exports; exportfs -ra
# client
echo "/home/remote /etc/auto.home --timeout=300" > /etc/auto.master.d/home.autofs
echo "* -rw,sync rhel1:/exports/home/&" > /etc/auto.home
systemctl reload autofs; ls /home/remote/ana; findmnt -t nfs4
```

</details>

---

## Exercise 5 – systemd automount via fstab ★★

Disable autofs. Configure `/mnt/share` as a systemd automount with a 60-second idle timeout in `/etc/fstab`.

**Verify:** `systemctl list-units --type=automount` shows `mnt-share.automount` waiting; `ls /mnt/share` mounts it.

<details>
<summary>Solution</summary>

```bash
systemctl disable --now autofs
mkdir -p /mnt/share
echo "rhel1:/exports/share /mnt/share nfs noauto,x-systemd.automount,x-systemd.idle-timeout=60,_netdev 0 0" >> /etc/fstab
systemctl daemon-reload && systemctl restart remote-fs.target
systemctl list-units --type=automount; ls /mnt/share; findmnt /mnt/share
```

</details>

---

## Exercise 6 – Units by hand ★★★

Replace the fstab line with hand-written `mnt-share.mount` and `mnt-share.automount` units. Enable the right one.

<details>
<summary>Solution</summary>

```bash
sed -i '\|/mnt/share|d' /etc/fstab
cat > /etc/systemd/system/mnt-share.mount << 'EOF'
[Unit]
Description=Lab share
[Mount]
What=rhel1:/exports/share
Where=/mnt/share
Type=nfs
Options=_netdev
EOF
cat > /etc/systemd/system/mnt-share.automount << 'EOF'
[Unit]
Description=Automount lab share
[Automount]
Where=/mnt/share
TimeoutIdleSec=60
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload && systemctl enable --now mnt-share.automount
```

</details>

---

## Challenge – Exam-style ★★★

> Export `/exports/pub` read-only to the lab subnet from `rhel1`. On `deb1`, automount it at `/net/pub` with autofs.

<details>
<summary>Solution</summary>

```bash
# rhel1
mkdir -p /exports/pub && echo hi > /exports/pub/readme
echo "/exports/pub 192.168.56.0/24(ro,sync,no_subtree_check)" >> /etc/exports.d/share.exports
exportfs -ra; firewall-cmd --permanent --add-service=nfs; firewall-cmd --reload
# deb1
echo "/net /etc/auto.net2 --timeout=60" > /etc/auto.master.d/net.autofs
echo "pub -ro rhel1:/exports/pub" > /etc/auto.net2
systemctl reload autofs; cat /net/pub/readme
```

(The map is named `auto.net2` because `/etc/auto.net` already exists as a program map on some systems.)

</details>

---

<!-- footer:start -->
[← Exercises: Network Block Devices and iSCSI](07-network-block-devices.md) · [Module exercises](README.md) · [Exercises: Monitoring Storage Performance →](09-storage-performance.md)
<!-- footer:end -->
