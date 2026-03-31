# Part 3 — Create a Storage Account (Standard_LRS)

## Steps

1. In the Azure Portal, click **+ Create a resource**.
2. Search for **Storage account** and select it.
3. Click **Create** and fill in the following:

| Field | Value |
|---|---|
| Subscription | (pre-selected — shared subscription) |
| Resource Group | Select your assigned resource group |
| Storage account name | `contosostudent<yourname>` (lowercase, no spaces, globally unique) |
| Region | Same as your lab region |
| Performance | **Standard** |
| Redundancy | **LRS (Locally-redundant storage)** |

4. Click **Review + Create** → **Create**.
5. Wait for deployment to succeed.
6. Navigate to your new storage account and explore:
   - **Containers** — create a blob container named `labdata`
   - **Access keys** — view (do not share) your storage keys
   - **Properties** — note the primary endpoint URL

## Verify

- Open your storage account → **Overview** → confirm SKU shows **Standard_LRS**.

> **Warning:** Attempting to create a storage account with any other redundancy (GRS, ZRS, GZRS, Premium) will be **automatically denied** by policy.

---

**Previous Step:** [Part 2 — Sign in to Azure Portal](./02-azure-signin.md)

**Next Step:** [Part 4 — Create a Virtual Machine](./04-virtual-machine.md)

*Contoso Tech Education — Azure Training Program 2026*
