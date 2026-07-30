# Microsoft Enterprise Access Architecture - Study Roadmap

> **Objective:** Understand the evolution of Microsoft's security architecture, from the traditional Active Directory Tier Model to the Enterprise Access Model (EAM), and build a solid foundation for designing secure infrastructures based on Zero Trust principles.

---

# Recommended Learning Path

## 1. Active Directory Tier Model

### Objectives

- Understand the lateral movement problem.
- Learn why administrative accounts must be separated.
- Understand Tier 0, Tier 1, and Tier 2.
- Learn about logon restrictions.
- Understand the purpose of Privileged Access Workstations (PAWs).

### Expected Outcome

Be able to design a secure administrative model for a traditional Active Directory environment.

---

## 2. Privileged Access Strategy

### Objectives

- Understand privileged identities.
- Apply the Principle of Least Privilege.
- Learn secure administrative practices.
- Understand Bastion Hosts.
- Learn how to protect privileged credentials.

### Expected Outcome

Understand Microsoft's strategy for protecting privileged access.

---

## 3. Zero Trust

### Objectives

- Verify Explicitly.
- Use Least Privilege.
- Assume Breach.
- Understand Defense in Depth.

### Expected Outcome

Understand the security philosophy behind Microsoft's modern architecture.

---

## 4. Enterprise Access Model (EAM)

### Objectives

- Understand the Control Plane.
- Understand the Management Plane.
- Understand the Data / Workload Plane.
- Learn how EAM differs from the traditional Tier Model.

### Expected Outcome

Understand why Microsoft evolved beyond the Tier Model.

---

## 5. Microsoft Entra ID

### Objectives

- Hybrid identities.
- Conditional Access.
- Multi-Factor Authentication (MFA).
- Device Identity.
- Identity Governance.

### Expected Outcome

Understand the role of identity within a Zero Trust architecture.

---

## 6. Privileged Identity Management (PIM)

### Objectives

- Just-In-Time (JIT) privilege elevation.
- Temporary administrative access.
- Approval workflows.
- Auditing privileged activities.

### Expected Outcome

Understand how to eliminate permanently privileged administrators.

---

## 7. Privileged Access Workstations (PAWs)

### Objectives

- Dedicated administrative workstations.
- Separation of duties.
- Restricted browsing.
- Secure workstation hardening.

### Expected Outcome

Understand how administrators should securely access the Control Plane.

---

## 8. Just Enough Administration (JEA)

### Objectives

- Least privilege delegation.
- PowerShell constrained administration.
- Limited administrative permissions.

### Expected Outcome

Allow administrators to perform only the tasks they are authorized to execute.

---

## 9. Just-In-Time Administration (JIT)

### Objectives

- Temporary privilege assignment.
- On-demand administrative elevation.
- Automated privilege management.

### Expected Outcome

Reduce the exposure window of privileged accounts.

---

## 10. Microsoft Security Baselines

### Objectives

- Official Microsoft security baselines.
- Operating system hardening.
- Group Policy Objects (GPOs).
- Security compliance.

### Expected Outcome

Apply Microsoft's recommended security configurations across the infrastructure.

---

## 11. Windows LAPS

### Objectives

- Automatic local administrator password management.
- Password rotation.
- Secure credential storage.

### Expected Outcome

Eliminate shared local administrator passwords across managed devices.

---

## 12. Enterprise Access Architecture

### Objectives

Integrate all previous concepts into a unified enterprise security architecture.

### Expected Outcome

Be able to design a modern enterprise infrastructure based on:

- Zero Trust
- Enterprise Access Model
- Least Privilege
- Defense in Depth
- Secure Administration

---

# Official Microsoft Resources

## 1. Active Directory Tiering

https://learn.microsoft.com/security/privileged-access-workstations/

---

## 2. Enterprise Access Model

https://learn.microsoft.com/security/zero-trust/security-adoption-discipline-identity-access-enterprise-model

---

## 3. Zero Trust

https://learn.microsoft.com/security/zero-trust/

---

## 4. Microsoft Entra

https://learn.microsoft.com/entra/

---

## 5. Privileged Identity Management

https://learn.microsoft.com/entra/id-governance/privileged-identity-management/

---

## 6. Microsoft Security Baselines

https://learn.microsoft.com/windows/security/operating-system-security/device-management/windows-security-configuration-framework/security-baselines

---

# Recommended Reading Order

- [ ] Active Directory Tier Model
- [ ] Privileged Access Strategy
- [ ] Zero Trust
- [ ] Enterprise Access Model
- [ ] Microsoft Entra ID
- [ ] Privileged Identity Management (PIM)
- [ ] Privileged Access Workstations (PAWs)
- [ ] Just Enough Administration (JEA)
- [ ] Just-In-Time Administration (JIT)
- [ ] Windows LAPS
- [ ] Microsoft Security Baselines
- [ ] Enterprise Access Architecture

---

# Final Goal

Develop the knowledge required to design and implement a modern enterprise security architecture based on Microsoft's best practices, protecting identities, management systems, and workloads through the Enterprise Access Model and Zero Trust principles.