# Module 06 – Kernel, Modules & Devices: Theory

> [All theory](../00-Theory-Index.md) · [Course module (commands & lab)](../../course/06-Kernel-Modules-Devices.md) · [Module exercises](../../exercises/06-kernel-devices/README.md)

Read the lessons in order. Each one explains the concepts with definitions and worked examples, then links to its exercises.

| # | Lesson | About | Practice |
|---|--------|----------------|----------|
| 1 | [The Kernel and Boot Parameters](01-kernel-boot-parameters.md) | The **kernel** is the core of Linux: it manages the CPU, memory, devices, filesystems, networking and security, and every program depends on it. | [Exercises](../../exercises/06-kernel-devices/01-kernel-boot-parameters.md) |
| 2 | [Kernel Parameters with sysctl](02-sysctl.md) | Many kernel behaviours can be changed **while the system runs**: whether it forwards packets between interfaces, how eagerly it swaps, how many files can be open, whether it answers pings. | [Exercises](../../exercises/06-kernel-devices/02-sysctl.md) |
| 3 | [Kernel Modules](03-kernel-modules.md) | Most Linux drivers and many features (filesystems, network protocols, firewall components) are built as **loadable kernel modules**: pieces of code that can be added to and removed from the running kernel. | [Exercises](../../exercises/06-kernel-devices/03-kernel-modules.md) |
| 4 | [Devices and udev](04-devices-udev.md) | Unix represents hardware as **device files** in `/dev`. | [Exercises](../../exercises/06-kernel-devices/04-devices-udev.md) |

**Suggested routine per lesson:** read the theory (30–45 min) → try the commands on your lab VMs → do the exercises without opening the solutions → answer *Check yourself*.
