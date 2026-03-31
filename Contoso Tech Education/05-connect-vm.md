# Part 5 — Connect to Your Virtual Machine

## If you chose Linux (Ubuntu)

1. Open a terminal (PowerShell, CMD, or your system terminal).
2. Run the following command:
   ```
   ssh azurestudent@<your-vm-public-ip>
   ```
3. Enter your password when prompted.
4. You are now connected. Run `hostname` to verify.

---

## If you chose Windows Server

1. Open **Remote Desktop Connection** (search `mstsc` in Start menu).
2. Enter your VM's **Public IP address**.
3. Click **Connect** → enter username `azurestudent` and your password.
4. You are now connected to your Windows VM.

---

> **Tip:** You can find your VM's Public IP address by going to your VM → **Overview** in the Azure Portal.

---

**Previous Step:** [Part 4 — Create a Virtual Machine](./04-virtual-machine.md)

**Next Step:** [Part 6 — Explore and Experiment](./06-exercises.md)

*Contoso Tech Education — Azure Training Program 2026*
