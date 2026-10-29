# User Resource Limits

<!-- nav:start -->
> **Module 02 · Users, Groups, Permissions & LDAP** — lesson 7 of 8  
> [Module index](README.md) · [Course lesson (commands)](../../course/02-Users-Groups-Permissions.md#lesson-7---user-resource-limits-gap) · [Exercises for this lesson](../../exercises/02-users-groups-permissions/07-resource-limits.md)
<!-- nav:end -->

A single user – or a single runaway program – can exhaust a whole server: a fork bomb creates processes until the system freezes, a buggy service opens files until it hits "Too many open files". Linux lets you set **resource limits** per process, per user and per group. In this lesson you will learn how limits work, how to set them temporarily with `ulimit`, permanently with `limits.conf`, and for services with systemd.

## What you will learn

- What resource limits are and which ones matter most
- **Soft** vs **hard** limits
- Viewing and changing limits with `ulimit`
- Persistent per-user and per-group limits with `/etc/security/limits.conf` and `limits.d/`
- How `pam_limits` applies them – and why they do **not** apply to systemd services
- Limits for services (`LimitNOFILE=`) and cgroup-based limits (`MemoryMax=`, `CPUQuota=`)

---

## Key terms

| Term | Definition |
|------|-----------|
| **rlimit** | A per-process resource limit enforced by the kernel (`setrlimit` system call). Inherited by child processes. |
| **Soft limit** | The value currently enforced. A user may raise it up to the hard limit. |
| **Hard limit** | The ceiling for the soft limit. Only root can raise it; anyone can lower it. |
| **`ulimit`** | Bash builtin that shows/sets limits for the current shell and its children. |
| **`pam_limits`** | PAM module that applies `/etc/security/limits.conf` when a session starts (login, SSH, `su`). |
| **cgroup** | Kernel *control group*: limits resources for a **group** of processes (used by systemd). |

---

## The most important limits

| `ulimit` flag | limits.conf item | Limits | Typical problem |
|---------------|------------------|--------|-----------------|
| `-n` | `nofile` | open file descriptors per process | "Too many open files" in web/database servers |
| `-u` | `nproc` | processes per **user** | fork bombs |
| `-c` | `core` | core dump size | disk filled with core files |
| `-f` | `fsize` | maximum file size a process can write | |
| `-v` | `as` | virtual memory (address space) | |
| `-s` | `stack` | stack size | |
| `-t` | `cpu` | CPU time in seconds | |
| – | `maxlogins` | concurrent logins for a user | |
| – | `priority` | default nice value | |

---

## Viewing limits

```console
$ ulimit -a                       # soft limits of this shell
real-time non-blocking time  (microseconds, -R) unlimited
core file size              (blocks, -c) 0
file size                   (blocks, -f) unlimited
max user processes                  (-u) 7645
open files                          (-n) 1024
...
$ ulimit -Sn; ulimit -Hn          # soft and hard limit for open files
1024
524288
$ cat /proc/$(pgrep -o nginx)/limits     # limits of a running process
Limit                     Soft Limit  Hard Limit  Units
Max open files            1024        524288      files
Max processes             7645        7645        processes
```

`/proc/<PID>/limits` is the source of truth for any running program.

---

## Changing limits temporarily with ulimit

```console
$ ulimit -n 4096          # set soft (and hard, if allowed) for this shell
$ ulimit -Sn 2048         # soft only
$ ulimit -Hn 2048         # lower the hard limit (irreversible for this shell unless root)
$ ulimit -u 100           # max 100 processes
$ ulimit -c unlimited     # allow core dumps
```

These changes affect only the current shell and the programs started from it. They disappear when the shell exits.

```console
$ ulimit -Hn 2048
$ ulimit -n 4096
bash: ulimit: open files: cannot modify limit: Operation not permitted
```

---

## Persistent limits: /etc/security/limits.conf

Format: `<domain> <type> <item> <value>`

```text
# /etc/security/limits.d/90-company.conf
# domain     type   item        value
alice        hard   nproc       50
@developers  soft   nofile      4096
@developers  hard   nofile      8192
*            hard   core        0
*            -      maxlogins   3
@students    hard   cpu         60
```

| Field | Values |
|-------|--------|
| domain | a user name, `@group`, `*` (everyone except root), `%group` (maxlogins), UID ranges `1000:` |
| type | `soft`, `hard`, or `-` (both) |
| item | `nofile`, `nproc`, `core`, `fsize`, `as`, `cpu`, `maxlogins`, `priority`, `memlock`… |
| value | a number or `unlimited` |

- Put your rules in a file under **`/etc/security/limits.d/`** (`*.conf`) rather than editing the main file.
- `*` does **not** apply to root; write `root` explicitly to limit root.
- Specific user entries override group entries, which override `*`.

### When are they applied?

`pam_limits` applies them when a **new session** starts: console login, SSH, `su -`, `sudo -i`, cron jobs (via `pam_limits` in `/etc/pam.d/cron`/`crond`). Existing sessions don't change.

```console
$ su - alice -c 'ulimit -Hu'
50
```

Check that the PAM module is enabled (it is by default):

```console
$ grep pam_limits /etc/pam.d/common-session /etc/pam.d/system-auth 2>/dev/null
/etc/pam.d/common-session:session required pam_limits.so
```

---

## Limits for systemd services

Services started by systemd do **not** go through a PAM login, so **`limits.conf` does not apply to them**. Set limits in the unit instead:

```console
$ sudo systemctl edit nginx
```

```ini
[Service]
LimitNOFILE=65536
LimitNPROC=4096
LimitCORE=0
```

```console
$ sudo systemctl restart nginx
$ systemctl show nginx -p LimitNOFILE
LimitNOFILE=65536
$ cat /proc/$(pgrep -o nginx)/limits | grep "open files"
Max open files            65536       65536       files
```

Global defaults for all services: `DefaultLimitNOFILE=` in `/etc/systemd/system.conf`.

---

## cgroup limits: limiting groups of processes

rlimits are **per process** (except `nproc`). To cap the total memory or CPU of a whole service or user session, systemd uses **cgroups**:

```ini
# in a service unit or drop-in
[Service]
MemoryMax=512M        # hard memory cap (OOM-killed above it)
MemoryHigh=400M       # throttle above this
CPUQuota=50%          # at most half of one CPU
TasksMax=200          # max processes/threads
IOWeight=100
```

Limit **all processes of a user** (every login of UID 1001):

```console
$ sudo systemctl set-property user-1001.slice MemoryMax=1G CPUQuota=100%
$ systemctl status user-1001.slice
```

Check usage live: `systemd-cgtop`.

| Mechanism | Scope | Configured in | Applies to |
|-----------|-------|---------------|-----------|
| `ulimit` | one shell + children | command line | temporary tests |
| `limits.conf` | users / groups at login | `/etc/security/limits.d/` | interactive & SSH sessions, cron |
| `Limit*=` | one service's processes | unit file / drop-in | systemd services |
| `MemoryMax=`, `CPUQuota=`, `TasksMax=` | a cgroup (service, slice, user) | unit / `systemctl set-property` | totals for a group of processes |

---

## Example: stop a fork bomb

```console
$ echo 'student hard nproc 100' | sudo tee /etc/security/limits.d/50-student.conf
$ su - student
student$ ulimit -u
100
student$ for i in $(seq 200); do sleep 100 & done
-bash: fork: retry: Resource temporarily unavailable
```

The system stays responsive; only `student` is affected.

---

## Try it yourself

1. Check your soft and hard `nofile` limits; lower the soft one in a subshell and verify with `ulimit -n`.
2. Limit user `alice` to 2 concurrent logins and test with two SSH sessions.
3. Give group `developers` a soft `nofile` of 4096 and hard of 8192; verify with `su - <member> -c 'ulimit -Sn; ulimit -Hn'`.
4. Set `LimitNOFILE=10000` for `cron`/`crond` with a drop-in and verify in `/proc/<pid>/limits`.

---

## Common mistakes

- Expecting `limits.conf` to affect systemd services.
- Expecting new limits to apply to an already open session.
- Forgetting that `*` doesn't include root.
- Setting a soft limit higher than the hard limit (ignored / error).
- Typing `nofiles` or `noproc` (invalid item names are silently ignored – check `man limits.conf`).

---

## Summary

- Soft = enforced, hard = ceiling. Users may raise soft up to hard.
- Temporary: `ulimit`. Persistent for users/groups: `/etc/security/limits.d/*.conf` (applied by `pam_limits` at login). Services: `LimitNOFILE=` etc. in a drop-in. Totals: cgroups (`MemoryMax=`, `CPUQuota=`, `TasksMax=`).
- Verify with `ulimit -a`, `/proc/PID/limits`, `systemctl show -p`.

## Check yourself

1. A user runs `ulimit -n 100000` and gets "Operation not permitted". Why?
2. You set `@web hard nofile 65536` but nginx still has 1024. Why, and how do you fix it?
3. Which limit stops a fork bomb?

<details>
<summary>Answers</summary>

1. The requested value exceeds the hard limit; only root can raise a hard limit.
2. nginx is started by systemd, not through a PAM login – set `LimitNOFILE=65536` in a drop-in and restart.
3. `nproc` (max user processes), or `TasksMax=` for a cgroup.

</details>

## Further reading

- `man 5 limits.conf`, `man 8 pam_limits`, `help ulimit`, `man 2 getrlimit`
- `man 5 systemd.exec` (`LimitNOFILE=`) · `man 5 systemd.resource-control` (`MemoryMax=`)
- [ArchWiki: Limits.conf](https://wiki.archlinux.org/title/Limits.conf)
- [Red Hat: Managing, monitoring and updating the kernel (RHEL 9) – see the control groups chapters](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_monitoring_and_updating_the_kernel/index)

---

<!-- footer:start -->
[← Personal and System-Wide Environment Profiles](06-environment-profiles.md) · [Module index](README.md) · [Using LDAP for Users and Groups →](08-ldap-clients.md)  
**Practice now:** [Exercises – User Resource Limits](../../exercises/02-users-groups-permissions/07-resource-limits.md)
<!-- footer:end -->
