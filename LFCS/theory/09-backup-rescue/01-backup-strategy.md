# Backup Strategy

<!-- nav:start -->
> **Module 09 · Backup, Recovery & System Rescue** — lesson 1 of 4  
> [Module index](README.md) · [Course lesson (commands)](../../course/09-Backup-Recovery-Rescue.md#lesson-1---backup-strategy-ch-32) · [Exercises for this lesson](../../exercises/09-backup-rescue/01-backup-strategy.md)
<!-- nav:end -->

Disks fail, people delete the wrong directory, upgrades go wrong, ransomware encrypts file servers. A backup is your only way back – but only if it was planned well and actually restores. In this lesson you will learn how to design a backup strategy: what to protect against, what to back up, how often, which backup types to use, where to keep copies, and how to prove that restores work.

## What you will learn

- Threats a backup must cover – and why RAID and snapshots are not backups
- RPO and RTO
- The 3-2-1 rule
- Full, incremental and differential backups; retention schemes
- What to back up on a Linux server (and what not)
- Consistency: backing up live data and databases
- Testing restores and documenting the procedure

---

## Key terms

| Term | Definition |
|------|-----------|
| **Backup** | An independent copy of data from which it can be restored after loss. |
| **Restore** | Getting data back from a backup – the only goal of a backup. |
| **RPO** | *Recovery Point Objective*: maximum acceptable data loss, measured in time (e.g. 24 h). |
| **RTO** | *Recovery Time Objective*: maximum acceptable downtime until service is restored (e.g. 4 h). |
| **Full backup** | Copy of all selected data. |
| **Incremental backup** | Copy of what changed since the **last backup of any type**. |
| **Differential backup** | Copy of what changed since the **last full** backup. |
| **Retention** | How long backups are kept (e.g. 7 daily, 4 weekly, 12 monthly). |
| **Off-site** | Stored in a different physical location. |
| **Immutable / offline** | Cannot be modified or deleted (WORM storage, disconnected media) – protects against ransomware. |
| **Consistent backup** | A backup representing one point in time, with no half-written files. |

---

## What are you protecting against?

| Threat | Example | What protects |
|--------|---------|---------------|
| Hardware failure | disk dies | RAID (availability) **+** backups on other hardware |
| Human error | `rm -rf /srv/data`, bad config change | versioned backups, snapshots |
| Software failure | upgrade corrupts a database | versioned backups, VM/LVM snapshots before changes |
| Malware / ransomware | files encrypted, backups on a mounted share also encrypted | **offline or immutable** copies |
| Site disaster | fire, flood, theft, cloud region outage | **off-site** copies |
| Legal / audit | must keep records 7 years | long retention, archives |

### Why RAID and snapshots are not backups

- **RAID** copies every change – including deletions and corruption – to all disks instantly. It keeps the service **available** when a disk fails; it doesn't let you go back in time.
- **Snapshots** (LVM, filesystem, VM) live on the **same storage**. If the storage dies, the snapshots die too. They're great for quick rollbacks and for taking consistent backups – not as the backup itself.
- **Sync** (e.g. a mirror with `rsync --delete`) replicates deletions too, unless you keep versions.

---

## RPO and RTO

```mermaid
flowchart LR
    LB["last good backup<br/>02:00"] -->|"RPO: data written in this window is lost"| F["failure<br/>15:00"]
    F -->|"RTO: time to restore service"| R["service back<br/>17:00"]
```

| Requirement | Drives |
|-------------|--------|
| RPO = 24 h | nightly backups are enough |
| RPO = 15 min | frequent incrementals, database log shipping, replication |
| RTO = 1 day | restore from tape/cloud is acceptable |
| RTO = 15 min | standby systems, local fast restore, VM snapshots/replicas |

Lower RPO/RTO = more cost. Agree on them with the data owners **before** designing the solution.

---

## The 3-2-1 rule

- **3** copies of the data (production + 2 backups),
- on **2** different types of media/storage,
- **1** copy off-site.

Modern extension: **3-2-1-1-0** – plus **1** offline/immutable copy, and **0** errors in restore tests.

---

## Backup types

```text
Sun        Mon        Tue        Wed        Thu
FULL       INC(Mon)   INC(Tue)   INC(Wed)   INC(Thu)     ← incremental: each since previous backup
FULL       DIFF(Mon)  DIFF(Mon–Tue) DIFF(Mon–Wed) DIFF(Mon–Thu)   ← differential: each since Sunday's full
```

| | Full | Incremental | Differential |
|--|------|-------------|--------------|
| Contains | everything | changes since last backup | changes since last full |
| Backup time / size | longest / largest | shortest / smallest | grows each day |
| Restore needs | 1 set | full + **every** incremental in order | full + **latest** differential |
| Restore speed | fastest | slowest | medium |
| Risk | – | one broken incremental breaks the chain after it | lower |

Example: full on Sunday, failure Friday morning:

- Incremental scheme → restore Sunday full, then Mon, Tue, Wed, Thu incrementals.
- Differential scheme → restore Sunday full, then Thursday's differential.

### Modern variants

- **Incremental forever** with deduplication (borg, restic, Bacula/Bareos with virtual fulls): only changes are sent, yet every backup can be restored as if it were full.
- **Snapshot-based** with hard links (`rsync --link-dest`): every day looks like a full copy, unchanged files take no extra space.

### Retention (GFS)

*Grandfather-Father-Son*: keep e.g. **7 daily**, **4 weekly**, **12 monthly**, **N yearly** backups. Tools like restic/borg implement it with `forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12`.

---

## What to back up on a Linux server

| Back up | Why |
|---------|-----|
| `/etc` | all configuration |
| `/home`, `/root` | user data, admin scripts |
| `/srv`, `/var/www` | served data |
| `/var/lib/<app>` | application state (databases, container volumes, LDAP) – usually via app-specific dumps |
| `/var/spool/cron`, `/var/spool/mail` | crontabs, mail |
| `/usr/local`, `/opt` | locally installed software |
| `/var/log` (optional) | forensics, compliance |
| **Package list** | `dpkg --get-selections` / `apt-mark showmanual` / `rpm -qa` – rebuild the OS quickly |
| **Partition / LVM layout** | `sfdisk -d /dev/sda`, `/etc/lvm/backup/`, `lsblk -f`, `blkid` |
| Bootloader / firmware info | for bare-metal recovery |

Usually **not**: `/proc`, `/sys`, `/dev`, `/run`, `/tmp`, `/var/tmp`, `/var/cache`, mounted network filesystems (back them up on their server), swap.

### Two recovery approaches

| | File-level backup + reinstall | Image / bare-metal backup |
|--|------------------------------|---------------------------|
| How | reinstall OS from media/automation, restore configs and data | restore the whole disk image |
| Tools | tar, rsync, borg, restic, Bacula | dd, Clonezilla, ReaR (Relax-and-Recover), VM snapshots/backups |
| Pros | small, portable, flexible | fast full recovery, exact copy |
| Cons | rebuild time | big; restores problems too; hardware dependent |

With configuration management (Ansible), "reinstall + apply config + restore data" is often the best approach.

---

## Consistency: live systems and databases

Copying files of a running database gives a backup that may **not start**: files change while being read.

| Approach | How |
|----------|-----|
| Application dump | `pg_dump`, `mysqldump --single-transaction`, `slapcat`, `etcdctl snapshot` |
| Stop / quiesce | stop the service, back up, start it (downtime) |
| Snapshot | LVM/filesystem/VM snapshot (ideally after flushing/freezing the app), back up the snapshot, remove it |
| Database-native | WAL archiving / binlogs for point-in-time recovery |

```console
$ sudo -u postgres pg_dump -Fc mydb > /backup/mydb-$(date +%F).dump
$ sudo lvcreate -s -n snap_db -L 2G /dev/vg01/lv_db && sudo mount -o ro /dev/vg01/snap_db /mnt/snap
```

---

## Security of backups

- Backups contain **everything**, including password hashes and keys: encrypt them (borg/restic encrypt by default; `gpg` for tar archives) and restrict access.
- Use a **pull** model or append-only credentials so a compromised server can't delete its own backups.
- Keep at least one copy offline or immutable.

---

## Test your restores

> A backup that has never been restored is a hope, not a backup.

- **Monitor** every job (exit status, size, duration – alert on anomalies).
- **Verify** archives (`tar -tf`, `restic check`, checksums).
- **Restore regularly**: single files monthly, a full server at least yearly, into an isolated environment.
- **Document** the procedure (runbook): where backups are, credentials, order of steps, expected time – so someone else can do it at 3 a.m.

---

## Example policy

| Item | Value |
|------|-------|
| Systems | web01, db01 |
| RPO / RTO | 24 h / 4 h |
| Schedule | daily 02:00 incremental, Sunday full (systemd timer) |
| Database | `pg_dump` at 01:30, included in the backup |
| Target | backup server on-site (7 daily, 4 weekly) + nightly copy to object storage off-site (12 monthly) |
| Encryption | restic repository passwords in a vault |
| Tests | monthly restore of random files; quarterly full restore of db01 into a test VM |

---

## Try it yourself

1. For your two lab VMs, write a one-page backup policy like the one above.
2. List everything on `deb1` that would be needed to rebuild it, and the commands to export the package list and partition layout.
3. Compute what you'd restore in an incremental and a differential scheme for a failure on Thursday evening with full backups on Sundays.

---

## Summary

- Know your **threats**, then set **RPO** (how much loss) and **RTO** (how long down).
- **3-2-1(-1-0)**: copies, media, off-site, offline, verified.
- **Full / incremental / differential**: trade backup size against restore complexity.
- Back up configs, data, package lists and layouts; get **consistent** copies of databases.
- RAID and snapshots are not backups. **Test restores.**

## Check yourself

1. What's the difference between RPO and RTO?
2. Full on Sunday, incrementals daily, failure on Wednesday. What do you restore?
3. Why isn't RAID 1 a backup?
4. Why can copying `/var/lib/mysql` of a running server produce a useless backup?

<details>
<summary>Answers</summary>

1. RPO: how much data (time) you can lose. RTO: how long recovery may take.
2. Sunday's full, then Monday's and Tuesday's incrementals (and Wednesday's if it ran before the failure), in order.
3. It mirrors every change immediately, including deletions and corruption; it only protects against a disk failure.
4. Files are modified during the copy, so the result is inconsistent; use a dump, stop the service, or back up a snapshot.

</details>

## Further reading

- [ArchWiki: Synchronization and backup programs](https://wiki.archlinux.org/title/Synchronization_and_backup_programs)
- [Backblaze: The 3-2-1 backup strategy](https://www.backblaze.com/blog/the-3-2-1-backup-strategy/)
- [Relax-and-Recover (ReaR)](https://relax-and-recover.org/) · [restic](https://restic.net/) · [BorgBackup](https://www.borgbackup.org/)
- [Red Hat: Relax-and-Recover (ReaR) (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_basic_system_settings/assembly_recovering-and-restoring-a-system_configuring-basic-system-settings)

---

<!-- footer:start -->
[Module index](README.md) · [Backup Tools: tar, rsync, dd and Friends →](02-backup-tools.md)  
**Practice now:** [Exercises – Backup Strategy](../../exercises/09-backup-rescue/01-backup-strategy.md)
<!-- footer:end -->
