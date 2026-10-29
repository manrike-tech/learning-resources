# Module 04 – Processes, Services, Scheduling & Performance: Theory

> [All theory](../00-Theory-Index.md) · [Course module (commands & lab)](../../course/04-Processes-Services-Performance.md) · [Module exercises](../../exercises/04-processes-services/README.md)

Read the lessons in order. Each one explains the concepts with definitions and worked examples, then links to its exercises.

| # | Lesson | About | Practice |
|---|--------|----------------|----------|
| 1 | [Processes](01-processes.md) | Everything that runs on Linux – your shell, the SSH server, a backup script – is a **process**. | [Exercises](../../exercises/04-processes-services/01-processes.md) |
| 2 | [Monitoring System Performance](02-monitoring.md) | "The server is slow" is one of the most common tickets an administrator receives. | [Exercises](../../exercises/04-processes-services/02-monitoring.md) |
| 3 | [Troubleshooting Disk Space](03-disk-space.md) | A full filesystem breaks things in surprising ways: databases stop, logins fail, services can't write PID files, package updates abort. | [Exercises](../../exercises/04-processes-services/03-disk-space.md) |
| 4 | [Managing Services with systemd](04-systemd.md) | **systemd** is the init system and service manager of every major distribution. | [Exercises](../../exercises/04-processes-services/04-systemd.md) |
| 5 | [Application and Service Constraints](05-service-constraints.md) | The LFCS competency *"Determine application and service specific constraints"* is about understanding everything that can stop a service from running correctly besides its own code: the user it runs as, file permissions, ports, resource limits, security policies, dependencies and the firewall. | [Exercises](../../exercises/04-processes-services/05-service-constraints.md) |
| 6 | [Scheduling Jobs: cron, at and systemd Timers](06-scheduling-jobs.md) | Backups at night, log cleanup every hour, a report every Monday morning, a one-off reboot at 23:00: administrators schedule work all the time. | [Exercises](../../exercises/04-processes-services/06-scheduling-jobs.md) |

**Suggested routine per lesson:** read the theory (30–45 min) → try the commands on your lab VMs → do the exercises without opening the solutions → answer *Check yourself*.
