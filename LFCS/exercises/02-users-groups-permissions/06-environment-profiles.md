# Exercises – Environment Profiles

<!-- nav:start -->
> **Module 02 · Users, Groups, Permissions & LDAP** — lesson 6 of 8  
> Read first: [Theory – Personal and System-Wide Environment Profiles](../../theory/02-users-groups-permissions/06-environment-profiles.md) · [Course lesson (commands)](../../course/02-Users-Groups-Permissions.md#lesson-6---environment-profiles-gap) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Setup

```bash
groupadd developers
useradd -m -s /bin/bash ana; useradd -m -s /bin/bash luis
usermod -aG developers ana
```

---

## Exercise 1 – System-wide variable ★

Make `COMPANY=acme` and `HISTTIMEFORMAT="%F %T "` available in every user's login shell. Do **not** edit `/etc/profile`.

**Verify:** `su - luis -c 'echo $COMPANY'` → `acme`.

<details>
<summary>Solution</summary>

```bash
cat > /etc/profile.d/company.sh << 'EOF'
export COMPANY=acme
export HISTTIMEFORMAT="%F %T "
EOF
bash -n /etc/profile.d/company.sh
```

</details>

---

## Exercise 2 – System-wide alias ★

Make alias `ports='ss -tulpn'` available in every **interactive** bash shell (also non-login).

**Verify:** `su luis -c 'bash -ic "type ports"'`.

<details>
<summary>Solution</summary>

```bash
echo "alias ports='ss -tulpn'" >> /etc/bash.bashrc   # Debian/Ubuntu
echo "alias ports='ss -tulpn'" >> /etc/bashrc        # RHEL
```

(Alternatively a file in `/etc/profile.d/`, which RHEL's `/etc/bashrc` also sources for interactive non-login shells.)

</details>

---

## Exercise 3 – Personal settings ★

For `ana` only: `EDITOR=nano` and alias `ll='ls -alF'`.

**Verify:** `su - ana -c 'echo $EDITOR; alias ll'`; `su - luis -c 'echo $EDITOR'` is unchanged.

<details>
<summary>Solution</summary>

```bash
cat >> /home/ana/.bashrc << 'EOF'
export EDITOR=nano
alias ll='ls -alF'
EOF
```

</details>

---

## Exercise 4 – PATH for everyone ★★

Install a tool script `/opt/tools/bin/hello` (prints `hello`) and make it runnable by name for every user, without breaking the existing PATH.

**Verify:** `su - luis -c 'hello'`.

<details>
<summary>Solution</summary>

```bash
mkdir -p /opt/tools/bin
printf '#!/bin/sh\necho hello\n' > /opt/tools/bin/hello; chmod 755 /opt/tools/bin/hello
echo 'export PATH="$PATH:/opt/tools/bin"' > /etc/profile.d/tools.sh
```

</details>

---

## Exercise 5 – Group-specific settings ★★

Only members of `developers` should get `umask 002` and `JAVA_OPTS="-Xmx512m"` at login.

**Verify:** `su - ana -c 'umask; echo $JAVA_OPTS'` → `0002 -Xmx512m`; luis gets the default umask and nothing.

<details>
<summary>Solution</summary>

```bash
cat > /etc/profile.d/developers.sh << 'EOF'
if id -nG | grep -qw developers; then
    umask 002
    export JAVA_OPTS="-Xmx512m"
fi
EOF
```

</details>

---

## Exercise 6 – /etc/environment ★

Set `TZ=UTC` in `/etc/environment`. Test with an SSH command (`ssh luis@localhost 'echo $TZ'`) and with `su - luis`.

<details>
<summary>Solution</summary>

```bash
echo 'TZ=UTC' >> /etc/environment
```

PAM (`pam_env`) applies it to sessions regardless of the shell.

</details>

---

## Exercise 7 – Future users ★★

New users (only) must get `~/.vimrc` with `set number` and a `~/projects` directory. Create `marta` to test; show that `ana` is unaffected.

<details>
<summary>Solution</summary>

```bash
echo 'set number' > /etc/skel/.vimrc
mkdir /etc/skel/projects
useradd -m marta; ls -a /home/marta; ls -a /home/ana
```

</details>

---

## Challenge – Idle root shells ★★★

Make **root's** interactive shells log out automatically after 10 minutes idle, in a way root can't simply `unset` in that session, without affecting other users.

<details>
<summary>Solution</summary>

```bash
cat > /etc/profile.d/tmout.sh << 'EOF'
if [ "$(id -u)" -eq 0 ]; then
    TMOUT=600
    readonly TMOUT
    export TMOUT
fi
EOF
```

Test: `sudo -i`, then `unset TMOUT` → `bash: unset: TMOUT: cannot unset: readonly variable`.

</details>

---

## Cleanup

```bash
rm -f /etc/profile.d/{company,tools,developers,tmout}.sh /etc/skel/.vimrc; rmdir /etc/skel/projects
sed -i '/ports=/d' /etc/bash.bashrc /etc/bashrc 2>/dev/null; sed -i '/^TZ=UTC/d' /etc/environment
for u in ana luis marta; do userdel -r $u; done; groupdel developers
```

---

<!-- footer:start -->
[← Exercises: Access Control Lists (ACLs)](05-acls.md) · [Module exercises](README.md) · [Exercises: User Resource Limits →](07-resource-limits.md)
<!-- footer:end -->
