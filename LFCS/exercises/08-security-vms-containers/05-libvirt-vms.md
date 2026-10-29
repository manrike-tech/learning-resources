# Exercises – Virtual Machines with libvirt

<!-- nav:start -->
> **Module 08 · Security, Virtualization & Containers** — lesson 5 of 6  
> Read first: [Theory – Virtual Machines with libvirt](../../theory/08-security-vms-containers/05-libvirt-vms.md) · [Course lesson (commands)](../../course/08-Security-VMs-Containers.md#lesson-5---virtual-machines-with-libvirt-gap) · [Module exercises](README.md)
<!-- nav:end -->

> You need a host with KVM: a lab VM with **nested virtualization** enabled, or your physical Linux workstation. Small cloud images keep things fast (Ubuntu Noble cloud image ~600 MB, or CirrOS ~20 MB for quick tests).

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Prepare the host ★

Check CPU support, install the libvirt stack, validate the host, list networks and pools.

<details>
<summary>Solution</summary>

```bash
grep -cE 'vmx|svm' /proc/cpuinfo
dnf install -y qemu-kvm libvirt virt-install guestfs-tools || apt install -y qemu-kvm libvirt-daemon-system virtinst libguestfs-tools
systemctl enable --now libvirtd
virt-host-validate
virsh net-list --all; virsh pool-list --all
```

</details>

---

## Exercise 2 – Create a VM from a cloud image ★★

Create `vm1` (1 vCPU, 1024 MiB, 10 GiB overlay disk on a cloud image) on the `default` network with no graphics. Find its IP and log in via `virsh console`.

<details>
<summary>Solution</summary>

```bash
cd /var/lib/libvirt/images
curl -LO https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
qemu-img create -f qcow2 -F qcow2 -b noble-server-cloudimg-amd64.img vm1.qcow2 10G
virt-install --name vm1 --memory 1024 --vcpus 1 --disk vm1.qcow2 --import \
  --osinfo ubuntu24.04 --network network=default --graphics none --noautoconsole \
  --cloud-init root-password-generate=on
virsh net-dhcp-leases default
virsh console vm1          # Ctrl+] to exit
```

(If your virt-install lacks `--cloud-init`, set a password offline: `virt-customize -a vm1.qcow2 --root-password password:lab`.)

</details>

---

## Exercise 3 – Lifecycle ★

Shut `vm1` down cleanly, start it, force it off, start it again, and make it autostart. Show `dominfo`.

<details>
<summary>Solution</summary>

```bash
virsh shutdown vm1; sleep 20; virsh list --all
virsh start vm1; virsh destroy vm1; virsh start vm1
virsh autostart vm1; virsh dominfo vm1 | grep -E 'State|Autostart'
```

</details>

---

## Exercise 4 – Resources (exam-style) ★★

> Give `vm1` 2 vCPUs and 2 GiB of RAM persistently.

**Verify:** after `virsh shutdown` and `virsh start`, `virsh dominfo vm1` shows 2 CPUs and 2097152 KiB.

<details>
<summary>Solution</summary>

```bash
virsh setvcpus vm1 2 --maximum --config
virsh setvcpus vm1 2 --config
virsh setmaxmem vm1 2G --config
virsh setmem vm1 2G --config
virsh shutdown vm1; sleep 20; virsh start vm1; virsh dominfo vm1
```

</details>

---

## Exercise 5 – Extra disk ★★

Create a 2 GiB qcow2 volume in the default pool, attach it persistently to `vm1` as `vdb`, and find it inside the guest.

<details>
<summary>Solution</summary>

```bash
virsh vol-create-as default vm1-data.qcow2 2G --format qcow2
virsh attach-disk vm1 /var/lib/libvirt/images/vm1-data.qcow2 vdb --driver qemu --subdriver qcow2 --persistent
virsh domblklist vm1
# inside the guest: lsblk
```

</details>

---

## Exercise 6 – XML ★★

Dump `vm1`'s XML to `/root/vm1.xml`. With `virsh edit`, change its description (`<description>Lab VM</description>`). Show that `virsh dumpxml` reflects it.

---

## Exercise 7 – Snapshots ★★

Take a snapshot `clean`, create a file in the guest, revert to `clean`, and confirm the file is gone. Delete the snapshot.

<details>
<summary>Solution</summary>

```bash
virsh snapshot-create-as vm1 clean
# in guest: touch /root/oops
virsh snapshot-revert vm1 clean
virsh snapshot-list vm1; virsh snapshot-delete vm1 clean
```

(Internal snapshots need qcow2 disks without external backing complications; if your overlay setup refuses, use `--disk-only` or shut the VM down first.)

</details>

---

## Exercise 8 – A new storage pool ★★

Create a directory pool `vmstore` at `/srv/vmstore` that autostarts; create a volume in it. On RHEL, fix the SELinux label.

<details>
<summary>Solution</summary>

```bash
virsh pool-define-as vmstore dir --target /srv/vmstore
virsh pool-build vmstore; virsh pool-start vmstore; virsh pool-autostart vmstore
virsh vol-create-as vmstore test.qcow2 1G --format qcow2
semanage fcontext -a -t virt_image_t "/srv/vmstore(/.*)?"; restorecon -Rv /srv/vmstore
```

</details>

---

## Challenge – Clone and clean up ★★★

1. Shut `vm1` down and clone it to `vm2` with `virt-clone`; reset its machine identity with `virt-sysprep`; start both.
2. Remove `vm2` **and its storage** completely.

<details>
<summary>Solution</summary>

```bash
virsh shutdown vm1
virt-clone --original vm1 --name vm2 --auto-clone
virt-sysprep -d vm2 --operations machine-id,ssh-hostkeys,net-hwaddr
virsh start vm1; virsh start vm2; virsh list
virsh destroy vm2; virsh undefine vm2 --remove-all-storage
```

</details>

---

<!-- footer:start -->
[← Exercises: Working with SSL/TLS Certificates](04-ssl-certificates.md) · [Module exercises](README.md) · [Exercises: Containers: Docker and Podman →](06-containers.md)
<!-- footer:end -->
