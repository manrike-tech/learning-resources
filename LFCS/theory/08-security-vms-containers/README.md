# Module 08 – Security, Virtualization & Containers: Theory

> [All theory](../00-Theory-Index.md) · [Course module (commands & lab)](../../course/08-Security-VMs-Containers.md) · [Module exercises](../../exercises/08-security-vms-containers/README.md)

Read the lessons in order. Each one explains the concepts with definitions and worked examples, then links to its exercises.

| # | Lesson | About | Practice |
|---|--------|----------------|----------|
| 1 | [Linux Security Modules: DAC and MAC](01-lsm-dac-mac.md) | File permissions (owner, group, other, ACLs) are only half of Linux access control. | [Exercises](../../exercises/08-security-vms-containers/01-lsm-dac-mac.md) |
| 2 | [SELinux](02-selinux.md) | **SELinux** (Security-Enhanced Linux) is the MAC system of the RHEL family. | [Exercises](../../exercises/08-security-vms-containers/02-selinux.md) |
| 3 | [AppArmor](03-apparmor.md) | **AppArmor** is the MAC system of Ubuntu, Debian and SUSE. | [Exercises](../../exercises/08-security-vms-containers/03-apparmor.md) |
| 4 | [Working with SSL/TLS Certificates](04-ssl-certificates.md) | HTTPS, LDAPS, SMTP with STARTTLS, VPNs and many APIs rely on **TLS**, and TLS relies on **X.509 certificates**. | [Exercises](../../exercises/08-security-vms-containers/04-ssl-certificates.md) |
| 5 | [Virtual Machines with libvirt](05-libvirt-vms.md) | Linux is itself a hypervisor: with the **KVM** kernel module, **QEMU** and the **libvirt** management layer, any Linux server can run virtual machines. | [Exercises](../../exercises/08-security-vms-containers/05-libvirt-vms.md) |
| 6 | [Containers: Docker and Podman](06-containers.md) | Containers package an application with everything it needs – libraries, runtime, configuration – and run it isolated from the rest of the system, sharing the host's kernel. | [Exercises](../../exercises/08-security-vms-containers/06-containers.md) |

**Suggested routine per lesson:** read the theory (30–45 min) → try the commands on your lab VMs → do the exercises without opening the solutions → answer *Check yourself*.
