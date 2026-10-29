# Module 07 - Networking

**LFS207:** Ch. 27 Network Addresses · Ch. 28 Network Devices and Configuration · Ch. 30 Firewalls
**Exam domain:** Networking (25%)  
**Theory:** [Module 07 theory notes](../theory/07-networking/README.md) · **Exercises:** [Module 07 exercises](../exercises/07-networking/README.md)  
**Gaps covered:** static routing, bridges & bonds, OpenSSH, time sync, reverse proxy & load balancing

---

## Learning Objectives

- Configure IPv4/IPv6 addressing and name resolution (runtime and persistent).
- Add static routes; build bridges and bonds.
- Filter packets, redirect ports and NAT.
- Configure SSH, NTP and an Nginx reverse proxy / load balancer.
- Troubleshoot connectivity layer by layer.

---

## Lesson 1 - Addresses (Ch. 27)

> **Theory:** [Network Addresses: IPv4 and IPv6](../theory/07-networking/01-ip-addressing.md) · **Exercises:** [practice tasks](../exercises/07-networking/01-ip-addressing.md)

- IPv4: 32 bits, CIDR (`/24` = 255.255.255.0). Private: `10/8`, `172.16/12`, `192.168/16`. Loopback `127.0.0.0/8`.
- IPv6: 128 bits, `/64` typical. Link-local `fe80::/10`, loopback `::1`, ULA `fd00::/8`.

```bash
ipcalc 192.168.56.10/26     # (package ipcalc) network, broadcast, range
```

---

## Lesson 2 - Runtime Configuration with `ip` (Ch. 28)

> **Theory:** [Runtime Network Configuration with ip](../theory/07-networking/02-runtime-ip.md) · **Exercises:** [practice tasks](../exercises/07-networking/02-runtime-ip.md)

Changes made with `ip` are **lost at reboot**.

```bash
ip -br addr; ip -br link
ip addr add 192.168.56.50/24 dev eth1
ip addr add 2001:db8::50/64 dev eth1
ip addr del 192.168.56.50/24 dev eth1
ip link set eth1 up|down
ip route; ip -6 route
ip route add 10.10.0.0/16 via 192.168.56.1
ip route add default via 192.168.56.1
ip route get 8.8.8.8             # which route will be used
ip neigh                         # ARP/NDP cache
```

---

## Lesson 3 - Persistent Configuration

> **Theory:** [Persistent Network Configuration: Netplan and NetworkManager](../theory/07-networking/03-persistent-config.md) · **Exercises:** [practice tasks](../exercises/07-networking/03-persistent-config.md)

### Ubuntu - Netplan

```yaml
# /etc/netplan/60-static.yaml   (chmod 600)
network:
  version: 2
  ethernets:
    eth1:
      dhcp4: false
      addresses:
        - 192.168.56.10/24
        - 2001:db8::10/64
      routes:
        - to: default
          via: 192.168.56.1
        - to: 10.10.0.0/16
          via: 192.168.56.254
      nameservers:
        addresses: [1.1.1.1, 9.9.9.9]
        search: [example.lab]
```
```bash
netplan try        # auto-rollback if you lose connection
netplan apply
```

### RHEL family - NetworkManager (`nmcli`)

```bash
nmcli device status; nmcli connection show
nmcli con mod eth1 ipv4.method manual ipv4.addresses 192.168.56.20/24 \
      ipv4.gateway 192.168.56.1 ipv4.dns "1.1.1.1 9.9.9.9" ipv4.dns-search example.lab
nmcli con mod eth1 +ipv4.routes "10.10.0.0/16 192.168.56.254"
nmcli con mod eth1 ipv6.method manual ipv6.addresses 2001:db8::20/64
nmcli con up eth1
nmcli con add type ethernet ifname eth2 con-name eth2 ipv4.method auto
nmtui        # text UI
```

---

## Lesson 4 - Hostname & Name Resolution

> **Theory:** [Hostname and Name Resolution](../theory/07-networking/04-hostname-resolution.md) · **Exercises:** [practice tasks](../exercises/07-networking/04-hostname-resolution.md)

```bash
hostnamectl set-hostname web01.example.lab
cat /etc/hosts               # 192.168.56.20  rhel1.example.lab rhel1
cat /etc/nsswitch.conf       # hosts: files dns  (order of lookup)
cat /etc/resolv.conf
resolvectl status            # systemd-resolved (Ubuntu)
getent hosts rhel1           # uses nsswitch (files + dns)
dig example.com; dig +short -x 1.1.1.1; host example.com; nslookup
```

> `dig` queries DNS directly and ignores `/etc/hosts`; `getent hosts` follows nsswitch.

---

## Lesson 5 - Static Routing (Gap)

> **Theory:** [Static Routing](../theory/07-networking/05-static-routing.md) · **Exercises:** [practice tasks](../exercises/07-networking/05-static-routing.md)

Make one VM a router between two networks:

```bash
sysctl -w net.ipv4.ip_forward=1                         # + persist in /etc/sysctl.d/
ip route add 10.20.0.0/24 via 192.168.56.20 dev eth1    # runtime
```
Persist with Netplan `routes:` or `nmcli con mod ... +ipv4.routes`.

```bash
ip route show table main; tracepath 10.20.0.5
```

---

## Lesson 6 - Bridges & Bonds (Gap)

> **Theory:** [Bridges and Bonds](../theory/07-networking/06-bridges-bonds.md) · **Exercises:** [practice tasks](../exercises/07-networking/06-bridges-bonds.md)

- **Bridge**: a virtual switch (VMs/containers share the physical NIC).
- **Bond**: aggregates NICs for redundancy/throughput. Modes: `active-backup` (1), `balance-rr` (0), `802.3ad`/LACP (4)…

### nmcli

```bash
# Bridge
nmcli con add type bridge ifname br0 con-name br0 ipv4.method manual ipv4.addresses 192.168.56.30/24
nmcli con add type bridge-slave ifname eth1 master br0      # (newer: type ethernet ... port-type bridge controller br0)
nmcli con up br0

# Bond
nmcli con add type bond ifname bond0 con-name bond0 bond.options "mode=active-backup,miimon=100" \
      ipv4.method manual ipv4.addresses 192.168.56.40/24
nmcli con add type ethernet ifname eth2 master bond0
nmcli con add type ethernet ifname eth3 master bond0
nmcli con up bond0
cat /proc/net/bonding/bond0
```

### Netplan

```yaml
network:
  version: 2
  ethernets:
    eth2: {}
    eth3: {}
  bonds:
    bond0:
      interfaces: [eth2, eth3]
      parameters:
        mode: active-backup
        mii-monitor-interval: 100
      addresses: [192.168.56.40/24]
  bridges:
    br0:
      interfaces: [eth1]
      addresses: [192.168.56.30/24]
```

### Runtime with `ip`

```bash
ip link add br0 type bridge; ip link set eth1 master br0; ip link set br0 up
ip link add bond0 type bond mode active-backup
bridge link show
```

> Add two extra NICs to a VM for these labs.

---

## Lesson 7 - Firewalls, Port Redirection & NAT (Ch. 30)

> **Theory:** [Packet Filtering, Port Redirection and NAT](../theory/07-networking/07-firewalls-nat.md) · **Exercises:** [practice tasks](../exercises/07-networking/07-firewalls-nat.md)

### firewalld (RHEL)

```bash
firewall-cmd --state; firewall-cmd --get-active-zones
firewall-cmd --list-all
firewall-cmd --add-service=http --permanent
firewall-cmd --add-port=8080/tcp --permanent
firewall-cmd --add-source=10.0.0.0/8 --zone=trusted --permanent
firewall-cmd --reload
# Port redirection: 80 → 8080 locally
firewall-cmd --add-forward-port=port=80:proto=tcp:toport=8080 --permanent
# Forward to another host (needs masquerade)
firewall-cmd --add-forward-port=port=2222:proto=tcp:toport=22:toaddr=192.168.56.20 --permanent
firewall-cmd --add-masquerade --permanent       # NAT / SNAT
firewall-cmd --add-rich-rule='rule family=ipv4 source address=192.168.56.0/24 service name=ssh accept' --permanent
```

> Without `--permanent` the change is runtime only; with it, it applies only after `--reload`. `--runtime-to-permanent` saves current state.

### ufw (Ubuntu)

```bash
ufw enable; ufw status verbose
ufw allow 22/tcp; ufw allow from 192.168.56.0/24 to any port 3306
ufw deny 23; ufw delete allow 80/tcp
```
NAT/redirect with ufw: rules in `/etc/ufw/before.rules` (`*nat` table) + `DEFAULT_FORWARD_POLICY="ACCEPT"` in `/etc/default/ufw`.

### nftables (distro-agnostic)

```bash
nft list ruleset
nft add table inet filter
nft add chain inet filter input '{ type filter hook input priority 0; policy accept; }'
nft add rule inet filter input tcp dport 22 accept
nft add rule inet filter input ct state established,related accept

# NAT
nft add table ip nat
nft add chain ip nat prerouting '{ type nat hook prerouting priority -100; }'
nft add chain ip nat postrouting '{ type nat hook postrouting priority 100; }'
nft add rule ip nat prerouting tcp dport 8080 redirect to :80                # local port redirect
nft add rule ip nat prerouting tcp dport 2222 dnat to 192.168.56.20:22       # DNAT
nft add rule ip nat postrouting oifname "eth0" masquerade                    # SNAT/masquerade
nft list ruleset > /etc/nftables.conf     # persist (path varies: /etc/sysconfig/nftables.conf on RHEL)
systemctl enable nftables
```

### iptables (legacy, still asked about)

```bash
iptables -L -n -v; iptables -t nat -L -n
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

> Use only **one** firewall front-end at a time.

---

## Lesson 8 - OpenSSH Server & Client (Gap)

> **Theory:** [The OpenSSH Server and Client](../theory/07-networking/08-openssh.md) · **Exercises:** [practice tasks](../exercises/07-networking/08-openssh.md)

**Server** `/etc/ssh/sshd_config` (or drop-in `/etc/ssh/sshd_config.d/*.conf`, read first-match-wins):

```
Port 22
PermitRootLogin no               # or prohibit-password
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers ana luis
AllowGroups sshusers
ClientAliveInterval 300
X11Forwarding no
```
```bash
sshd -t                     # syntax test - ALWAYS before restart
systemctl restart sshd      # Ubuntu: ssh
# Non-standard port on RHEL: semanage port -a -t ssh_port_t -p tcp 2222 + firewall
```

**Client:**

```bash
ssh-keygen -t ed25519
ssh-copy-id ana@rhel1
ssh -p 2222 -i ~/.ssh/id_ed25519 ana@rhel1
scp file ana@rhel1:/tmp/; rsync -avz dir/ ana@rhel1:/backup/
ssh -L 8080:localhost:80 ana@rhel1     # local port forward
ssh -J bastion ana@internal            # jump host
```

`~/.ssh/config`:

```
Host web
    HostName 192.168.56.20
    User ana
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
```

Permissions: `~/.ssh` 700, `authorized_keys` 600, private key 600.

---

## Lesson 9 - Time Synchronization (Gap)

> **Theory:** [Setting and Synchronizing System Time](../theory/07-networking/09-time-sync.md) · **Exercises:** [practice tasks](../exercises/07-networking/09-time-sync.md)

```bash
timedatectl
timedatectl set-timezone Europe/Madrid
timedatectl set-ntp true
timedatectl set-time "2026-10-20 10:00:00"     # only with NTP off
```

### chrony

```
# /etc/chrony.conf  (Debian: /etc/chrony/chrony.conf)
server 0.pool.ntp.org iburst
pool time.example.lab iburst
allow 192.168.56.0/24            # act as NTP server for this subnet
```
```bash
systemctl restart chronyd        # Ubuntu: chrony
chronyc sources -v; chronyc tracking; chronyc makestep
```

Ubuntu may use `systemd-timesyncd` instead: `/etc/systemd/timesyncd.conf` → `NTP=`.

---

## Lesson 10 - Reverse Proxy & Load Balancer (Gap)

> **Theory:** [Reverse Proxies and Load Balancers](../theory/07-networking/10-reverse-proxy.md) · **Exercises:** [practice tasks](../exercises/07-networking/10-reverse-proxy.md)

Nginx in front of backends.

```nginx
# /etc/nginx/conf.d/app.conf  (Ubuntu: also sites-available/ + symlink in sites-enabled/)
upstream app_backend {
    # round-robin by default; also: least_conn; ip_hash;
    server 192.168.56.20:8080 weight=2;
    server 192.168.56.21:8080;
    server 192.168.56.22:8080 backup;
}

server {
    listen 80;
    server_name app.example.lab;

    location / {
        proxy_pass http://app_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
```bash
nginx -t && systemctl reload nginx
curl -H "Host: app.example.lab" http://localhost/
# RHEL + SELinux: allow nginx to connect to backends
setsebool -P httpd_can_network_connect 1
```

Simple single-backend reverse proxy: `proxy_pass http://127.0.0.1:3000;` inside `location /`.
Know that **HAProxy** and Apache `mod_proxy_balancer` do the same job.

---

## Lesson 11 - Monitoring & Troubleshooting

> **Theory:** [Monitoring and Troubleshooting the Network](../theory/07-networking/11-troubleshooting.md) · **Exercises:** [practice tasks](../exercises/07-networking/11-troubleshooting.md)

Bottom-up method:

| Layer | Check |
|-------|-------|
| Link | `ip -br link`, `ethtool eth0`, cable/NIC state |
| IP | `ip -br addr`, correct subnet? |
| Routing | `ip route`, `ip route get`, `ping gateway`, `tracepath` / `traceroute` / `mtr` |
| DNS | `getent hosts name`, `dig`, `/etc/resolv.conf` |
| Port/service | `ss -tulpn`, `nc -zv host 80`, `curl -v` |
| Firewall | `firewall-cmd --list-all`, `nft list ruleset`, `ufw status` |
| Traffic | `tcpdump -i eth1 -nn port 80`, `ip -s link` (errors/drops) |

---

## Lab 07

1. Give `deb1` a static IPv4 + IPv6 address persistently (Netplan); do the same on `rhel1` with `nmcli`.
2. Make `rhel1` resolve `deb1.example.lab` via `/etc/hosts`.
3. Add a persistent static route to `10.99.0.0/24` via `deb1`; enable forwarding on `deb1`.
4. Create `bond0` (active-backup) from two NICs; unplug one and verify failover in `/proc/net/bonding/bond0`.
5. Create bridge `br0` with an IP.
6. With firewalld: allow HTTP, redirect port 8080 → 80; with nftables on `deb1`: masquerade traffic out of `eth0`.
7. SSH: key-only login, root login disabled, port 2222 (handle SELinux + firewall on RHEL).
8. Make `deb1` an NTP server for the lab subnet and `rhel1` its client.
9. Run two simple backends (`python3 -m http.server 8081` / `8082` in different dirs) and load-balance them with Nginx on port 80.

## Self-check

- [ ] I can configure networking persistently with Netplan **and** nmcli.
- [ ] I can write DNAT, port redirect and masquerade rules in firewalld and nftables.
- [ ] I always run `sshd -t` / `nginx -t` before restarting.
- [ ] I can troubleshoot a "can't reach the service" ticket layer by layer.
