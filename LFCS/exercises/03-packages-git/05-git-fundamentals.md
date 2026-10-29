# Exercises – Git Fundamentals

<!-- nav:start -->
> **Module 03 · Package Management & Git** — lesson 5 of 5  
> Read first: [Theory – Git Fundamentals](../../theory/03-packages-git/05-git-fundamentals.md) · [Course lesson (commands)](../../course/03-Packages-and-Git.md#lesson-5---git-fundamentals-ch-13) · [Module exercises](README.md)
<!-- nav:end -->

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Setup

```bash
dnf install -y git || apt install -y git
git config --global user.name "Lab User"
git config --global user.email lab@example.lab
git config --global init.defaultBranch main
```

---

## Exercise 1 – First repository ★

1. Create a repository in `~/configs`.
2. Add a file `motd.txt` with one line and commit it with message `Add motd`.
3. Show the history in one line per commit.

**Verify:** `git log --oneline` shows one commit `Add motd`.

<details>
<summary>Solution</summary>

```bash
git init ~/configs && cd ~/configs
echo "Welcome" > motd.txt
git add motd.txt && git commit -m "Add motd"
git log --oneline
```

</details>

---

## Exercise 2 – The three areas ★

1. Modify `motd.txt` and create `hosts.txt`. Run `git status`.
2. Stage only `motd.txt`. Show `git diff` and `git diff --staged`.
3. Commit, then add and commit `hosts.txt`.

<details>
<summary>Solution</summary>

```bash
echo "Authorized use only" >> motd.txt; echo "10.0.0.1 gw" > hosts.txt
git status
git add motd.txt
git diff            # nothing for motd.txt (staged); hosts.txt is untracked so not shown
git diff --staged   # the motd change
git commit -m "Update motd"
git add hosts.txt && git commit -m "Add hosts"
```

</details>

---

## Exercise 3 – Undo ★★

1. Break `motd.txt` (overwrite it), then restore it from the last commit.
2. Stage a change and unstage it.
3. Commit a bad change, then reverse it with a new commit.

<details>
<summary>Solution</summary>

```bash
echo "oops" > motd.txt; git restore motd.txt
echo "x" >> hosts.txt; git add hosts.txt; git restore --staged hosts.txt; git restore hosts.txt
echo "BAD" >> motd.txt; git commit -am "Bad change"; git revert --no-edit HEAD
git log --oneline
```

</details>

---

## Exercise 4 – Branch and merge (fast-forward) ★

Create branch `feature-ntp`, add `ntp.conf`, commit, merge into `main`. Was it a fast-forward? Delete the branch.

<details>
<summary>Solution</summary>

```bash
git switch -c feature-ntp
echo "server pool.ntp.org iburst" > ntp.conf
git add ntp.conf && git commit -m "Add NTP config"
git switch main && git merge feature-ntp     # "Fast-forward"
git branch -d feature-ntp
```

</details>

---

## Exercise 5 – Merge conflict ★★

1. On branch `change-a`, change line 1 of `motd.txt` to `Welcome to A`; commit.
2. On `main`, change the same line to `Welcome to MAIN`; commit.
3. Merge `change-a` into `main`, resolve the conflict keeping `Welcome to the lab`, finish the merge.

**Verify:** `git log --oneline --graph` shows a merge commit with two parents.

<details>
<summary>Solution</summary>

```bash
git switch -c change-a; sed -i '1s/.*/Welcome to A/' motd.txt; git commit -am "A"
git switch main; sed -i '1s/.*/Welcome to MAIN/' motd.txt; git commit -am "Main"
git merge change-a            # CONFLICT
# edit motd.txt: remove markers, line 1 = "Welcome to the lab"
sed -i '/^<<<<<<<\|^=======\|^>>>>>>>/d; /Welcome to A\|Welcome to MAIN/d' motd.txt
sed -i '1i Welcome to the lab' motd.txt
git add motd.txt && git commit --no-edit
git log --oneline --graph
```

</details>

---

## Exercise 6 – Central bare repository ★★

1. Create a bare repo `/srv/git/app.git` owned by a group `developers` so its members can push.
2. Clone it as user `ana` into `~/app`, add `README.md`, commit and push `main`.
3. As user `luis`, clone, pull the change, add a commit and push.
4. As `ana`, pull and see luis's commit.

<details>
<summary>Solution</summary>

```bash
groupadd developers; for u in ana luis; do useradd -m -G developers $u; done
git init --bare --shared=group /srv/git/app.git
chgrp -R developers /srv/git/app.git
for u in ana luis; do su - $u -c 'git config --global user.name '$u'; git config --global user.email '$u'@lab; git config --global init.defaultBranch main'; done
su - ana -c 'git clone /srv/git/app.git app && cd app && echo "# App" > README.md && git add . && git commit -m "Initial" && git push -u origin main'
su - luis -c 'git clone /srv/git/app.git app && cd app && echo "by luis" >> README.md && git commit -am "Luis edit" && git push'
su - ana -c 'cd app && git pull && git log --oneline'
```

</details>

---

## Exercise 7 – Rejected push ★★

As `ana`, commit a change **without pulling first** after luis pushed another one. Observe the rejected push and fix it.

<details>
<summary>Solution</summary>

`git push` → `! [rejected] main -> main (fetch first)`. Run `git pull` (merge or resolve conflicts), then `git push`.

</details>

---

## Exercise 8 – Tags, stash, ignore ★

1. Tag the current commit `v1.0` (annotated) and push the tag.
2. Make an uncommitted change, stash it, verify the tree is clean, restore it.
3. Make Git ignore `*.log` files.

<details>
<summary>Solution</summary>

```bash
git tag -a v1.0 -m "First release"; git push origin v1.0
echo tmp >> README.md; git stash; git status; git stash pop
echo "*.log" >> .gitignore; git add .gitignore; git commit -m "Ignore logs"
```

</details>

---

## Challenge – Exam-style task ★★★

> Clone `/srv/git/app.git` into `/root/app`. Create a branch `hotfix`, add a file `fix.txt` containing `patched`, commit it with message `Hotfix`, push the branch to `origin`. Then merge `hotfix` into `main` and push `main`.

**Verify:** `git ls-remote /srv/git/app.git` lists `refs/heads/hotfix` and `refs/heads/main` pointing to commits containing `fix.txt`.

<details>
<summary>Solution</summary>

```bash
git clone /srv/git/app.git /root/app && cd /root/app
git switch -c hotfix
echo patched > fix.txt && git add fix.txt && git commit -m "Hotfix"
git push -u origin hotfix
git switch main && git merge hotfix && git push origin main
git ls-remote /srv/git/app.git
```

</details>

---

<!-- footer:start -->
[← Exercises: SUSE Packages: zypper](04-zypper.md) · [Module exercises](README.md)
<!-- footer:end -->
