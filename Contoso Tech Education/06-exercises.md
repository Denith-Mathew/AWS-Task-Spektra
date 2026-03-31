# Part 6 — Explore and Experiment

Try the following exercises within your allowed resources.

---

## Exercise A — Blob Storage Operations

1. Go to your Storage Account → **Containers** → `labdata`.
2. Click **Upload** → upload any small file (< 5 MB).
3. Click on the uploaded file → copy its URL.
4. Open the URL in a new tab (it will fail unless you set public access — this is expected).
5. Generate a **SAS token** for the blob and access it using the SAS URL.

---
<img src ="lab-screenshots\Screenshot 2026-04-01 001951.png">

---
---

## Exercise B — VM Management

1. Go to your VM → click **Stop** to deallocate it.
2. Wait for status to show **Stopped (deallocated)**.
   - Notice: the disk cost optimization will automatically downgrade your disk tier while stopped.
3. Click **Start** to restart the VM.
4. Reconnect via SSH or RDP and verify everything still works.
---
<img src ="lab-screenshots\Screenshot 2026-04-01 011015.png">

---
---
<img src ="lab-screenshots\Screenshot 2026-04-01 011041.png">

---

## Exercise C — Monitor Your Costs

1. Go to **Cost Management + Billing** in the Azure Portal.
2. Click **Cost analysis** → filter by your Resource Group.
3. Observe your accumulated cost.

---
<img src ="lab-screenshots\Screenshot 2026-04-01 012036.png">

---
---
<img src ="lab-screenshots\Screenshot 2026-04-01 012439.png">

---

> **Note:** If your usage reaches 70%, 90%, or 100% of the budget limit, your instructor will receive an email alert. Stay within limits.

---

**Previous Step:** [Part 5 — Connect to Your Virtual Machine](./05-connect-vm.md)

**Next Step:** [Part 7 — Clean Up](./07-cleanup.md)

*Contoso Tech Education — Azure Training Program 2026*
