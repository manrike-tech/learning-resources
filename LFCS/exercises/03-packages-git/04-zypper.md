# Exercises – zypper

<!-- nav:start -->
> **Module 03 · Package Management & Git** — lesson 4 of 5  
> Read first: [Theory – SUSE Packages: zypper](../../theory/03-packages-git/04-zypper.md) · [Course lesson (commands)](../../course/03-Packages-and-Git.md#lesson-4---zypper-ch-12) · [Module exercises](README.md)
<!-- nav:end -->

> You need openSUSE. The quickest option is a container on either VM:
> `podman run -it --rm registry.opensuse.org/opensuse/leap:15.6 bash` (or `docker run …`).

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Repositories ★

List repositories with their URIs and refresh them.

<details>
<summary>Solution</summary>

```bash
zypper lr -u
zypper refresh
```

</details>

---

## Exercise 2 – Search, install, query ★

1. Search for `tree`, show its info, install it non-interactively.
2. List its files with rpm.

<details>
<summary>Solution</summary>

```bash
zypper se tree; zypper if tree
zypper -n in tree
rpm -ql tree
```

</details>

---

## Exercise 3 – Provider of a file ★★

Find which package provides `/usr/bin/dig` and install it. Confirm with `rpm -qf`.

<details>
<summary>Solution</summary>

```bash
zypper wp /usr/bin/dig
zypper -n in bind-utils
rpm -qf /usr/bin/dig
```

</details>

---

## Exercise 4 – Locks ★★

Lock `tree`, try to remove it, list locks, and remove the lock.

<details>
<summary>Solution</summary>

```bash
zypper al tree
zypper -n rm tree        # refuses / proposes to break the lock
zypper ll
zypper rl tree
```

</details>

---

## Exercise 5 – Add and disable a repository ★★

Add the openSUSE "Leap 15.6 update" repository under alias `myupdates` (or any valid URL), disable it, then remove it.

<details>
<summary>Solution</summary>

```bash
zypper ar https://download.opensuse.org/update/leap/15.6/oss/ myupdates
zypper mr -d myupdates
zypper lr
zypper rr myupdates
```

</details>

---

## Challenge – Translate ★★★

Write a table in `/root/pkg-cheatsheet.md` with the apt, dnf and zypper commands for: refresh, install, remove, search, info, provides, list repos, add repo, lock, list updates. Check each zypper command actually works in the container.

<details>
<summary>Solution</summary>

See the command map in the [theory lesson](../../theory/03-packages-git/04-zypper.md#command-map-apt-vs-dnf-vs-zypper).

</details>

---

<!-- footer:start -->
[← Exercises: RPM Packages: rpm and dnf](03-rpm-dnf.md) · [Module exercises](README.md) · [Exercises: Git Fundamentals →](05-git-fundamentals.md)
<!-- footer:end -->
