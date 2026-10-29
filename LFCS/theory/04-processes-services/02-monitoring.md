# Monitoring System Performance

<!-- nav:start -->
> **Module 04 · Processes, Services, Scheduling & Performance** — lesson 2 of 6  
> [Module index](README.md) · [Course lesson (commands)](../../course/04-Processes-Services-Performance.md#lesson-2---monitoring-ch-1517) · [Exercises for this lesson](../../exercises/04-processes-services/02-monitoring.md)
<!-- nav:end -->

"The server is slow" is one of the most common tickets an administrator receives. To answer it you need to know what to measure – CPU, memory, disk I/O, network – how to read the numbers, and how to find the process responsible. In this lesson you will learn the core monitoring tools and a simple method to investigate performance problems.

## What you will learn

- The **USE method**: utilization, saturation, errors
- CPU metrics: load average, user/system/iowait/steal time
- Memory metrics: used, free, **available**, buffers/cache, swap activity
- Disk I/O metrics: IOPS, throughput, latency, utilization
- The tools: `uptime`, `top`/`htop`, `vmstat`, `mpstat`, `free`, `iostat`, `iotop`, `pidstat`, `sar`, `lsof`, `ss`
- How to find the process behind a problem

---

## Key terms

| Term | Definition |
|------|-----------|
| **Utilization** | How busy a resource is (e.g. CPU 80% busy). |
| **Saturation** | How much work is waiting because the resource is busy (run queue, I/O queue). |
| **Load average** | Average number of tasks running, waiting for CPU, or in uninterruptible (I/O) sleep, over 1, 5 and 15 minutes. |
| **iowait (`wa`)** | Percentage of time the CPU was idle while some task was waiting for disk I/O. |
| **steal (`st`)** | Time a virtual CPU wanted to run but the hypervisor gave the physical CPU to someone else. |
| **Page cache** | RAM used to cache file contents; freed automatically when programs need memory. |
| **Swap in/out** | Moving memory pages between RAM and swap; constant activity means RAM is too small. |
| **IOPS** | I/O operations per second. |
| **Latency (`await`)** | Average time an I/O request takes, including queueing. |

---

## A method: USE

For **each** resource, ask three questions:

| | CPU | Memory | Disk | Network |
|--|-----|--------|------|---------|
| **U**tilization | `top` `%Cpu`, `mpstat` | `free` used | `iostat %util` | `sar -n DEV`, `ip -s link` |
| **S**aturation | load average > CPUs, `vmstat r` | swap in/out (`vmstat si/so`), OOM kills | `iostat aqu-sz`, `await` | drops, retransmits (`ss -s`, `nstat`) |
| **E**rrors | `dmesg` (MCE) | `dmesg` (ECC, OOM) | `dmesg`, SMART | `ip -s link` errors |

Checking all of them quickly avoids the classic mistake of blaming the CPU for a disk problem.

### A 60-second checklist

```console
$ uptime                 # load averages
$ dmesg -T | tail        # recent kernel errors (OOM, disk, network)
$ vmstat 1 5             # run queue, memory, swap, I/O, CPU split
$ mpstat -P ALL 1 3      # per-CPU balance
$ free -h                # memory and swap
$ iostat -xz 1 3         # disk latency and utilization
$ pidstat 1 3            # per-process CPU
$ ss -s                  # socket summary
$ top                    # overall view, find the culprit
```

`mpstat`, `iostat`, `pidstat` and `sar` come from the **sysstat** package (`apt/dnf install sysstat`).

---

## CPU and load

### uptime and load average

```console
$ uptime
 10:15:02 up 3 days,  2:11,  2 users,  load average: 3.84, 2.10, 1.05
$ nproc
2
```

- Three numbers: last **1, 5 and 15** minutes. Here load is rising (3.84 > 2.10 > 1.05).
- Compare with the number of CPUs: on 2 CPUs, a load of 2 means fully busy; 3.84 means tasks are queuing (saturation).
- On Linux the load includes tasks in **D state** (waiting for disk), so a high load with idle CPUs usually means an **I/O** problem.

### top

```console
$ top
top - 10:15:30 up 3 days,  2:11,  2 users,  load average: 3.84, 2.10, 1.05
Tasks: 142 total,   3 running, 139 sleeping,   0 stopped,   0 zombie
%Cpu(s): 92.1 us,  6.2 sy,  0.0 ni,  0.0 id,  1.5 wa,  0.0 hi,  0.2 si,  0.0 st
MiB Mem :   1964.1 total,    112.4 free,   1203.7 used,    648.0 buff/cache
MiB Swap:   2048.0 total,   1980.2 free,     67.8 used.    560.3 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   4521 alice     20   0    8352   1024    896 R  99.7   0.1   2:10.33 yes
   4610 www-data  20   0  210444  48120  10240 S  80.1   2.4   1:01.02 php-fpm
```

| CPU field | Meaning | High value suggests |
|-----------|---------|---------------------|
| `us` | user-space code | busy applications |
| `sy` | kernel code | many system calls, network/disk-heavy work |
| `ni` | niced user processes | background jobs |
| `id` | idle | – |
| `wa` | waiting for I/O | **disk bottleneck** |
| `hi`/`si` | hardware/software interrupts | network floods |
| `st` | stolen by hypervisor | overloaded host (VMs) |

Interactive keys: `P` sort by CPU, `M` by memory, `1` per-CPU view, `k` kill, `r` renice, `u` filter by user, `c` full command, `H` threads, `q` quit. Batch mode for scripts: `top -b -n 1 | head -15`.

`htop` (package `htop`) is a friendlier alternative with colors and mouse support.

### vmstat

```console
$ vmstat 1 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 3  0  69432 115120  20480 643000    0    0     4    12  410  780 92  6  0  2  0
 4  1  69432 114980  20480 643010    0    0     0   850  420  790 90  7  0  3  0
```

| Column | Meaning |
|--------|---------|
| `r` | runnable tasks (waiting for CPU) – compare with CPU count |
| `b` | tasks blocked in uninterruptible sleep (I/O) |
| `si` / `so` | KiB/s swapped in / out – should be ~0 |
| `bi` / `bo` | blocks read / written per second |
| `in` / `cs` | interrupts / context switches per second |
| `us sy id wa st` | CPU split |

The first line is an average since boot; read the following lines.

### Per-CPU and per-process CPU

```console
$ mpstat -P ALL 1 1          # is one CPU at 100% while others idle? (single-threaded app)
$ pidstat 1 3                # %usr %system per process
$ pidstat -t -p 4610 1       # per thread
```

---

## Memory

```console
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           1.9Gi       1.2Gi       112Mi        12Mi       648Mi       560Mi
Swap:          2.0Gi        67Mi       1.9Gi
```

| Column | Meaning |
|--------|---------|
| `used` | memory used by processes and the kernel |
| `free` | completely unused RAM – **normally small, and that's fine** |
| `buff/cache` | page cache and buffers – reclaimable |
| `available` | estimate of memory available for new programs **without swapping** – the number to watch |

Linux deliberately fills unused RAM with cache ("Linux ate my RAM"). A low `free` with a healthy `available` is normal. Trouble signs:

- `available` near zero,
- constant `si`/`so` in `vmstat`,
- **OOM killer** messages:

```console
$ journalctl -k | grep -i -E "out of memory|oom-kill"
Oct 21 09:58:12 web01 kernel: Out of memory: Killed process 4610 (php-fpm) total-vm:2104440kB, anon-rss:1650000kB
```

Details: `/proc/meminfo`; top consumers: `ps -eo pid,rss,cmd --sort=-rss | head`, or `top` then `M`. Swap configuration is covered in [Swap](../05-storage/05-swap.md).

---

## Disk I/O

```console
$ iostat -xz 1 3
Device   r/s    w/s   rkB/s   wkB/s  r_await w_await aqu-sz  %util
vda     2.00 310.00   16.00 39680.0     0.50   45.20  14.01  99.80
```

| Column | Meaning |
|--------|---------|
| `r/s`, `w/s` | read/write IOPS |
| `rkB/s`, `wkB/s` | throughput |
| `r_await`, `w_await` | average latency per request (ms) – includes queue time |
| `aqu-sz` | average queue length (saturation) |
| `%util` | percent of time the device was busy |

Here writes take 45 ms each and the device is 99.8% busy: **saturated**. (`%util` can be misleading for SSD/RAID/NVMe, which serve many requests in parallel; trust `await` and `aqu-sz` more.)

Which process?

```console
$ sudo iotop -o              # only processes doing I/O (package iotop)
$ pidstat -d 1               # kB_rd/s kB_wr/s per process
```

More in [Storage Performance](../05-storage/09-storage-performance.md).

---

## Open files and sockets

```console
$ sudo lsof -p 905 | head           # files opened by a process
$ sudo lsof /var/log/syslog         # who has this file open
$ sudo lsof -u alice                # files opened by a user
$ sudo lsof -i :80                  # who uses TCP/UDP port 80
$ sudo fuser -vm /mnt/data          # processes using a mount point
$ ss -tulpn                         # listening TCP/UDP sockets with process
Netid State  Local Address:Port  Process
tcp   LISTEN 0.0.0.0:22          users:(("sshd",pid=812,fd=3))
tcp   LISTEN 0.0.0.0:80          users:(("nginx",pid=905,fd=6))
```

---

## History: sar

`top` shows **now**. To know what happened at 03:00, use **sar**, which samples every 10 minutes once enabled:

```console
$ sudo systemctl enable --now sysstat       # RHEL: sysstat-collect.timer; Debian: also set ENABLED="true" in /etc/default/sysstat
$ sar -u                                    # CPU today
$ sar -r                                    # memory
$ sar -d                                    # disks
$ sar -n DEV                                # network
$ sar -q -s 02:00:00 -e 04:00:00            # load between 2 and 4 AM
$ sar -u -f /var/log/sa/sa20                # another day (RHEL path; Debian: /var/log/sysstat/)
```

---

## Example investigation

> Ticket: *"web01 is slow since 10:00."*

1. `uptime` → load 6 on 2 CPUs (saturated).
2. `vmstat 1` → `r`=1, `b`=5, `wa`=60% → tasks are blocked on **I/O**, not CPU.
3. `iostat -xz 1` → `vda` `%util` 100%, `w_await` 200 ms.
4. `iotop -o` → `tar` process writing at 80 MB/s, started by a cron backup.
5. Fix: `ionice -c3 -p <pid>` now; reschedule the backup and run it with `nice`/`ionice`.

---

## Try it yourself

1. Run `stress`-like load with `yes > /dev/null &` (twice) and watch `uptime`, `top` and `mpstat -P ALL 1`.
2. Generate disk writes with `dd if=/dev/zero of=/tmp/big bs=1M count=2000 oflag=direct` and watch `iostat -xz 1` and `iotop -o`.
3. Find the 5 processes using the most memory.
4. Find which process listens on port 22 with two different tools.
5. Enable sysstat and, after a while, view today's CPU history with `sar -u`.

---

## Summary

| Question | Tool |
|----------|------|
| Is the system overloaded? | `uptime`, `vmstat 1` (`r`, `b`) |
| Who uses CPU? | `top` (`P`), `pidstat`, `ps --sort=-%cpu` |
| Is memory short? | `free -h` (available), `vmstat` (`si/so`), OOM in `journalctl -k` |
| Who uses memory? | `top` (`M`), `ps --sort=-rss` |
| Is the disk the bottleneck? | `iostat -xz 1` (`await`, `aqu-sz`, `%util`), `vmstat wa` |
| Who does I/O? | `iotop -o`, `pidstat -d` |
| Who has a file/port open? | `lsof`, `fuser`, `ss -tulpn` |
| What happened earlier? | `sar` |

## Check yourself

1. Load average 4 on a 4-CPU system with 0% idle CPU. Saturated?
2. `free` shows 100 MiB free and 1.4 GiB available. Should you add RAM?
3. High load, CPUs mostly idle, high `wa`. What's the likely bottleneck?
4. Which `vmstat` columns indicate memory pressure?

<details>
<summary>Answers</summary>

1. Fully utilized but just at the limit; above 4 tasks would start queuing.
2. No – most "used" memory is reclaimable cache; 1.4 GiB is available.
3. Disk I/O (tasks waiting in D state count towards load).
4. `si`/`so` (swap in/out) consistently above zero, and low `free` with high `b`.

</details>

## Further reading

- `man 1 top`, `man 8 vmstat`, `man 1 free`, `man 1 iostat`, `man 1 sar`, `man 8 lsof`
- [Brendan Gregg: The USE Method](https://www.brendangregg.com/usemethod.html) · [Linux Performance](https://www.brendangregg.com/linuxperf.html)
- [Linux Performance Analysis in 60,000 Milliseconds (Netflix)](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55)
- [Linux ate my RAM!](https://www.linuxatemyram.com/)

---

<!-- footer:start -->
[← Processes](01-processes.md) · [Module index](README.md) · [Troubleshooting Disk Space →](03-disk-space.md)  
**Practice now:** [Exercises – Monitoring System Performance](../../exercises/04-processes-services/02-monitoring.md)
<!-- footer:end -->
