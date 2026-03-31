# Part 4 — Create a Virtual Machine (Standard_B1s)

## Steps

1. In the Azure Portal, click **+ Create a resource**.
2. Search for **Virtual machine** and select it.
3. Click **Create** → **Azure virtual machine** and fill in:

| Field | Value |
|---|---|
| Subscription | (pre-selected) |
| Resource Group | Select your assigned resource group |
| Virtual machine name | `contoso-vm-<yourname>` |
| Region | Same as your lab region |
| Image | **Ubuntu Server 22.04 LTS** or **Windows Server 2022 Datacenter** |
| Size | **Standard_B1s** (click "See all sizes" → filter by B1s) |
| Authentication | Password |
| Username | `azurestudent` |
| Password | Choose a strong password (12+ chars, upper, lower, number, special) |

4. Under **Disks** tab:
   - OS disk type: **Standard HDD** (recommended for cost savings)

5. Under **Networking** tab:
   - Leave defaults (a new vNet, subnet, and public IP will be created)

6. Click **Review + Create** → **Create**.
7. Wait for deployment to complete.

## Verify

- Go to your VM → **Overview** → confirm Size shows **Standard_B1s**.
- Note the **Public IP address** for connecting later.

> **Warning:** Attempting to create any VM size other than Standard_B1s (e.g., B2s, D2s_v3, etc.) will be **automatically denied** by policy.

---

**Previous Step:** [Part 3 — Create a Storage Account](./03-storage-account.md)

**Next Step:** [Part 5 — Connect to Your Virtual Machine](./05-connect-vm.md)

*Contoso Tech Education — Azure Training Program 2026*
