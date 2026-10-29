# Exercises – rpm and dnf

<!-- nav:start -->
> **Module 03 · Package Management & Git** — lesson 3 of 5  
> Read first: [Theory – RPM Packages: rpm and dnf](../../theory/03-packages-git/03-rpm-dnf.md) · [Course lesson (commands)](../../course/03-Packages-and-Git.md#lesson-3---rpm-dnfyum-ch-1011) · [Module exercises](README.md)
<!-- nav:end -->

> Run on `rhel1` (Rocky/Alma/RHEL).

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Provider and install ★

Find which package provides `/usr/bin/dig`, install it, and save its documentation file list to `/opt/dig-docs.txt`.

<details>
<summary>Solution</summary>

```bash
dnf provides /usr/bin/dig
dnf install -y bind-utils
rpm -qd bind-utils > /opt/dig-docs.txt
```

</details>

---

## Exercise 2 – Ownership and config files ★

1. Which package owns `/etc/ssh/sshd_config`?
2. List all config files of that package.
3. Find which package provides the `semanage` command (not installed) using a wildcard.

<details>
<summary>Solution</summary>

```bash
rpm -qf /etc/ssh/sshd_config
rpm -qc openssh-server
dnf provides '*/semanage'
```

</details>

---

## Exercise 3 – Verify integrity ★★

1. Append a comment to `/etc/chrony.conf` and change the mode of `/usr/bin/chronyc` to `0700`.
2. Detect both changes with `rpm -V` and interpret each column.
3. Repair the binary's mode using rpm (`rpm --setperms`) or by reinstalling.

<details>
<summary>Solution</summary>

```bash
echo "# test" >> /etc/chrony.conf; chmod 700 /usr/bin/chronyc
rpm -V chrony
# S.5....T.  c /etc/chrony.conf   → size, checksum, time changed; c = config file
# .M.......    /usr/bin/chronyc   → mode changed
rpm --setperms chrony      # or: dnf reinstall -y chrony
rpm -V chrony
```

</details>

---

## Exercise 4 – Repositories ★★

1. List enabled and all repositories.
2. Create `/etc/yum.repos.d/lab.repo` defining repo `lab` with `baseurl=https://repo.example.lab/el9/`, `enabled=0`, `gpgcheck=1`.
3. Show it appears only in `dnf repolist all` as disabled.
4. Enable CRB and install EPEL; verify `epel` is in the enabled list.

<details>
<summary>Solution</summary>

```bash
dnf repolist; dnf repolist all
cat > /etc/yum.repos.d/lab.repo << 'EOF'
[lab]
name=Lab repository
baseurl=https://repo.example.lab/el9/
enabled=0
gpgcheck=1
gpgkey=https://repo.example.lab/RPM-GPG-KEY-lab
EOF
dnf repolist all | grep lab
dnf config-manager --set-enabled crb
dnf install -y epel-release && dnf repolist
```

</details>

---

## Exercise 5 – History and undo ★★

Install `tree` and `tmux` in one command. Find the transaction ID, look at its details, and undo it.

<details>
<summary>Solution</summary>

```bash
dnf install -y tree tmux
dnf history | head -3
dnf history info last
dnf history undo last -y
rpm -q tree tmux
```

</details>

---

## Exercise 6 – Local ISO repository ★★★

Mount the installation ISO (attach it to the VM as a CD, usually `/dev/sr0`) and create a repository file that uses the `BaseOS` and `AppStream` directories on it. Disable the network repos temporarily and install `tree` from the ISO.

<details>
<summary>Solution</summary>

```bash
mkdir -p /mnt/iso && mount -o ro /dev/sr0 /mnt/iso
cat > /etc/yum.repos.d/iso.repo << 'EOF'
[iso-baseos]
name=ISO BaseOS
baseurl=file:///mnt/iso/BaseOS
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9
[iso-appstream]
name=ISO AppStream
baseurl=file:///mnt/iso/AppStream
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9
EOF
dnf --disablerepo='*' --enablerepo='iso-*' install -y tree
```

(Use the key file that exists in `/etc/pki/rpm-gpg/` for your distribution.)

</details>

---

## Exercise 7 – Groups and version lock ★★

1. Show the packages in the group "Development Tools" without installing it.
2. Lock the current version of `openssh-server` and confirm `dnf upgrade` would skip it.

<details>
<summary>Solution</summary>

```bash
dnf group info "Development Tools"
dnf install -y 'dnf-command(versionlock)'
dnf versionlock add openssh-server
dnf versionlock list
dnf versionlock delete openssh-server
```

</details>

---

## Challenge – Who changed this binary? ★★★

Run `rpm -Va` and save every line reporting a **checksum** change (`5` in column 3) on a **non-config** file to `/root/modified-binaries.txt`. Explain how you'd restore those files.

<details>
<summary>Solution</summary>

```bash
rpm -Va 2>/dev/null | awk 'substr($1,3,1)=="5" && $2!="c"' > /root/modified-binaries.txt
# restore: for each file, find its package and reinstall
awk '{print $NF}' /root/modified-binaries.txt | xargs -r rpm -qf | sort -u | xargs -r dnf reinstall -y
```

</details>

---

## Cleanup

```bash
rm -f /etc/yum.repos.d/lab.repo /etc/yum.repos.d/iso.repo; umount /mnt/iso 2>/dev/null
```

---

<!-- footer:start -->
[← Exercises: Debian Packages: dpkg and APT](02-dpkg-apt.md) · [Module exercises](README.md) · [Exercises: SUSE Packages: zypper →](04-zypper.md)
<!-- footer:end -->
