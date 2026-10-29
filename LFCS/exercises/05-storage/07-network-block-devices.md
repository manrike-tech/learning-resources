# Exercises – Network Block Devices

<!-- nav:start -->
> **Module 05 · Storage** — lesson 7 of 9  
> Read first: [Theory – Network Block Devices and iSCSI](../../theory/05-storage/07-network-block-devices.md) · [Course lesson (commands)](../../course/05-Storage.md#lesson-7---network-block-devices-gap) · [Module exercises](README.md)
<!-- nav:end -->

> Server: `deb1` (Ubuntu, `nbd-server`). Client: `rhel1` (needs EPEL for the `nbd` package) or a second Ubuntu VM.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Export an image file ★

On the server, create a 1 GiB sparse image `/srv/nbd/disk1.img` and export it as `disk1`. Allow listing of exports.

**Verify:** `ss -tlnp | grep 10809`.

<details>
<summary>Solution</summary>

```bash
apt install -y nbd-server
mkdir -p /srv/nbd && truncate -s 1G /srv/nbd/disk1.img && chown nbd:nbd /srv/nbd/disk1.img
cat > /etc/nbd-server/config << 'EOF'
[generic]
    user = nbd
    group = nbd
    allowlist = true
[disk1]
    exportname = /srv/nbd/disk1.img
EOF
systemctl restart nbd-server
ufw allow 10809/tcp 2>/dev/null
```

</details>

---

## Exercise 2 – Attach and use ★

On the client: load the module, list exports, attach `disk1` as `/dev/nbd0`, create ext4, mount at `/mnt/nbd`, write a file, unmount and detach.

<details>
<summary>Solution</summary>

```bash
dnf install -y epel-release && dnf install -y nbd     # or: apt install nbd-client
modprobe nbd
nbd-client -l deb1
nbd-client deb1 /dev/nbd0 -N disk1
mkfs.ext4 /dev/nbd0 && mkdir -p /mnt/nbd && mount /dev/nbd0 /mnt/nbd
echo hello > /mnt/nbd/test.txt
umount /mnt/nbd && nbd-client -d /dev/nbd0
```

</details>

---

## Exercise 3 – Data persists on the server ★

Re-attach `disk1` and verify `test.txt` is still there. On the server, check the real space used by the sparse image (`du -h` vs `ls -lh`).

<details>
<summary>Solution</summary>

```bash
nbd-client deb1 /dev/nbd0 -N disk1 && mount /dev/nbd0 /mnt/nbd && cat /mnt/nbd/test.txt
# server:
ls -lh /srv/nbd/disk1.img; du -h /srv/nbd/disk1.img
```

</details>

---

## Exercise 4 – Export an LV, restrict clients ★★

Export a 512 MiB LV as `disk2`, only to the client's IP via an `authfile`. Test from the allowed client and from another host (or temporarily change the allowed IP).

<details>
<summary>Solution</summary>

```bash
lvcreate -n lvnbd -L 512M vgapp
cat >> /etc/nbd-server/config << 'EOF'
[disk2]
    exportname = /dev/vgapp/lvnbd
    authfile = /etc/nbd-server/allow-disk2
EOF
echo "192.168.56.20" > /etc/nbd-server/allow-disk2
setfacl -m u:nbd:rw /dev/vgapp/lvnbd       # lab shortcut (not persistent across reboots)
systemctl restart nbd-server
```

</details>

---

## Exercise 5 – Partitions on NBD ★★

Reload the client module with `max_part=8`, attach `disk2`, create two partitions on `/dev/nbd0`, and show `/dev/nbd0p1` and `/dev/nbd0p2`. Make the module option persistent.

<details>
<summary>Solution</summary>

```bash
nbd-client -d /dev/nbd0 2>/dev/null; modprobe -r nbd; modprobe nbd max_part=8
nbd-client deb1 /dev/nbd0 -N disk2
parted /dev/nbd0 --script mklabel gpt mkpart a 1MiB 50% mkpart b 50% 100%
lsblk /dev/nbd0
echo "options nbd max_part=8" > /etc/modprobe.d/nbd.conf
echo nbd > /etc/modules-load.d/nbd.conf
```

</details>

---

## Challenge – Persistent NBD mount ★★★

On a Debian/Ubuntu client, make `disk1` attach automatically at boot as `/dev/nbd0` and mount at `/mnt/nbd`, without blocking boot if the server is down. Reboot to verify; then reboot with the server stopped.

<details>
<summary>Solution</summary>

```bash
apt install -y nbd-client
echo nbd > /etc/modules-load.d/nbd.conf
echo "nbd0 deb1 disk1 persist" >> /etc/nbdtab
echo "/dev/nbd0 /mnt/nbd ext4 defaults,_netdev,nofail,x-systemd.device-timeout=10s 0 0" >> /etc/fstab
systemctl enable nbd-client 2>/dev/null; systemctl daemon-reload
```

</details>

---

## Optional – iSCSI ★★★

Using `targetcli` on the server (`dnf install targetcli` / `apt install targetcli-fb`), create a block backstore from an LV, a target `iqn.2026-10.lab.example:storage`, a LUN and an ACL for the client's initiator name; on the client, discover and log in with `iscsiadm`, then format and mount the new disk with `_netdev` in fstab.

<details>
<summary>Solution (outline)</summary>

```bash
# server
targetcli /backstores/block create name=disk1 dev=/dev/vgapp/lviscsi
targetcli /iscsi create iqn.2026-10.lab.example:storage
targetcli /iscsi/iqn.2026-10.lab.example:storage/tpg1/luns create /backstores/block/disk1
targetcli /iscsi/iqn.2026-10.lab.example:storage/tpg1/acls create $(ssh client cat /etc/iscsi/initiatorname.iscsi | cut -d= -f2)
targetcli saveconfig; firewall-cmd --permanent --add-service=iscsi-target; firewall-cmd --reload
# client
iscsiadm -m discovery -t st -p SERVER_IP
iscsiadm -m node -T iqn.2026-10.lab.example:storage -p SERVER_IP --login
lsblk
```

</details>

---

<!-- footer:start -->
[← Exercises: NFS: Network File System](06-nfs.md) · [Module exercises](README.md) · [Exercises: Filesystem Automounters →](08-automounters.md)
<!-- footer:end -->
