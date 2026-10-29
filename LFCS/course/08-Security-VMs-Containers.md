# Module 08 - Security, Virtualization & Containers

**LFS207:** Ch. 18 Containers Overview · Ch. 33 Linux Security Modules
**Exam domain:** Operations & Deployment (25%) · Essential Commands (20%)  
**Theory:** [Module 08 theory notes](../theory/08-security-vms-containers/README.md) · **Exercises:** [Module 08 exercises](../exercises/08-security-vms-containers/README.md)  
**Gaps covered:** SSL certificates, libvirt VMs, hands-on container management

---

## Learning Objectives

- Explain DAC vs MAC; manage SELinux modes, contexts, booleans and ports.
- Read AppArmor status and profiles.
- Generate, sign and inspect TLS certificates.
- Create and manage VMs with libvirt.
- Run and manage containers with Docker or Podman.

---

## Lesson 1 - Linux Security Modules (Ch. 33)

> **Theory:** [Linux Security Modules: DAC and MAC](../theory/08-security-vms-containers/01-lsm-dac-mac.md) · **Exercises:** [practice tasks](../exercises/08-security-vms-containers/01-lsm-dac-mac.md)

- **DAC** (Discretionary): owner decides (`chmod`, ACLs).
- **MAC** (Mandatory): system policy decides, even for root. Implemented by **LSMs**: SELinux (RHEL/Fedora), AppArmor (Ubuntu/SUSE), plus smaller ones (Yama, Landlock…).
- MAC is checked **after** DAC: both must allow the action.

---

## Lesson 2 - SELinux

> **Theory:** [SELinux](../theory/08-security-vms-containers/02-selinux.md) · **Exercises:** [practice tasks](../exercises/08-security-vms-containers/02-selinux.md)

### Modes

```bash
getenforce; sestatus
setenforce 0|1                 # permissive / enforcing (runtime only)
# Persistent: /etc/selinux/config → SELINUX=enforcing|permissive|disabled
# Switching from disabled → enforcing requires a relabel: touch /.autorelabel && reboot
```

### Contexts: `user:role:type:level` - the **type** matters most

```bash
ls -Z /var/www/html; ps -eZ | grep httpd; id -Z
```

### Fixing file contexts

```bash
# Temporary (lost on relabel)
chcon -t httpd_sys_content_t /srv/web/index.html
# Persistent - the right way
semanage fcontext -a -t httpd_sys_content_t "/srv/web(/.*)?"
restorecon -Rv /srv/web
semanage fcontext -l | grep /srv/web
```

> **Classic trap:** `mv` keeps the original context; `cp` gets the destination's. A file moved from `/root` into `/var/www/html` → 403 Forbidden. Fix with `restorecon`.

### Booleans

```bash
getsebool -a | grep httpd
setsebool -P httpd_can_network_connect on     # -P = persistent
semanage boolean -l | grep nfs
```

### Ports

```bash
semanage port -l | grep http
semanage port -a -t http_port_t -p tcp 8888
```

### Troubleshooting denials

```bash
ausearch -m AVC -ts recent
journalctl -t setroubleshoot           # if setroubleshoot-server installed
sealert -a /var/log/audit/audit.log
audit2why < /var/log/audit/audit.log
# Last resort - custom policy module:
ausearch -m AVC -c myapp --raw | audit2allow -M mymod && semodule -i mymod.pp
```

`semanage` comes from `policycoreutils-python-utils`.

---

## Lesson 3 - AppArmor (Ubuntu)

> **Theory:** [AppArmor](../theory/08-security-vms-containers/03-apparmor.md) · **Exercises:** [practice tasks](../exercises/08-security-vms-containers/03-apparmor.md)

```bash
aa-status
ls /etc/apparmor.d/
aa-complain /etc/apparmor.d/usr.sbin.nginx   # log only   (apparmor-utils)
aa-enforce  /etc/apparmor.d/usr.sbin.nginx
apparmor_parser -r /etc/apparmor.d/profile    # reload
journalctl -k | grep -i apparmor              # denials
```

Path-based (profiles per program), versus SELinux label-based.

---

## Lesson 4 - SSL/TLS Certificates (Gap)

> **Theory:** [Working with SSL/TLS Certificates](../theory/08-security-vms-containers/04-ssl-certificates.md) · **Exercises:** [practice tasks](../exercises/08-security-vms-containers/04-ssl-certificates.md)

```bash
# Private key
openssl genrsa -out server.key 2048
openssl genpkey -algorithm ed25519 -out server.key

# CSR (certificate signing request)
openssl req -new -key server.key -out server.csr -subj "/CN=web.example.lab/O=Lab"

# Self-signed certificate (key + cert in one go)
openssl req -x509 -newkey rsa:2048 -nodes -keyout server.key -out server.crt \
        -days 365 -subj "/CN=web.example.lab" -addext "subjectAltName=DNS:web.example.lab"

# Your own CA signs the CSR
openssl req -x509 -newkey rsa:4096 -nodes -keyout ca.key -out ca.crt -days 3650 -subj "/CN=Lab CA"
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365

# Inspect
openssl x509 -in server.crt -noout -text
openssl x509 -in server.crt -noout -subject -issuer -dates
openssl req -in server.csr -noout -text
openssl verify -CAfile ca.crt server.crt
openssl s_client -connect web.example.lab:443 -servername web.example.lab </dev/null

# Does key match cert?
openssl x509 -noout -modulus -in server.crt | openssl md5
openssl rsa  -noout -modulus -in server.key | openssl md5

# Conversions
openssl x509 -in cert.pem -outform der -out cert.der
openssl pkcs12 -export -inkey server.key -in server.crt -out server.p12
```

Trust a CA system-wide:

```bash
# Debian/Ubuntu
cp ca.crt /usr/local/share/ca-certificates/lab-ca.crt && update-ca-certificates
# RHEL family
cp ca.crt /etc/pki/ca-trust/source/anchors/ && update-ca-trust
```

Use in Nginx:

```nginx
server {
    listen 443 ssl;
    server_name web.example.lab;
    ssl_certificate     /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;
}
```

Keys: mode `600`, owned by root.

---

## Lesson 5 - Virtual Machines with libvirt (Gap)

> **Theory:** [Virtual Machines with libvirt](../theory/08-security-vms-containers/05-libvirt-vms.md) · **Exercises:** [practice tasks](../exercises/08-security-vms-containers/05-libvirt-vms.md)

```bash
# Check hardware support (nested virtualization needed inside a VM)
grep -cE 'vmx|svm' /proc/cpuinfo
# Install
apt install qemu-kvm libvirt-daemon-system libvirt-clients virtinst
dnf install qemu-kvm libvirt virt-install
systemctl enable --now libvirtd
virt-host-validate
```

```bash
virsh list --all
virsh start|shutdown|reboot|destroy vm1        # destroy = hard power off
virsh autostart vm1; virsh autostart --disable vm1
virsh dominfo vm1; virsh domblklist vm1; virsh domiflist vm1
virsh console vm1                               # Ctrl+] to exit
virsh edit vm1                                  # edit XML definition
virsh dumpxml vm1 > vm1.xml; virsh define vm1.xml; virsh undefine vm1 --remove-all-storage
virsh setvcpus vm1 2 --config; virsh setmaxmem vm1 2G --config; virsh setmem vm1 2G --config
virsh snapshot-create-as vm1 snap1; virsh snapshot-list vm1; virsh snapshot-revert vm1 snap1
virsh net-list --all; virsh pool-list --all; virsh vol-list default

# Create a VM from a cloud image (fast)
virt-install --name vm1 --memory 1024 --vcpus 1 \
  --disk /var/lib/libvirt/images/vm1.qcow2,size=10 \
  --import --os-variant generic --network network=default --graphics none --noautoconsole
# Or from an ISO
virt-install --name vm2 --memory 2048 --vcpus 2 --disk size=10 \
  --cdrom /var/lib/libvirt/images/ubuntu.iso --os-variant ubuntu24.04

qemu-img create -f qcow2 disk.qcow2 10G; qemu-img info disk.qcow2
osinfo-query os | grep -i ubuntu
```

---

## Lesson 6 - Containers (Ch. 18 + Gap)

> **Theory:** [Containers: Docker and Podman](../theory/08-security-vms-containers/06-containers.md) · **Exercises:** [practice tasks](../exercises/08-security-vms-containers/06-containers.md)

Concepts: containers share the host kernel; isolation via **namespaces** (pid, net, mnt, uts, ipc, user) and resource control via **cgroups**. Images are layered; containers are running instances.

```bash
# Install an engine
apt install docker.io     |  dnf install podman       # podman is daemonless, CLI-compatible
systemctl enable --now docker
usermod -aG docker ana    # run docker without sudo

docker pull nginx:alpine
docker images; docker rmi nginx:alpine
docker run -d --name web -p 8080:80 nginx:alpine
docker run -d --name web2 -p 8081:80 -v /srv/html:/usr/share/nginx/html:ro \
           --restart unless-stopped -e TZ=Europe/Madrid nginx:alpine
docker run -it --rm alpine sh
docker ps; docker ps -a
docker logs -f web; docker inspect web; docker stats; docker top web
docker exec -it web sh
docker stop web; docker start web; docker restart web; docker rm -f web
docker cp file web:/tmp/
docker volume create data; docker volume ls
docker network create appnet; docker run -d --network appnet --name db redis
docker system df; docker system prune
```

Build an image:

```dockerfile
# Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
```
```bash
docker build -t mysite:1.0 .
docker tag mysite:1.0 registry.example.lab/mysite:1.0
docker push registry.example.lab/mysite:1.0
```

Engine configuration: Docker `/etc/docker/daemon.json` (e.g. `"log-driver"`, `"insecure-registries"`, `"data-root"`), then `systemctl restart docker`. Podman: `/etc/containers/registries.conf`, `/etc/containers/containers.conf`.

Podman + systemd (run a container as a service): use a **Quadlet** file in `/etc/containers/systemd/web.container`, or the older `podman generate systemd --new --name web`.

> On SELinux hosts, add `:Z` (private) or `:z` (shared) to bind mounts: `-v /srv/html:/usr/share/nginx/html:Z`.

---

## Lab 08

1. On `rhel1`, serve `/srv/web` with Apache/Nginx in enforcing mode: fix contexts persistently and make it listen on port 8888 (SELinux port + firewall).
2. Put AppArmor's nginx profile (if present) in complain mode and back.
3. Create a lab CA, sign a certificate for `web.example.lab` with a SAN, trust the CA system-wide, enable HTTPS in Nginx and verify with `curl https://web.example.lab` (no `-k`).
4. Create a VM with `virt-install`, set it to autostart, take a snapshot, give it 2 vCPUs persistently.
5. Run `nginx:alpine` named `web` on port 8080 with a bind-mounted page, restart policy `always`; check logs; build and run your own image from a Dockerfile.

## Self-check

- [ ] I use `semanage fcontext` + `restorecon`, not `chcon`, for persistent fixes.
- [ ] I can find and interpret an AVC denial.
- [ ] I can create a key, CSR, CA-signed cert and inspect expiry/SAN.
- [ ] I can manage VMs with `virsh` without a GUI.
- [ ] I can run, inspect, exec into and clean up containers.
