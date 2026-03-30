# Usage Policy – Contoso Tech Education Azure Lab

## Allowed Resources
- Virtual Machines: **Standard_B1s only**
- Storage Accounts: **Standard_LRS only**

## Blocked Resources
- Any VM size other than Standard_B1s
- Any Storage SKU other than Standard_LRS (Premium_LRS, Standard_GRS, Standard_ZRS)
- Premium Disks
- Any other Azure services not listed above

## Policy Rules
- Any attempt to create a larger VM or premium storage will be **automatically blocked** by Azure Policy
- No manual approval or override is available
- Violations are logged and reported to the lab administrator

## Cost Control
- Each student has a budget limit per lab session
- Email alert is triggered when usage reaches the threshold
- Lab access is suspended if budget is exceeded

## Access Policy
- Lab access is valid for **10 days** from registration
- Each student can register **only once**
- All resources are **automatically deleted** after 10 days

> Attempting to bypass these restrictions will result in immediate lab suspension.
