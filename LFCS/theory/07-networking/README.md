# Module 07 – Networking: Theory

> [All theory](../00-Theory-Index.md) · [Course module (commands & lab)](../../course/07-Networking.md) · [Module exercises](../../exercises/07-networking/README.md)

Read the lessons in order. Each one explains the concepts with definitions and worked examples, then links to its exercises.

| # | Lesson | About | Practice |
|---|--------|----------------|----------|
| 1 | [Network Addresses: IPv4 and IPv6](01-ip-addressing.md) | Every networking task – assigning addresses, adding routes, writing firewall rules – assumes you can read an address like `192.168.56.77/26` and immediately know its network, its broadcast address and which hosts are on the same link. | [Exercises](../../exercises/07-networking/01-ip-addressing.md) |
| 2 | [Runtime Network Configuration with ip](02-runtime-ip.md) | The `ip` command (package `iproute2`) talks directly to the kernel's networking stack: it shows and changes interfaces, addresses, routes and neighbour tables. | [Exercises](../../exercises/07-networking/02-runtime-ip.md) |
| 3 | [Persistent Network Configuration: Netplan and NetworkManager](03-persistent-config.md) | Network settings must survive reboots. | [Exercises](../../exercises/07-networking/03-persistent-config.md) |
| 4 | [Hostname and Name Resolution](04-hostname-resolution.md) | Humans use names – `web01`, `db.example.lab`, `www.kernel.org` – while the network uses IP addresses. | [Exercises](../../exercises/07-networking/04-hostname-resolution.md) |
| 5 | [Static Routing](05-static-routing.md) | A **route** tells the kernel where to send a packet. | [Exercises](../../exercises/07-networking/05-static-routing.md) |
| 6 | [Bridges and Bonds](06-bridges-bonds.md) | Linux can build **virtual network devices** on top of physical NICs. | [Exercises](../../exercises/07-networking/06-bridges-bonds.md) |
| 7 | [Packet Filtering, Port Redirection and NAT](07-firewalls-nat.md) | A firewall decides which packets may enter, leave or pass through a host. | [Exercises](../../exercises/07-networking/07-firewalls-nat.md) |
| 8 | [The OpenSSH Server and Client](08-openssh.md) | **SSH** (*Secure Shell*) is how administrators reach Linux servers: encrypted remote logins, remote commands, file transfers and tunnels. | [Exercises](../../exercises/07-networking/08-openssh.md) |
| 9 | [Setting and Synchronizing System Time](09-time-sync.md) | Correct time is critical: log timestamps must line up across servers, TLS certificates have validity dates, Kerberos rejects clocks that are more than 5 minutes off, cron and timers fire at the wrong moment, and distributed databases can corrupt data with skewed clocks. | [Exercises](../../exercises/07-networking/09-time-sync.md) |
| 10 | [Reverse Proxies and Load Balancers](10-reverse-proxy.md) | Most web applications are not exposed directly to users. | [Exercises](../../exercises/07-networking/10-reverse-proxy.md) |
| 11 | [Monitoring and Troubleshooting the Network](11-troubleshooting.md) | "I can't reach the server" can mean a dozen different problems: a cable, an address, a route, DNS, a firewall, a stopped service, a service listening on the wrong address, or SELinux. | [Exercises](../../exercises/07-networking/11-troubleshooting.md) |

**Suggested routine per lesson:** read the theory (30–45 min) → try the commands on your lab VMs → do the exercises without opening the solutions → answer *Check yourself*.
