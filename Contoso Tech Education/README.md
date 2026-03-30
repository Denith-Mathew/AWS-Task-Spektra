# Contoso Tech Education - Azure Cost-Controlled Training Lab

## Overview
This repository contains all resources for the Azure Cost-Controlled Training Lab by Contoso Tech Education. Students work within a shared Azure subscription with strict cost controls, restricted to Standard_LRS Storage Accounts and Standard_B1s VMs only.

---

## Lab Details

| Item | Details |
|---|---|
| **Lab Name** | Azure Cost-Controlled Training Lab |
| **Platform** | CloudLabs Spektra |
| **Access Duration** | 10 Days from Registration |
| **Registration** | One-time only per student |
| **Allowed VM Size** | Standard_B1s only |
| **Allowed Storage** | Standard_LRS only |

---

## Repository Files

### 📘 Lab Guide
Step-by-step instructions for students to complete the lab exercises.

🔗 [View Lab Guide](https://raw.githubusercontent.com/Denith-Mathew/AWS-Task-Spektra/refs/heads/main/Contoso%20Tech%20Education/lab_quide.md)

---

### 📋 Policy Reference
Azure Policy definitions restricting VM sizes and Storage Account SKUs.

🔗 [View Policy File](https://raw.githubusercontent.com/Denith-Mathew/AWS-Task-Spektra/refs/heads/main/Contoso%20Tech%20Education/policy-vm-storage.md)

---

### 🖼️ Logo
Contoso Tech Education custom logo used in the lab launch page.

🔗 [View Logo](https://raw.githubusercontent.com/Denith-Mathew/AWS-Task-Spektra/main/Contoso%20Tech%20Education/contoso-logo.jpg)

---

## Lab Restrictions
- ❌ VM sizes larger than Standard_B1s are **blocked**
- ❌ Storage SKUs other than Standard_LRS are **blocked**
- ✅ VM disks are **auto-downgraded** when VM is stopped
- ✅ Budget alerts sent to admin via **email**
- ✅ Lab auto-expires after **10 days**
- ✅ Each student can register **only once**

---

## Contact
For lab issues, contact your instructor or lab administrator.
