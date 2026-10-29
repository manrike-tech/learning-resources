# Exercises – NFS

<!-- nav:start -->
> **Module 05 · Storage** — lesson 6 of 9  
> Read first: [Theory – NFS: Network File System](../../theory/05-storage/06-nfs.md) · [Course lesson (commands)](../../course/05-Storage.md#lesson-6---nfs-gap) · [Module exercises](README.md)
<!-- nav:end -->

> Server: `rhel1` (192.168.56.20). Client: `deb1` (192.168.56.10). Adjust IPs/hostnames to your lab; add both to `/etc/hosts`.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Basic export ★

On the server, export `/exports/share` read-write to the lab subnet. Open the firewall for NFSv4.

**Verify:** `exportfs -v` on the server.

<details>
<summary>Solution</summary>

```bash
dnf install -y nfs-utils
mkdir -p /exports/share && chmod 777 /exports/share     # lab only – see Exercise 4 for proper ownership
echo "/exports/share 192.168.56.0/24(rw,sync,no_subtree_check)" > /etc/exports.d/share.exports
systemctl enable --now nfs-server
exportfs -arv
firewall-cmd --permanent --add-service=nfs && firewall-cmd --reload
```

</details>

---

## Exercise 2 – Mount on the client ★

Install the client tools, list the exports, mount the share on `/mnt/share`, create a file.

<details>
<summary>Solution</summary>

```bash
apt install -y nfs-common
showmount -e rhel1          # fails? open mountd and rpc-bind on the server (needed by showmount only)
mkdir -p /mnt/share && mount -t nfs rhel1:/exports/share /mnt/share
touch /mnt/share/from-client; findmnt /mnt/share
```

</details>

---

## Exercise 3 – showmount needs more ports ★★

Explain why `showmount -e` fails while `mount` works after Exercise 1. Fix it.

<details>
<summary>Solution</summary>

`showmount` uses the NFSv3 MOUNT protocol via rpcbind (111) and mountd; NFSv4 mounts only need 2049.

```bash
firewall-cmd --permanent --add-service={mountd,rpc-bind} && firewall-cmd --reload
```

</details>

---

## Exercise 4 – root_squash and UIDs ★★

1. As root on the client, create a file in the share. Who owns it on the server?
2. On the client create user `ana` with UID 2000; on the server create a **different** user `luis` with the same UID 2000. As `ana` on the client, create a file in the share. Who owns it on the server?

<details>
<summary>Solution</summary>

1. `nobody` (`nfsnobody` on old systems) – root is squashed.
2. On the server the file belongs to `luis`: NFS (`sec=sys`) transmits numeric UIDs. On the client `ls -l` may show `ana` or `nobody` depending on NFSv4 idmapping.

</details>

---

## Exercise 5 – Export options ★★

1. Export `/exports/ro` read-only to everyone and verify a write fails.
2. Export `/exports/drop` with `all_squash,anonuid=3000,anongid=3000`; create files as different client users and check owners on the server.

<details>
<summary>Solution</summary>

```bash
mkdir -p /exports/ro /exports/drop; chown 3000:3000 /exports/drop
cat >> /etc/exports.d/share.exports << 'EOF'
/exports/ro   *(ro,sync,no_subtree_check)
/exports/drop 192.168.56.0/24(rw,sync,all_squash,anonuid=3000,anongid=3000,no_subtree_check)
EOF
exportfs -arv
```

</details>

---

## Exercise 6 – Persistent mount ★

Make the client mount the share at boot with `_netdev,nofail`. Reboot the client to verify.

<details>
<summary>Solution</summary>

```bash
echo "rhel1:/exports/share /mnt/share nfs defaults,_netdev,nofail 0 0" >> /etc/fstab
systemctl daemon-reload && mount -a && findmnt /mnt/share
```

</details>

---

## Exercise 7 – The space trap ★★

Add `/exports/share 192.168.56.10 (rw)` (with a space) and run `exportfs -arv`. Read the warning and `exportfs -v`. Fix it.

<details>
<summary>Solution</summary>

`exportfs -v` shows 192.168.56.10 with default `ro` **and** `*` (everyone) with `rw`. Remove the space: `192.168.56.10(rw)`.

</details>

---

## Challenge – Server outage ★★★

1. With the share mounted (default `hard`), stop `nfs-server`. Run `ls /mnt/share` in the background and look at its process state.
2. Start the server – what happens to the stuck process?
3. Mount the share again with `soft,timeo=30,retrans=2` and repeat. Compare.

<details>
<summary>Solution</summary>

1. `ls` hangs in state `D` (`ps -o stat,cmd -C ls`) – `hard` retries forever.
2. It completes as soon as the server is back – no data lost.
3. With `soft`, `ls` returns `Input/output error` after the timeout. Safer for reads you can retry; risky for writes.

</details>

---

<!-- footer:start -->
[← Exercises: Swap Space](05-swap.md) · [Module exercises](README.md) · [Exercises: Network Block Devices and iSCSI →](07-network-block-devices.md)
<!-- footer:end -->
