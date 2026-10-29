# Kernel Parameters with sysctl

<!-- nav:start -->
> **Module 06 · Kernel, Modules & Devices** — lesson 2 of 4  
> [Module index](README.md) · [Course lesson (commands)](../../course/06-Kernel-Modules-Devices.md#lesson-2---sysctl-kernel-parameters-ch-24) · [Exercises for this lesson](../../exercises/06-kernel-devices/02-sysctl.md)
<!-- nav:end -->

Many kernel behaviours can be changed **while the system runs**: whether it forwards packets between interfaces, how eagerly it swaps, how many files can be open, whether it answers pings. These *tunables* live under `/proc/sys/` and are managed with **sysctl**. The LFCS competency is explicit: *configure kernel parameters, persistent and non-persistent*. In this lesson you will learn both.

## What you will learn

- What `/proc/sys` is and how sysctl names map to paths
- Reading parameters
- Non-persistent changes (`sysctl -w`, writing to `/proc/sys`)
- Persistent changes (`/etc/sysctl.d/*.conf`) and load order
- Applying configuration without rebooting
- Commonly used parameters, by category
- Parameters that depend on modules; per-interface network parameters

---

## Key terms

| Term | Definition |
|------|-----------|
| **Kernel tunable** | A runtime-adjustable kernel setting. |
| **`/proc/sys/`** | Virtual directory where every tunable is a file. |
| **sysctl** | The command (and naming scheme) to read and set tunables: `net.ipv4.ip_forward`. |
| **Non-persistent** | Set in the running kernel only; lost at reboot. |
| **Persistent** | Stored in a configuration file and applied at every boot by `systemd-sysctl`. |
| **`systemd-sysctl`** | Service that applies `sysctl.d` files at boot. |

---

## Names and paths

A sysctl name is the path below `/proc/sys/` with `/` replaced by `.`:

```text
net.ipv4.ip_forward        ↔   /proc/sys/net/ipv4/ip_forward
vm.swappiness              ↔   /proc/sys/vm/swappiness
kernel.hostname            ↔   /proc/sys/kernel/hostname
fs.file-max                ↔   /proc/sys/fs/file-max
```

| Top-level | Area |
|-----------|------|
| `kernel.*` | general kernel (hostname, panic behaviour, sysrq, pid_max) |
| `vm.*` | virtual memory (swappiness, dirty page writeback, overcommit) |
| `fs.*` | filesystems (file handle limits, inotify) |
| `net.*` | networking (IPv4/IPv6, core buffers, bridge netfilter) |
| `dev.*` | device-specific |

---

## Reading

```console
$ sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 0
$ sysctl -n vm.swappiness                  # value only
60
$ cat /proc/sys/net/ipv4/ip_forward
0
$ sysctl -a | grep -i forward | head       # all parameters, filtered
net.ipv4.conf.all.forwarding = 0
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0
$ sysctl -a 2>/dev/null | wc -l
1100
```

---

## Non-persistent changes

Take effect immediately, lost at reboot:

```console
$ sudo sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
$ echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward        # equivalent
$ sudo sysctl -w net.ipv4.icmp_echo_ignore_all=1          # stop answering pings
```

Note: `sudo echo 1 > /proc/sys/...` fails because the redirection runs without root – use `tee` or `sysctl -w`.

---

## Persistent changes

Write `key = value` lines into a file in **`/etc/sysctl.d/`**, ending in `.conf`:

```console
$ echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/90-forward.conf
$ sudo sysctl --system                       # apply all config files now
* Applying /usr/lib/sysctl.d/50-default.conf ...
* Applying /etc/sysctl.d/90-forward.conf ...
net.ipv4.ip_forward = 1
$ sudo sysctl -p /etc/sysctl.d/90-forward.conf   # apply one file
```

At boot, `systemd-sysctl.service` applies the files automatically.

### File locations and order

```text
/usr/lib/sysctl.d/*.conf     ← shipped by packages (don't edit)
/run/sysctl.d/*.conf         ← runtime
/etc/sysctl.d/*.conf         ← YOUR settings
/etc/sysctl.conf             ← legacy single file, still read (applied last on many distros)
```

All files are sorted by **file name** across these directories (a file in `/etc` overrides a same-named file in `/usr/lib`) and applied in that order; **the last value wins**. Name your files with a high number, e.g. `90-` or `99-`, to override defaults.

Syntax:

```text
# comment
; also a comment
net.ipv4.ip_forward = 1
vm.swappiness=10
-net.bridge.bridge-nf-call-iptables = 1     # leading "-": ignore if the key doesn't exist
```

> Keep `/etc/sysctl.conf` as is on modern systems and create your own file in `/etc/sysctl.d/` – easy to find, document and remove.

---

## Persistent and non-persistent at the same time

Exam tasks often mix both:

> *Set `net.ipv4.ip_forward=1` persistently and `kernel.sysrq=0` only for the running system.*

```console
$ echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/90-forward.conf
$ sudo sysctl --system
$ sudo sysctl -w kernel.sysrq=0
```

Verify: both values are active now; after a reboot only `ip_forward` remains.

> Careful: `sysctl --system` re-applies all files; if a file sets `kernel.sysrq`, it will override your runtime value. Run `sysctl -w` **after** `sysctl --system`.

---

## Commonly used parameters

### Networking

| Parameter | Typical value | Purpose |
|-----------|--------------|---------|
| `net.ipv4.ip_forward` | `1` | route packets between interfaces (router, NAT, containers, VPN) |
| `net.ipv6.conf.all.forwarding` | `1` | IPv6 routing |
| `net.ipv4.icmp_echo_ignore_all` | `1` | ignore ping |
| `net.ipv4.icmp_echo_ignore_broadcasts` | `1` | ignore broadcast pings (smurf protection) |
| `net.ipv4.conf.all.rp_filter` | `1`/`2` | reverse path filtering (anti-spoofing) |
| `net.ipv4.conf.all.accept_redirects` | `0` | ignore ICMP redirects |
| `net.ipv6.conf.all.disable_ipv6` | `1` | disable IPv6 |
| `net.ipv4.ip_local_port_range` | `1024 65000` | ports for outgoing connections |
| `net.core.somaxconn` | `4096` | listen backlog |
| `net.ipv4.tcp_syncookies` | `1` | SYN flood protection |

Per-interface settings use the interface name: `net.ipv4.conf.eth1.forwarding`, `net.ipv6.conf.eth1.disable_ipv6`. `all` applies to every interface; `default` to interfaces created later.

### Memory

| Parameter | Purpose |
|-----------|---------|
| `vm.swappiness` | eagerness to swap (see [Swap](../05-storage/05-swap.md)) |
| `vm.dirty_ratio`, `vm.dirty_background_ratio` | how much RAM may hold unwritten data before writes block / background flushing starts |
| `vm.overcommit_memory` | memory overcommit policy (0 heuristic, 1 always, 2 strict) |
| `vm.max_map_count` | memory map areas per process (Elasticsearch needs more) |

### Kernel and filesystems

| Parameter | Purpose |
|-----------|---------|
| `kernel.hostname` | hostname (prefer `hostnamectl`) |
| `kernel.sysrq` | magic SysRq key functions (0 disables) |
| `kernel.pid_max` | maximum PID |
| `kernel.panic` | seconds before reboot after a panic |
| `fs.file-max` | system-wide maximum open files |
| `fs.inotify.max_user_watches` | inotify watches per user (IDEs, sync tools) |

Documentation for every parameter: `/usr/share/doc/kernel-doc*/Documentation/admin-guide/sysctl/` (if installed), or online at docs.kernel.org.

---

## Parameters provided by modules

Some keys only exist when a module is loaded. Example: `net.bridge.bridge-nf-call-iptables` (Kubernetes, containers) needs `br_netfilter`:

```console
$ sudo sysctl -w net.bridge.bridge-nf-call-iptables=1
sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-iptables: No such file or directory
$ sudo modprobe br_netfilter
$ echo br_netfilter | sudo tee /etc/modules-load.d/br_netfilter.conf    # load at boot too
$ sudo sysctl -w net.bridge.bridge-nf-call-iptables=1
```

Modules are covered in the [next lesson](03-kernel-modules.md).

---

## Other places that change kernel settings

- **`/sys`** – device and driver attributes (I/O scheduler, CPU frequency governor). Not managed by sysctl; persist with udev rules or `tmpfiles.d`.
- **tuned** (RHEL) – applies whole *profiles* of sysctl and sysfs settings: `tuned-adm list`, `tuned-adm profile throughput-performance`.
- Applications or container runtimes may set some values at startup (e.g. Docker enables `ip_forward`).

---

## Try it yourself

1. Read `net.ipv4.ip_forward` three different ways.
2. Enable IPv4 forwarding non-persistently; reboot; verify it's off again.
3. Make `vm.swappiness=10` persistent; apply without rebooting; reboot and verify.
4. Make the system ignore pings persistently; test from the other VM; revert and remove the file.
5. Find which configuration file on your system sets `kernel.sysrq` (`grep -r sysrq /usr/lib/sysctl.d /etc/sysctl.d /etc/sysctl.conf`).

---

## Common mistakes

- Spaces or typos in key names (`net.ipv4.ip-forward`) – sysctl reports "unknown key".
- Writing the file but not applying it (`sysctl --system`) when the task requires it now.
- Using `sysctl -w` when the task requires persistence (or vice versa).
- File in `/etc/sysctl.d/` without the `.conf` suffix (ignored).
- Being overridden by a later file – check with `sysctl --system` output and `systemd-analyze cat-config sysctl.d`.

---

## Summary

| Task | Command / file |
|------|----------------|
| Read | `sysctl key`, `sysctl -a`, `cat /proc/sys/…` |
| Set now (non-persistent) | `sysctl -w key=value`, `echo v \| tee /proc/sys/…` |
| Set permanently | `/etc/sysctl.d/90-name.conf` with `key = value` |
| Apply files now | `sysctl --system` or `sysctl -p FILE` |
| See effective config | `systemd-analyze cat-config sysctl.d` |

## Check yourself

1. What is the `/proc/sys` path of `net.ipv6.conf.all.forwarding`?
2. Two files set `vm.swappiness`: `/etc/sysctl.d/10-a.conf` (=20) and `/etc/sysctl.d/90-b.conf` (=50). Which value wins?
3. You set a value with `sysctl -w`. Is it still active after reboot?
4. `sysctl -w net.bridge.bridge-nf-call-iptables=1` says "No such file". Why?

<details>
<summary>Answers</summary>

1. `/proc/sys/net/ipv6/conf/all/forwarding`.
2. 50 – files are applied in name order and the last value wins.
3. No.
4. The `br_netfilter` module that provides the key isn't loaded.

</details>

## Further reading

- `man 8 sysctl`, `man 5 sysctl.d`, `man 5 proc` (section `/proc/sys`)
- [Kernel docs: Documentation for /proc/sys](https://docs.kernel.org/admin-guide/sysctl/index.html) · [IP sysctl](https://docs.kernel.org/networking/ip-sysctl.html)
- [ArchWiki: sysctl](https://wiki.archlinux.org/title/Sysctl)
- [Red Hat: Configuring kernel parameters at runtime (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_monitoring_and_updating_the_kernel/configuring-kernel-parameters-at-runtime_managing-monitoring-and-updating-the-kernel)

---

<!-- footer:start -->
[← The Kernel and Boot Parameters](01-kernel-boot-parameters.md) · [Module index](README.md) · [Kernel Modules →](03-kernel-modules.md)  
**Practice now:** [Exercises – Kernel Parameters with sysctl](../../exercises/06-kernel-devices/02-sysctl.md)
<!-- footer:end -->
