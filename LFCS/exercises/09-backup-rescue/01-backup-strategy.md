# Exercises – Backup Strategy

<!-- nav:start -->
> **Module 09 · Backup, Recovery & System Rescue** — lesson 1 of 4  
> Read first: [Theory – Backup Strategy](../../theory/09-backup-rescue/01-backup-strategy.md) · [Course lesson (commands)](../../course/09-Backup-Recovery-Rescue.md#lesson-1---backup-strategy-ch-32) · [Module exercises](README.md)
<!-- nav:end -->

> These exercises are mostly planning and reasoning – the same skills an interviewer or an auditor will test. Write your answers in `/root/backup-plan.md`.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Threats ★

For each scenario, say whether RAID 1, an LVM snapshot, an on-site backup and an off-site offline backup would let you recover:

1. A disk fails. 2. A user deletes a directory. 3. Ransomware encrypts the server and its mounted backup share. 4. The server room floods.

<details>
<summary>Solution</summary>

| | RAID 1 | LVM snapshot | on-site backup | off-site offline |
|--|--------|--------------|----------------|------------------|
| 1 disk fails | ✔ | ✘ (same disks) | ✔ | ✔ |
| 2 deletion | ✘ | ✔ (if taken before) | ✔ | ✔ |
| 3 ransomware | ✘ | ✘ (likely) | ✘ if mounted/writable | ✔ |
| 4 flood | ✘ | ✘ | ✘ | ✔ |

</details>

---

## Exercise 2 – RPO and RTO ★

A shop's database may lose at most 1 hour of orders and must be back within 2 hours. Is a nightly tar backup to a USB disk sufficient? What would you propose?

<details>
<summary>Solution</summary>

No: RPO 24 h > 1 h. Propose hourly incrementals or continuous WAL/binlog archiving, plus nightly full dumps, a local fast restore target (RTO), and an off-site copy.

</details>

---

## Exercise 3 – Restore sets ★★

Full backups Sunday 02:00. Daily backups 02:00 Monday–Saturday. Failure Thursday 15:00. List exactly which backups you restore with (a) incremental and (b) differential schemes. How much data is lost?

<details>
<summary>Solution</summary>

(a) Sunday full + Mon, Tue, Wed, Thu incrementals. (b) Sunday full + Thursday differential. Data written Thursday 02:00–15:00 is lost (13 h).

</details>

---

## Exercise 4 – What to back up ★★

For `deb1`, list in `/root/backup-plan.md` the directories and extra information you'd back up, with the commands to export the package list and the partition/LVM layout. List what you'd exclude and why.

<details>
<summary>Solution</summary>

```bash
apt-mark showmanual > /backup/meta/packages.txt          # or dpkg --get-selections
sfdisk -d /dev/vda > /backup/meta/vda.sfdisk
cp -a /etc/lvm/backup /backup/meta/lvm-backup
lsblk -f > /backup/meta/lsblk.txt; blkid > /backup/meta/blkid.txt
```

Include `/etc /home /root /srv /var/lib/<apps> /var/spool/cron /usr/local /opt`; exclude `/proc /sys /dev /run /tmp /var/tmp /var/cache`, network mounts.

</details>

---

## Exercise 5 – Consistency ★★

Explain how you'd take a consistent backup of a PostgreSQL database that can't be stopped. Give two methods with commands.

<details>
<summary>Solution</summary>

1. `sudo -u postgres pg_dump -Fc dbname > /backup/db.dump` (logical, consistent snapshot of the DB).
2. LVM snapshot of the data LV (after `CHECKPOINT`), mount read-only, archive, remove the snapshot.

</details>

---

## Challenge – Write the policy ★★★

Write a complete backup policy for your lab (both VMs) covering: RPO/RTO, what, schedule (full/incremental), tools, targets (on-site, off-site), retention (GFS), encryption, monitoring, and a restore-test schedule. Then implement at least the `/etc` part in the [backup tools exercises](02-backup-tools.md).

---

<!-- footer:start -->
[Module exercises](README.md) · [Exercises: Backup Tools: tar, rsync, dd and Friends →](02-backup-tools.md)
<!-- footer:end -->
