# Module 09 – Backup, Recovery & System Rescue: Theory

> [All theory](../00-Theory-Index.md) · [Course module (commands & lab)](../../course/09-Backup-Recovery-Rescue.md) · [Module exercises](../../exercises/09-backup-rescue/README.md)

Read the lessons in order. Each one explains the concepts with definitions and worked examples, then links to its exercises.

| # | Lesson | About | Practice |
|---|--------|----------------|----------|
| 1 | [Backup Strategy](01-backup-strategy.md) | Disks fail, people delete the wrong directory, upgrades go wrong, ransomware encrypts file servers. | [Exercises](../../exercises/09-backup-rescue/01-backup-strategy.md) |
| 2 | [Backup Tools: tar, rsync, dd and Friends](02-backup-tools.md) | Linux ships with powerful backup tools: **tar** for archives, **rsync** for efficient copies and mirrors, **dd** for raw disk images, **cpio** for file lists, plus filesystem-specific dump tools and LVM snapshots for consistency. | [Exercises](../../exercises/09-backup-rescue/02-backup-tools.md) |
| 3 | [The Boot Process – and Where It Breaks](03-boot-process.md) | To repair a system that doesn't boot, you need to know exactly what happens between pressing the power button and seeing a login prompt, and what each failure looks like. | [Exercises](../../exercises/09-backup-rescue/03-boot-process.md) |
| 4 | [System Rescue and Recovery](04-system-rescue.md) | When a system won't boot or nobody can log in, you need to get **inside** it anyway: through special boot targets, by interrupting the boot early, or from external rescue media. | [Exercises](../../exercises/09-backup-rescue/04-system-rescue.md) |

**Suggested routine per lesson:** read the theory (30–45 min) → try the commands on your lab VMs → do the exercises without opening the solutions → answer *Check yourself*.
