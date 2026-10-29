# Exercises – The OpenSSH Server and Client

<!-- nav:start -->
> **Module 07 · Networking** — lesson 8 of 11  
> Read first: [Theory – The OpenSSH Server and Client](../../theory/07-networking/08-openssh.md) · [Course lesson (commands)](../../course/07-Networking.md#lesson-8---openssh-server--client-gap) · [Module exercises](README.md)
<!-- nav:end -->

> Client: `deb1`. Server: `rhel1`. Keep one root session open on the server while changing `sshd` settings.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Setup

```bash
# on both VMs
useradd -m -s /bin/bash ana; echo 'ana:Passw0rd!' | chpasswd
```

---

## Exercise 1 – Host keys ★

1. On `rhel1`, show the fingerprint of the ED25519 host key.
2. On `deb1`, connect as `ana` for the first time and compare the fingerprint before accepting.
3. Find the stored entry in `~/.ssh/known_hosts`.

<details>
<summary>Solution</summary>

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub         # rhel1
ssh ana@rhel1                                            # deb1 – compare, then yes
ssh-keygen -F rhel1
```

</details>

---

## Exercise 2 – Key-based login ★

As `ana` on `deb1`: create an ed25519 key with a passphrase, install it on `rhel1`, load it in an agent and log in without typing the password.

<details>
<summary>Solution</summary>

```bash
ssh-keygen -t ed25519 -C "ana@deb1"
ssh-copy-id ana@rhel1
eval "$(ssh-agent)"; ssh-add
ssh ana@rhel1 hostname
```

</details>

---

## Exercise 3 – Broken permissions ★★

On `rhel1`, run `chmod 777 /home/ana` and try logging in with the key. Read the server log to find the reason. Fix it.

<details>
<summary>Solution</summary>

`journalctl -u sshd -n 20` → `Authentication refused: bad ownership or modes for directory /home/ana`. Fix: `chmod 755 /home/ana` (and `700 ~/.ssh`, `600 authorized_keys`).

</details>

---

## Exercise 4 – Harden the server ★★

> Key-only login, root login disabled.

Use a drop-in. Prove with `sshd -T`, and test that password login is refused (`ssh -o PubkeyAuthentication=no ana@rhel1`) while key login works.

<details>
<summary>Solution</summary>

```bash
cat > /etc/ssh/sshd_config.d/50-hardening.conf << 'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
EOF
grep -r PasswordAuthentication /etc/ssh/sshd_config.d/     # check for conflicting earlier drop-ins
sshd -t && systemctl reload sshd
sshd -T | grep -E 'permitrootlogin|passwordauthentication'
```

</details>

---

## Exercise 5 – Restrict users ★★

Allow SSH only for members of group `sshusers`; add `ana` to it. Verify another user is rejected (and see the log message).

<details>
<summary>Solution</summary>

```bash
groupadd sshusers; usermod -aG sshusers ana
echo "AllowGroups sshusers" > /etc/ssh/sshd_config.d/60-allow.conf
sshd -t && systemctl reload sshd
journalctl -u sshd | grep "not allowed because none of user's groups"
```

</details>

---

## Exercise 6 – Change the port (RHEL) ★★★

> SSH must listen on port 2222 (and no longer on 22). Handle SELinux and the firewall.

<details>
<summary>Solution</summary>

```bash
echo "Port 2222" > /etc/ssh/sshd_config.d/10-port.conf
semanage port -a -t ssh_port_t -p tcp 2222
firewall-cmd --permanent --add-port=2222/tcp
firewall-cmd --permanent --remove-service=ssh
sshd -t && systemctl restart sshd
firewall-cmd --reload
ss -tlnp | grep sshd
# from deb1 (new terminal!): ssh -p 2222 ana@rhel1
```

(On Ubuntu 24.04 with socket activation: `systemctl daemon-reload && systemctl restart ssh.socket`.)

</details>

---

## Exercise 7 – Client config ★

On `deb1`, create a `~/.ssh/config` entry `lab` for `rhel1` (user `ana`, port 2222, your key). Use it with `ssh`, `scp` and `rsync`.

<details>
<summary>Solution</summary>

```text
Host lab
    HostName 192.168.56.20
    User ana
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
```

```bash
chmod 600 ~/.ssh/config
ssh lab uptime; scp /etc/hosts lab:/tmp/; rsync -av ~/.bashrc lab:/tmp/
```

</details>

---

## Exercise 8 – Local port forwarding ★★

On `rhel1`, run `python3 -m http.server 8000 --bind 127.0.0.1`. From `deb1`, reach it through an SSH tunnel on local port 9000.

<details>
<summary>Solution</summary>

```bash
ssh -fN -L 9000:localhost:8000 lab
curl http://localhost:9000/
```

</details>

---

## Exercise 9 – Restricted key ★★★

Create a second key on `deb1` that may only run `uptime` on `rhel1`, only from `deb1`'s IP, with no port forwarding.

<details>
<summary>Solution</summary>

```bash
ssh-keygen -t ed25519 -f ~/.ssh/uptime_key -N ''
# on rhel1, append to ~ana/.ssh/authorized_keys:
from="192.168.56.10",command="/usr/bin/uptime",no-port-forwarding,no-pty ssh-ed25519 AAAA… ana@deb1
ssh -i ~/.ssh/uptime_key lab whoami     # prints uptime instead
```

</details>

---

## Challenge – Jump host ★★★

Treat `rhel1` as a bastion: reach `deb1`'s SSH **through** `rhel1` from your workstation (or from `deb1` to itself via `rhel1`) using `-J`, then make it permanent with `ProxyJump` in `~/.ssh/config`.

<details>
<summary>Solution</summary>

```bash
ssh -J ana@rhel1:2222 ana@192.168.56.10
```

```text
Host deb1-via-bastion
    HostName 192.168.56.10
    User ana
    ProxyJump lab
```

</details>

---

## Cleanup

```bash
rm -f /etc/ssh/sshd_config.d/{10-port,50-hardening,60-allow}.conf
semanage port -d -t ssh_port_t -p tcp 2222
firewall-cmd --permanent --add-service=ssh --remove-port=2222/tcp && firewall-cmd --reload
sshd -t && systemctl restart sshd
```

---

<!-- footer:start -->
[← Exercises: Packet Filtering, Port Redirection and NAT](07-firewalls-nat.md) · [Module exercises](README.md) · [Exercises: Setting and Synchronizing System Time →](09-time-sync.md)
<!-- footer:end -->
