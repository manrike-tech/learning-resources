# Network Addresses: IPv4 and IPv6

<!-- nav:start -->
> **Module 07 · Networking** — lesson 1 of 11  
> [Module index](README.md) · [Course lesson (commands)](../../course/07-Networking.md#lesson-1---addresses-ch-27) · [Exercises for this lesson](../../exercises/07-networking/01-ip-addressing.md)
<!-- nav:end -->

Every networking task – assigning addresses, adding routes, writing firewall rules – assumes you can read an address like `192.168.56.77/26` and immediately know its network, its broadcast address and which hosts are on the same link. In this lesson you will learn the layers of a network, MAC addresses, IPv4 addressing and subnetting, special address ranges, and the essentials of IPv6.

## What you will learn

- The TCP/IP layers and the address used at each one
- MAC addresses and how ARP/NDP link them to IP addresses
- IPv4 structure, CIDR prefixes and netmasks
- Calculating network, broadcast and host ranges by hand and with `ipcalc`
- Private, loopback, link-local and other special ranges
- IPv6: notation, prefixes, address types, SLAAC, NDP
- Ports as the transport-layer address

---

## Key terms

| Term | Definition |
|------|-----------|
| **MAC address** | 48-bit hardware address of a network interface (`52:54:00:3a:1f:9c`), used within one link (layer 2). |
| **IP address** | Logical address of an interface used for routing across networks (layer 3). |
| **Prefix length (CIDR)** | Number of leading bits that identify the network: `/24`. |
| **Netmask** | The prefix written as an address: `/24` = `255.255.255.0`. |
| **Network address** | The first address of a subnet (all host bits 0) – identifies the subnet itself. |
| **Broadcast address** | The last IPv4 address of a subnet (all host bits 1) – reaches every host on it. |
| **Gateway (default route)** | The router a host sends packets to when the destination is not on its own subnet. |
| **ARP / NDP** | Protocols that find the MAC address for an IPv4 (ARP) or IPv6 (NDP) address on the local link. |
| **Port** | 16-bit number identifying a service on a host (TCP/UDP, layer 4). |

---

## Layers and addresses

| TCP/IP layer | Unit | Address | Example | Linux view |
|--------------|------|---------|---------|-----------|
| Link | frame | MAC | `52:54:00:3a:1f:9c` | `ip link`, `ip neigh` |
| Internet | packet | IP | `192.168.56.10`, `2001:db8::10` | `ip addr`, `ip route` |
| Transport | segment | port | TCP 22, UDP 53 | `ss -tulpn` |
| Application | message | names, URLs | `https://web01.example.lab/` | `dig`, `curl` |

A connection is identified by the 5-tuple: protocol + source IP + source port + destination IP + destination port.

---

## MAC addresses and ARP

```console
$ ip link show enp1s0
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:3a:1f:9c brd ff:ff:ff:ff:ff:ff
```

- The first 3 bytes are the vendor (OUI): `52:54:00` = QEMU/KVM.
- `ff:ff:ff:ff:ff:ff` is the Ethernet broadcast.
- Frames are delivered by MAC **within one link**. To reach `192.168.56.20` on the same subnet, the host asks "who has 192.168.56.20?" (ARP) and caches the answer:

```console
$ ip neigh
192.168.56.1  dev enp1s0 lladdr 52:54:00:12:35:02 REACHABLE
192.168.56.20 dev enp1s0 lladdr 52:54:00:8c:44:10 STALE
```

---

## IPv4 addresses

An IPv4 address is **32 bits**, written as four decimal octets (0–255):

```text
192      .168      .56       .77
11000000  10101000  00111000  01001101
```

The **prefix length** splits it into network and host parts:

```text
192.168.56.77/26
11000000.10101000.00111000.01 | 001101
└──────── network (26 bits) ───┘ └ host (6 bits)
```

### Prefix and netmask table

| CIDR | Netmask | Addresses | Usable hosts | Block size (in the "interesting" octet) |
|------|---------|-----------|--------------|------------------------------------------|
| /8 | 255.0.0.0 | 16,777,216 | 16,777,214 | – |
| /16 | 255.255.0.0 | 65,536 | 65,534 | – |
| /20 | 255.255.240.0 | 4,096 | 4,094 | 16 (3rd octet) |
| /22 | 255.255.252.0 | 1,024 | 1,022 | 4 (3rd octet) |
| /24 | 255.255.255.0 | 256 | 254 | – |
| /25 | 255.255.255.128 | 128 | 126 | 128 |
| /26 | 255.255.255.192 | 64 | 62 | 64 |
| /27 | 255.255.255.224 | 32 | 30 | 32 |
| /28 | 255.255.255.240 | 16 | 14 | 16 |
| /29 | 255.255.255.248 | 8 | 6 | 8 |
| /30 | 255.255.255.252 | 4 | 2 | 4 |
| /31 | 255.255.255.254 | 2 | 2 (point-to-point, RFC 3021) | 2 |
| /32 | 255.255.255.255 | 1 | a single host | 1 |

Usable hosts = 2^(32 − prefix) − 2 (network and broadcast addresses are reserved).

### Calculating a subnet by hand – the block-size method

Example: **`192.168.56.77/26`**

1. /26 → the last octet is the interesting one; mask `192` → **block size** = 256 − 192 = **64**.
2. Subnets start at multiples of 64: `.0`, `.64`, `.128`, `.192`.
3. 77 falls in the block starting at **64**.

| | Address |
|--|---------|
| Network | `192.168.56.64` |
| First host | `192.168.56.65` |
| Last host | `192.168.56.126` |
| Broadcast | `192.168.56.127` |

Example: **`172.20.13.200/20`**

1. /20 → the 3rd octet is interesting; mask `240` → block size **16**.
2. Blocks in the 3rd octet: 0, 16, 32… → 13 is in block **0**.
3. Network `172.20.0.0`, broadcast `172.20.15.255`, hosts `172.20.0.1` – `172.20.15.254`.

### Same subnet or not?

Two hosts can talk directly (without a router) only if they are on the same subnet. `192.168.56.70/26` and `192.168.56.130/26` are in **different** subnets (64–127 vs 128–191) – traffic must go through a router.

### Let a tool check you

```console
$ ipcalc 192.168.56.77/26          # package ipcalc (Debian output shown)
Address:   192.168.56.77        11000000.10101000.00111000.01 001101
Netmask:   255.255.255.192 = 26 11111111.11111111.11111111.11 000000
Network:   192.168.56.64/26
HostMin:   192.168.56.65
HostMax:   192.168.56.126
Broadcast: 192.168.56.127
Hosts/Net: 62                    Class C, Private Internet
```

On RHEL: `ipcalc -n -b 192.168.56.77/26` (different output format). Python works everywhere: `python3 -c "import ipaddress; n=ipaddress.ip_interface('192.168.56.77/26').network; print(n, n.broadcast_address)"`.

---

## Special IPv4 ranges

| Range | Purpose |
|-------|---------|
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | **private** (RFC 1918) – not routed on the internet; reach it via NAT |
| `127.0.0.0/8` | **loopback** – `127.0.0.1` is this host |
| `169.254.0.0/16` | **link-local** – self-assigned when DHCP fails (APIPA) |
| `100.64.0.0/10` | carrier-grade NAT |
| `0.0.0.0` | "any address" (listening sockets) / "unspecified"; `0.0.0.0/0` = default route |
| `224.0.0.0/4` | multicast |
| `255.255.255.255` | limited broadcast |
| `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24` | documentation examples |

---

## IPv6

IPv6 addresses are **128 bits**, written as eight groups of four hex digits:

```text
2001:0db8:0000:0000:0000:ff00:0042:8329
```

Shortening rules:

1. Drop leading zeros in each group: `2001:db8:0:0:0:ff00:42:8329`
2. Replace **one** run of consecutive zero groups with `::` : `2001:db8::ff00:42:8329`

`::1` is loopback; `::` is the unspecified address. With ports, IPv6 addresses are written in brackets: `[2001:db8::10]:443`.

### Prefixes

- Subnets are almost always **/64**: the first 64 bits are the network, the last 64 the **interface identifier**.
- Organizations typically receive a /48 or /56 and carve /64 subnets from it.
- There is **no broadcast** in IPv6 – multicast is used instead.

### Address types

| Prefix | Type | Example / use |
|--------|------|--------------|
| `::1/128` | loopback | |
| `fe80::/10` | **link-local** – every interface always has one; valid only on that link | `fe80::5054:ff:fe3a:1f9c` |
| `2000::/3` | **global unicast** – public, routable | `2001:db8::10` (documentation range) |
| `fc00::/7` (in practice `fd00::/8`) | **unique local** (ULA) – private, like RFC 1918 | `fd00:56::20` |
| `ff00::/8` | multicast | `ff02::1` all nodes on the link |

```console
$ ip -6 addr show enp1s0
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet6 fd00:56::10/64 scope global
    inet6 fe80::5054:ff:fe3a:1f9c/64 scope link
```

### How IPv6 hosts get addresses

| Method | How |
|--------|-----|
| **SLAAC** | the router advertises the prefix (Router Advertisement); the host builds its own address from prefix + interface ID |
| **DHCPv6** | a server assigns addresses (stateful) or only options like DNS (stateless) |
| **Static** | configured manually |

**NDP** (Neighbor Discovery Protocol, part of ICMPv6) replaces ARP and also finds routers and detects duplicate addresses. Blocking all ICMPv6 breaks IPv6 – firewalls must allow it.

---

## Ports

Ports identify services on a host. `/etc/services` maps names to numbers.

| Port | Service | Port | Service |
|------|---------|------|---------|
| 22/tcp | SSH | 123/udp | NTP |
| 25/tcp | SMTP | 389, 636/tcp | LDAP, LDAPS |
| 53/tcp+udp | DNS | 443/tcp | HTTPS |
| 67, 68/udp | DHCP | 2049/tcp | NFS |
| 80/tcp | HTTP | 3260/tcp | iSCSI |
| 111/tcp+udp | rpcbind | 3306 / 5432 | MySQL / PostgreSQL |

- 0–1023: well-known (privileged – binding needs root or `CAP_NET_BIND_SERVICE`).
- 1024–49151: registered. 49152–65535: dynamic/ephemeral (client side of connections; Linux uses `net.ipv4.ip_local_port_range`).

---

## Try it yourself

1. Find the MAC, IPv4 and IPv6 addresses of each interface (`ip -br link`, `ip -br addr`).
2. For your lab IP, compute by hand the network, broadcast and host range; check with `ipcalc`.
3. Compute the same for `10.20.30.40/22` and `172.16.5.130/25`.
4. Shorten `2001:0db8:0000:0000:0001:0000:0000:0010` correctly.
5. Ping the other VM, then look at `ip neigh`. Ping its IPv6 link-local address: `ping -6 fe80::…%enp1s0`.

---

## Common mistakes

- Assigning the network or broadcast address to a host.
- Assuming two hosts are on the same subnet because the first three octets match (only true for /24).
- Using `::` twice in an IPv6 address.
- Forgetting the `%interface` scope when pinging a link-local IPv6 address.
- Blocking ICMPv6 completely.

---

## Summary

- MAC = link-local delivery; IP = routing between networks; port = service.
- Prefix → block size → network/broadcast/host range.
- Private: `10/8`, `172.16/12`, `192.168/16`. Loopback `127/8`, link-local `169.254/16`.
- IPv6: 128 bits, `/64` subnets, link-local `fe80::/10` always present, ULA `fd00::/8`, SLAAC + NDP.

## Check yourself

1. Network and broadcast of `10.1.2.130/25`?
2. How many usable hosts in a /27?
3. Are `192.168.10.5/23` and `192.168.11.200/23` on the same subnet?
4. Which IPv6 range is the equivalent of private IPv4 addresses?
5. What replaces ARP in IPv6?

<details>
<summary>Answers</summary>

1. Network `10.1.2.128`, broadcast `10.1.2.255`.
2. 30.
3. Yes – /23 covers `192.168.10.0` – `192.168.11.255`.
4. Unique local addresses, `fd00::/8` (from `fc00::/7`).
5. NDP (Neighbor Discovery Protocol, ICMPv6).

</details>

## Further reading

- `man 8 ip-address`, `man 7 ip`, `man 7 ipv6`, `man 5 services`
- [Red Hat: Configuring and managing networking (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_and_managing_networking/index)
- [Subnetting practice](https://subnettingpractice.com/) · [ipcalc online](https://jodies.de/ipcalc)
- [RFC 1918 – Private address space](https://www.rfc-editor.org/rfc/rfc1918) · [RFC 4291 – IPv6 addressing](https://www.rfc-editor.org/rfc/rfc4291)
- [Julia Evans: Networking zines](https://wizardzines.com/)

---

<!-- footer:start -->
[Module index](README.md) · [Runtime Network Configuration with ip →](02-runtime-ip.md)  
**Practice now:** [Exercises – Network Addresses: IPv4 and IPv6](../../exercises/07-networking/01-ip-addressing.md)
<!-- footer:end -->
