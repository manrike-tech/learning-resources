# Exercises – User Resource Limits

<!-- nav:start -->
> **Module 02 · Users, Groups, Permissions & LDAP** — lesson 7 of 8  
> Read first: [Theory – User Resource Limits](../../theory/02-users-groups-permissions/07-resource-limits.md) · [Course lesson (commands)](../../course/02-Users-Groups-Permissions.md#lesson-7---user-resource-limits-gap) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Setup

```bash
groupadd developers
useradd -m -s /bin/bash ana; useradd -m -s /bin/bash student
usermod -aG developers ana
echo 'student:Passw0rd!' | chpasswd
```

---

## Exercise 1 – Read limits ★

1. Show all your soft limits, and the soft and hard `nofile` limits.
2. Show the limits of the running `sshd` (or `systemd-journald`) process.

<details>
<summary>Solution</summary>

```bash
ulimit -a; ulimit -Sn; ulimit -Hn
cat /proc/$(pgrep -o sshd)/limits
```

</details>

---

## Exercise 2 – Temporary limits ★

In a subshell, set the open files soft limit to 256 and verify. Then try to raise the **hard** limit above its value as a normal user.

<details>
<summary>Solution</summary>

```bash
bash -c 'ulimit -Sn 256; ulimit -Sn'
su - student -c 'ulimit -Hn $(( $(ulimit -Hn) + 1 ))'   # Operation not permitted
```

</details>

---

## Exercise 3 – Per-user persistent limit ★★

Limit `student` to a hard maximum of 50 processes.

**Verify:** `su - student -c 'ulimit -Hu'` → `50`.

<details>
<summary>Solution</summary>

```bash
echo 'student hard nproc 50' > /etc/security/limits.d/50-student.conf
```

</details>

---

## Exercise 4 – Group limits ★★

Members of `developers`: soft `nofile` 4096, hard 8192.

**Verify:** `su - ana -c 'ulimit -Sn; ulimit -Hn'` → `4096` `8192`.

<details>
<summary>Solution</summary>

```bash
cat > /etc/security/limits.d/60-developers.conf << 'EOF'
@developers soft nofile 4096
@developers hard nofile 8192
EOF
```

</details>

---

## Exercise 5 – Concurrent logins ★★

Allow `student` at most 1 concurrent login. Test with two SSH sessions.

<details>
<summary>Solution</summary>

```bash
echo 'student - maxlogins 1' >> /etc/security/limits.d/50-student.conf
ssh student@localhost      # first session: OK; keep it open
ssh student@localhost      # second: "Too many logins for 'student'."
```

</details>

---

## Exercise 6 – Survive a fork bomb ★★

With the `nproc` limit from Exercise 3 in place, as `student` start 100 background `sleep 300` processes. What happens? Is the system still responsive? Kill them afterwards.

<details>
<summary>Solution</summary>

```bash
su - student -c 'for i in $(seq 100); do sleep 300 & done; wait' 
# "fork: retry: Resource temporarily unavailable" after ~50
pkill -u student
```

</details>

---

## Exercise 7 – Limits for a service ★★

Set `LimitNOFILE=20000` for the cron service (`cron` on Ubuntu, `crond` on RHEL) with a drop-in and prove it applies to the running process. Show that `limits.conf` would not have worked.

<details>
<summary>Solution</summary>

```bash
SVC=$(systemctl list-unit-files | grep -qE '^crond\.service' && echo crond || echo cron)
mkdir -p /etc/systemd/system/$SVC.service.d
printf '[Service]\nLimitNOFILE=20000\n' > /etc/systemd/system/$SVC.service.d/limits.conf
systemctl daemon-reload && systemctl restart $SVC
systemctl show $SVC -p LimitNOFILE
grep "open files" /proc/$(systemctl show -p MainPID --value $SVC)/limits
```

Services are not started via a PAM login session, so `pam_limits` never reads `limits.conf` for them.

</details>

---

## Challenge – Cap a user's memory ★★★

Limit **all** processes of `student` together to 300 MiB of memory using systemd/cgroups. Prove it by running a memory hog as student (`python3 -c 'a = bytearray(500*1024*1024); input()'`) and finding the OOM event.

<details>
<summary>Solution</summary>

```bash
systemctl set-property user-$(id -u student).slice MemoryMax=300M
ssh student@localhost "python3 -c 'a = bytearray(500*1024*1024)'"   # Killed
journalctl -k --since "-5min" | grep -i -E "oom|memory"
systemctl show user-$(id -u student).slice -p MemoryMax
```

`set-property` writes a drop-in under `/etc/systemd/system.control/`, so it is persistent.

</details>

---

## Cleanup

```bash
rm -f /etc/security/limits.d/{50-student,60-developers}.conf
rm -rf /etc/systemd/system/cron*.service.d /etc/systemd/system.control/user-*.slice.d
systemctl daemon-reload
userdel -r ana; userdel -r student; groupdel developers
```

---

<!-- footer:start -->
[← Exercises: Personal and System-Wide Environment Profiles](06-environment-profiles.md) · [Module exercises](README.md) · [Exercises: Using LDAP for Users and Groups →](08-ldap-clients.md)
<!-- footer:end -->
