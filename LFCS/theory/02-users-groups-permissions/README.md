# Module 02 – Users, Groups, Permissions & LDAP: Theory

> [All theory](../00-Theory-Index.md) · [Course module (commands & lab)](../../course/02-Users-Groups-Permissions.md) · [Module exercises](../../exercises/02-users-groups-permissions/README.md)

Read the lessons in order. Each one explains the concepts with definitions and worked examples, then links to its exercises.

| # | Lesson | About | Practice |
|---|--------|----------------|----------|
| 1 | [User Accounts](01-user-accounts.md) | Linux is a multi-user system: every process and every file belongs to a user. | [Exercises](../../exercises/02-users-groups-permissions/01-user-accounts.md) |
| 2 | [Groups](02-groups.md) | Groups let you grant the same access to many users at once: instead of giving five developers access to a directory one by one, you give access to the `developers` group and add the five users to it. | [Exercises](../../exercises/02-users-groups-permissions/02-groups.md) |
| 3 | [Privilege Escalation with sudo and su](03-sudo.md) | Logging in as root for daily work is dangerous: one typo can destroy the system, and nobody can tell who did what. | [Exercises](../../exercises/02-users-groups-permissions/03-sudo.md) |
| 4 | [File Permissions and Ownership](04-permissions-ownership.md) | Every file and directory on Linux has an **owner**, a **group** and a set of **permission bits** that decide who can read, write or execute it. | [Exercises](../../exercises/02-users-groups-permissions/04-permissions-ownership.md) |
| 5 | [Access Control Lists (ACLs)](05-acls.md) | Standard permissions give a file exactly **one** owner and **one** group. | [Exercises](../../exercises/02-users-groups-permissions/05-acls.md) |
| 6 | [Personal and System-Wide Environment Profiles](06-environment-profiles.md) | The LFCS objective *"Manage personal and system-wide environment profiles"* is about deciding **where** a setting must live so that it applies to the right users, at the right time, and survives logouts and package upgrades. | [Exercises](../../exercises/02-users-groups-permissions/06-environment-profiles.md) |
| 7 | [User Resource Limits](07-resource-limits.md) | A single user – or a single runaway program – can exhaust a whole server: a fork bomb creates processes until the system freezes, a buggy service opens files until it hits "Too many open files". | [Exercises](../../exercises/02-users-groups-permissions/07-resource-limits.md) |
| 8 | [Using LDAP for Users and Groups](08-ldap-clients.md) | In an organization with hundreds of servers, creating every user on every machine is impossible to manage. | [Exercises](../../exercises/02-users-groups-permissions/08-ldap-clients.md) |

**Suggested routine per lesson:** read the theory (30–45 min) → try the commands on your lab VMs → do the exercises without opening the solutions → answer *Check yourself*.
