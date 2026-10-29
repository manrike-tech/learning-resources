# Exercises – Monitoring System Performance

<!-- nav:start -->
> **Module 04 · Processes, Services, Scheduling & Performance** — lesson 2 of 6  
> Read first: [Theory – Monitoring System Performance](../../theory/04-processes-services/02-monitoring.md) · [Course lesson (commands)](../../course/04-Processes-Services-Performance.md#lesson-2---monitoring-ch-1517) · [Module exercises](README.md)
<!-- nav:end -->

> Install the tools first: `apt install -y sysstat iotop htop` or `dnf install -y sysstat iotop htop` (htop needs EPEL on RHEL).

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Baseline ★

Record a baseline of the idle system in `/root/baseline.txt`: uptime/load, CPU count, `free -h`, one `vmstat 1 3` and one `iostat -xz 1 2`.

<details>
<summary>Solution</summary>

```bash
{ uptime; nproc; free -h; vmstat 1 3; iostat -xz 1 2; } > /root/baseline.txt
```

</details>

---

## Exercise 2 – CPU load ★★

1. Start one CPU-burning process per CPU (`for i in $(seq $(nproc)); do yes > /dev/null & done`).
2. Watch `uptime` for 2 minutes. How do the three load numbers evolve?
3. In `vmstat 1`, which columns change?
4. In `top`, sort by CPU and kill the processes from inside `top`.

<details>
<summary>Solution</summary>

1-minute load rises towards `nproc` first, 5- and 15-minute values follow slowly. In `vmstat`, `r` ≈ number of CPUs and `us` ≈ 100, `id` ≈ 0. In `top` press `P`, then `k` and the PID (or `pkill yes`).

</details>

---

## Exercise 3 – Per-CPU imbalance ★

Run a single `yes > /dev/null` on a multi-CPU VM. Show with `mpstat -P ALL 1 3` that one CPU is at 100% while the total looks moderate.

<details>
<summary>Solution</summary>

```bash
yes > /dev/null & mpstat -P ALL 1 3; kill $!
```

A single-threaded process can use at most one CPU – "50% total" on 2 CPUs may mean one CPU is saturated.

</details>

---

## Exercise 4 – Memory ★★

1. Read `free -h` and explain `free` vs `available`.
2. Drop the page cache (`sync; echo 3 > /proc/sys/vm/drop_caches`) and read again. Then `cat` a large file (e.g. `find /usr -type f -exec cat {} + > /dev/null`) and watch `buff/cache` grow.
3. List the 5 biggest memory consumers by RSS.

<details>
<summary>Solution</summary>

```bash
free -h
sync; echo 3 | sudo tee /proc/sys/vm/drop_caches; free -h
find /usr -type f -exec cat {} + > /dev/null 2>&1; free -h
ps -eo pid,rss,cmd --sort=-rss | head -6
```

</details>

---

## Exercise 5 – Disk I/O ★★

1. In one terminal run `iostat -xz 1`; in another write 2 GB with `dd if=/dev/zero of=/var/tmp/io.test bs=1M count=2000 oflag=direct`.
2. Which columns show the load? Which process does `iotop -o` show?
3. Repeat with `ionice -c3` and compare.

<details>
<summary>Solution</summary>

`w/s`, `wkB/s`, `w_await`, `aqu-sz` and `%util` rise; `vmstat` shows `bo` and `wa`. `sudo iotop -o` shows `dd`. Then `ionice -c3 dd …`. Remove `/var/tmp/io.test` afterwards.

</details>

---

## Exercise 6 – Files and ports ★

1. Which process listens on TCP 22? Use two tools.
2. Which processes have files open under `/var/log`?
3. Which process prevents `umount` of a mount point? (Test: `cd /mnt/somefs` in another shell.)

<details>
<summary>Solution</summary>

```bash
sudo ss -tlnp | grep ':22 '; sudo lsof -i :22
sudo lsof +D /var/log 2>/dev/null | head
sudo fuser -vm /mnt/somefs
```

</details>

---

## Exercise 7 – History with sar ★★

Enable sysstat collection, wait ~20 minutes (or generate load meanwhile), then show today's CPU, memory and disk history.

<details>
<summary>Solution</summary>

```bash
# Debian/Ubuntu
sed -i 's/ENABLED="false"/ENABLED="true"/' /etc/default/sysstat
systemctl enable --now sysstat
# RHEL
systemctl enable --now sysstat
sar -u; sar -r; sar -d
```

</details>

---

## Challenge – Diagnose a mystery ★★★

Have a partner (or a script) run **one** of these without telling you which:

```bash
# A: CPU      for i in $(seq $(nproc)); do yes >/dev/null & done
# B: disk     dd if=/dev/zero of=/var/tmp/x bs=1M count=5000 oflag=direct &
# C: memory   python3 -c 'a=bytearray(int(0.8*__import__("os").sysconf("SC_PHYS_PAGES")*4096)); input()' &
```

Using only monitoring tools, identify the bottleneck and the responsible PID within 3 minutes. Write your reasoning (which metrics pointed where) in `/root/diagnosis.txt`.

<details>
<summary>Solution</summary>

- **A:** high load ≈ CPUs, `us` ~100%, `r` high, `wa` ~0 → `top` P.
- **B:** high load, CPU idle/high `wa`, `b` > 0, `iostat` `%util`/`await` high → `iotop -o`.
- **C:** `available` near 0, `si/so` non-zero, maybe OOM in `journalctl -k` → `top` M.

</details>

---

<!-- footer:start -->
[← Exercises: Processes](01-processes.md) · [Module exercises](README.md) · [Exercises: Troubleshooting Disk Space →](03-disk-space.md)
<!-- footer:end -->
