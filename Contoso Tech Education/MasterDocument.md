# Contoso Tech Education — Azure Training Lab Guide

> **Azure Training Program 2026** | Powered by Spektra Systems CloudLabs

---

## About This Lab

This lab guide walks you through setting up and using an Azure environment with restricted resources for training purposes. You will create and work with a **Storage Account** and a **Virtual Machine** within defined policy boundaries.

| Detail | Value |
|---|---|
| Client | Contoso Tech Education |
| Duration | 10 days from registration |
| Subscription | Shared Azure Subscription |
| Allowed VM Size | Standard_B1s only |
| Allowed Storage SKU | Standard_LRS only |

> **Important:** You can only register once using your email. Your lab environment expires automatically 10 days after registration.

---

## Lab Guide Files

| # | File | Description |
|---|---|---|
| 00 | [Overview](./00-overview.md) | Lab overview, structure, prerequisites, and rules summary |
| 01 | [Registration](./01-registration.md) | Register and launch the lab environment |
| 02 | [Azure Sign In](./02-azure-signin.md) | Sign in to the Azure Portal |
| 03 | [Storage Account](./03-storage-account.md) | Create a Storage Account with Standard_LRS redundancy |
| 04 | [Virtual Machine](./04-virtual-machine.md) | Create a Virtual Machine with Standard_B1s size |
| 05 | [Connect to VM](./05-connect-vm.md) | Connect via SSH (Linux) or RDP (Windows) |
| 06 | [Exercises](./06-exercises.md) | Blob storage, VM management, and cost monitoring exercises |
| 07 | [Clean Up](./07-cleanup.md) | Delete resources at the end of the lab |
| 08 | [Rules & Help](./08-rules-and-help.md) | Restrictions summary and support contacts |

---

## Quick Start

Follow the steps in order:

1. [Part 1 — Register and Launch the Lab](./01-registration.md)
2. [Part 2 — Sign in to Azure Portal](./02-azure-signin.md)
3. [Part 3 — Create a Storage Account (Standard_LRS)](./03-storage-account.md)
4. [Part 4 — Create a Virtual Machine (Standard_B1s)](./04-virtual-machine.md)
5. [Part 5 — Connect to Your Virtual Machine](./05-connect-vm.md)
6. [Part 6 — Explore and Experiment](./06-exercises.md)
7. [Part 7 — Clean Up (End of Lab)](./07-cleanup.md)

---

## Prerequisites

- A valid email address (personal or organizational)
- Web browser (Edge, Chrome, or Firefox recommended)
- Basic understanding of Azure Portal navigation

---

## Rules and Restrictions

| Rule | Detail |
|---|---|
| Allowed VM Size | Standard_B1s only |
| Allowed Storage SKU | Standard_LRS only |
| Larger resources | Automatically blocked by Azure Policy |
| Disk cost optimization | Enabled — disks downgrade when VM is stopped |
| Lab duration | 10 days from registration |
| Registration | One-time only per email |

---


*Contoso Tech Education — Azure Training Program 2026*
