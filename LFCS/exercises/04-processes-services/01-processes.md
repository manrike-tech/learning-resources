# Exercises – Processes

<!-- nav:start -->
> **Module 04 · Processes, Services, Scheduling & Performance** — lesson 1 of 6  
> Read first: [Theory – Processes](../../theory/04-processes-services/01-processes.md) · [Course lesson (commands)](../../course/04-Processes-Services-Performance.md#lesson-1---processes-ch-14) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Your own process ★

Find the PID of your current shell, its parent's PID and the parent's command name. Show the chain up to PID 1 with `pstree`.

<details>
<summary>Solution</summary>

```bash
echo $$
ps -o ppid= -p $$
ps -o comm= -p "$(ps -o ppid= -p $$)"
pstree -ps $$
```

</details>

---

## Exercise 2 – Listing and sorting ★

1. Save the 3 processes using the most memory (PID, user, RSS, command) to `/root/topmem.txt`.
2. List all processes of user `root` that have no terminal.

<details>
<summary>Solution</summary>

```bash
ps -eo pid,user,rss,cmd --sort=-rss | head -4 > /root/topmem.txt
ps -U root -u root -o pid,tty,cmd | awk '$2=="?"'
```

</details>

---

## Exercise 3 – Signals ★★

1. Start `sleep 9999 &`. Terminate it gracefully, confirm it's gone.
2. Write a script that traps `SIGTERM` (`trap 'echo got TERM' TERM; while :; do sleep 1; done`), start it in the background, send `SIGTERM` – what happens? Then stop it for good.
3. Use signal 0 to test whether PID 1 exists.

<details>
<summary>Solution</summary>

```bash
sleep 9999 & kill $!; ps -p $! || echo gone
cat > /tmp/trap.sh << 'EOF'
#!/bin/bash
trap 'echo got TERM' TERM
while :; do sleep 1; done
EOF
chmod +x /tmp/trap.sh; /tmp/trap.sh & P=$!
kill $P          # prints "got TERM", keeps running
kill -9 $P       # SIGKILL can't be trapped
sudo kill -0 1 && echo "PID 1 exists"
```

</details>

---

## Exercise 4 – nice and renice ★★

1. Start `sleep 9999` with nice value 15.
2. As root, change it to 5. As a normal user, try changing your own niced process from 15 to 10.
3. Start two `yes > /dev/null` processes on a single CPU (`taskset -c 0`), renice one to 19, and compare their CPU usage in `top`.

<details>
<summary>Solution</summary>

```bash
nice -n 15 sleep 9999 & P=$!
ps -o pid,ni,cmd -p $P
sudo renice -n 5 -p $P
su - someuser -c 'nice -n 15 sleep 999 & sleep 1; renice -n 10 -p $!'   # Permission denied
taskset -c 0 yes > /dev/null & A=$!; taskset -c 0 yes > /dev/null & B=$!
renice -n 19 -p $B; top -b -n 1 -p $A,$B | tail -3
kill $A $B $P
```

</details>

---

## Exercise 5 – Job control ★

1. Start `sleep 300`, suspend it with Ctrl+Z, check its state in `ps` (T), resume it in the background, bring it to the foreground and end it with Ctrl+C.
2. Start a job that survives logout and write its output to `/tmp/longjob.log`.

<details>
<summary>Solution</summary>

```bash
sleep 300      # Ctrl+Z
ps -o pid,stat,cmd -C sleep
bg %1; jobs; fg %1   # Ctrl+C
nohup bash -c 'for i in $(seq 100); do date; sleep 5; done' > /tmp/longjob.log 2>&1 &
```

</details>

---

## Exercise 6 – Zombies ★★★

Create a zombie on purpose, find it, and get rid of it.

```bash
python3 -c 'import os,time
pid=os.fork()
if pid==0: os._exit(0)
time.sleep(600)' &
```

<details>
<summary>Solution</summary>

```bash
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /Z/'
kill -9 <zombie_pid>        # no effect – it's already dead
kill <ppid>                 # kill the parent (python3); PID 1 reaps the zombie
ps -eo stat | grep -c Z
```

</details>

---

## Exercise 7 – Inspect via /proc ★★

For the `sshd` main process, find from `/proc` only: its current working directory, the executable path, the number of open file descriptors and its RSS.

<details>
<summary>Solution</summary>

```bash
P=$(pgrep -o sshd)
sudo ls -l /proc/$P/cwd /proc/$P/exe
sudo ls /proc/$P/fd | wc -l
grep VmRSS /proc/$P/status
```

</details>

---

## Challenge – Exam-style ★★★

> A process is consuming 100% CPU. Find it and lower its priority to 19 **without killing it**. Then save its PID, user and full command line to `/root/cpuhog.txt`.

Setup: `su - someuser -c 'nohup sh -c "while :; do :; done" >/dev/null 2>&1 &'`

<details>
<summary>Solution</summary>

```bash
P=$(ps -eo pid,%cpu --sort=-%cpu --no-headers | head -1 | awk '{print $1}')
sudo renice -n 19 -p $P
ps -o pid,user,ni,args -p $P | tee /root/cpuhog.txt
```

</details>

---

<!-- footer:start -->
[Module exercises](README.md) · [Exercises: Monitoring System Performance →](02-monitoring.md)
<!-- footer:end -->
