# Exercises – Monitoring Storage Performance

<!-- nav:start -->
> **Module 05 · Storage** — lesson 9 of 9  
> Read first: [Theory – Monitoring Storage Performance](../../theory/05-storage/09-storage-performance.md) · [Course lesson (commands)](../../course/05-Storage.md#lesson-9---storage-performance-gap) · [Module exercises](README.md)
<!-- nav:end -->

> Install: `apt install -y sysstat iotop fio` / `dnf install -y sysstat iotop fio`. Run tests on a scratch filesystem (e.g. `/data`), never on raw devices that hold data.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Read iostat ★

Run `iostat -xz 1 5` on an idle system and explain each of: `r/s`, `wkB/s`, `w_await`, `aqu-sz`, `%util`. Why ignore the first report?

<details>
<summary>Solution</summary>

See the table in the theory lesson. The first report contains averages since boot, not current activity.

</details>

---

## Exercise 2 – Sequential write load ★

In terminal 1 run `iostat -xz 1`; in terminal 2 run `dd if=/dev/zero of=/data/seq.img bs=1M count=1500 oflag=direct`. Note IOPS, throughput, request size and latency. Then remove the file.

<details>
<summary>Solution</summary>

Large `wareq-sz` (~1024 KiB), high `wkB/s`, moderate `w/s`, `%util` near 100%.

</details>

---

## Exercise 3 – Random I/O with fio ★★

Run a 30 s 4k random read test with fio on `/data`. Compare IOPS and request size with Exercise 2 in `iostat`.

<details>
<summary>Solution</summary>

```bash
fio --name=rr --filename=/data/fio.test --size=512M --rw=randread --bs=4k --iodepth=16 \
    --ioengine=libaio --direct=1 --runtime=30 --time_based --group_reporting
rm /data/fio.test
```

Many more IOPS with small `rareq-sz` (4 KiB) and much lower throughput.

</details>

---

## Exercise 4 – Find the culprit ★★

Start a hidden I/O job: `nohup bash -c 'while :; do dd if=/dev/zero of=/var/tmp/.x bs=1M count=500 oflag=direct 2>/dev/null; done' &`. Find which process does the I/O using two tools, and stop it.

<details>
<summary>Solution</summary>

```bash
iotop -o -b -n 3 | head -20
pidstat -d 1 3
pkill -f '/var/tmp/.x'; rm -f /var/tmp/.x
```

</details>

---

## Exercise 5 – ionice ★★

Run two parallel `dd` writes, one with `ionice -c3`. Compare their speed and check the scheduler (`cat /sys/block/<dev>/queue/scheduler`). Switch to `bfq` temporarily and repeat.

<details>
<summary>Solution</summary>

```bash
dd if=/dev/zero of=/data/a bs=1M count=800 oflag=direct & ionice -c3 dd if=/dev/zero of=/data/b bs=1M count=800 oflag=direct & wait
cat /sys/block/vda/queue/scheduler
echo bfq > /sys/block/vda/queue/scheduler     # if available (modprobe bfq)
```

`ionice` classes are fully honored by `bfq`; with `mq-deadline`/`none` the difference may be small.

</details>

---

## Exercise 6 – Pressure and D state ★

While Exercise 2's `dd` runs, show `/proc/pressure/io` and list processes in `D` state.

<details>
<summary>Solution</summary>

```bash
cat /proc/pressure/io
ps -eo pid,stat,cmd | awk '$2 ~ /^D/'
```

</details>

---

## Challenge – Persistent scheduler rule ★★★

Write a udev rule that sets `mq-deadline` for rotational disks and `none` for non-rotational `sd*`/`vd*` disks. Reload and trigger udev; verify.

<details>
<summary>Solution</summary>

```bash
cat > /etc/udev/rules.d/60-iosched.rules << 'EOF'
ACTION=="add|change", KERNEL=="sd[a-z]|vd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="mq-deadline"
ACTION=="add|change", KERNEL=="sd[a-z]|vd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"
EOF
udevadm control --reload && udevadm trigger --subsystem-match=block
grep . /sys/block/*/queue/scheduler
```

</details>

---

<!-- footer:start -->
[← Exercises: Filesystem Automounters](08-automounters.md) · [Module exercises](README.md)
<!-- footer:end -->
