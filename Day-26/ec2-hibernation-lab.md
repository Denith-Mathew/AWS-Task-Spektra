# AWS EC2 Hibernation – Definition and Lab Guide

## 📘 Definition

**EC2 Hibernation** is a feature in AWS that allows an EC2 instance to save the contents of its **RAM to the encrypted root EBS volume** and stop the instance.

When the instance starts again, AWS **restores the RAM state**, allowing applications and processes to continue exactly where they left off.

### Key Points

* RAM state is saved to **EBS root volume**
* Instance is **stopped (no compute cost)**
* On restart, **RAM is restored**
* Applications continue from the previous state
* Works only with **supported instance types**

---

# 🧠 How EC2 Hibernation Works

```
Running EC2 Instance
        ↓
User triggers Hibernate
        ↓
RAM contents saved to EBS
        ↓
Instance stops
        ↓
Instance started again
        ↓
RAM restored
        ↓
Applications resume
```

---

# 🧪 EC2 Hibernation Lab

This lab demonstrates **how EC2 hibernation preserves RAM state** using:

* `uptime`
* environment variable test
* simple Apache web page

---

# 1️⃣ Prerequisites

EC2 hibernation works only if the following conditions are met:

| Requirement      | Description                                    |
| ---------------- | ---------------------------------------------- |
| Instance type    | Must support hibernation (t3, t2, m5, c5 etc.) |
| Root volume      | Must be **EBS**                                |
| Root volume size | Must be **greater than or equal to RAM size**  |
| Encryption       | Root volume must be **encrypted**              |
| Supported AMI    | Example: **Amazon Linux 2**                    |

---

# 2️⃣ Launch EC2 Instance with Hibernation

Go to:

```
AWS Console → EC2 → Launch Instance
```

### Step 1 — Select AMI

```
Amazon Linux 2
```

---

### Step 2 — Choose Instance Type

Example:

```
t3.micro
```

This instance type **supports hibernation**.

---

### Step 3 — Enable Hibernation

Scroll to **Advanced Details**.

Enable:

```
Enable Hibernation
```

---

# 3️⃣ Add User Data Script

Add the following **User Data** script.

This installs Apache and creates a test webpage.

```bash
#!/bin/bash

yum update -y
yum install httpd -y

systemctl start httpd
systemctl enable httpd

echo "<html>
<h1>EC2 Hibernation Test</h1>
<p>Instance is running</p>
</html>" > /var/www/html/index.html
```

Launch the instance.

---

# 4️⃣ Connect to EC2

SSH into the instance.

```bash
ssh -i key.pem ec2-user@PUBLIC-IP
```

---

# 5️⃣ Run Test Commands

Check instance uptime.

```bash
uptime
```

Example output:

```
10:20:11 up 3 min, 1 user
```

Now create a **RAM test variable**.

```bash
export TEST_VAR="EC2_HIBERNATION_TEST"
```

Verify the variable:

```bash
echo $TEST_VAR
```

Expected output:

```
EC2_HIBERNATION_TEST
```

---

# 6️⃣ Test Web Server

Open the browser:

```
http://PUBLIC-IP
```

You should see:

```
EC2 Hibernation Test
Instance is running
```

---

# 7️⃣ Put Instance into Hibernation

Go to:

```
EC2 → Instance → Instance State → Hibernate
```

AWS performs the following:

```
RAM → saved to encrypted EBS
Instance stops
```

---

# 8️⃣ Start the Instance Again

Click:

```
Start Instance
```

AWS restores:

```
EBS → RAM
```

---

# 9️⃣ Verify Hibernation Worked

Reconnect using SSH.

Check uptime again:

```bash
uptime
```

Example output:

```
up 35 min
```

Notice the uptime **continues instead of resetting**.

This confirms the instance **resumed instead of rebooting**.

---

Check the RAM variable:

```bash
echo $TEST_VAR
```

If the output is still:

```
EC2_HIBERNATION_TEST
```

then **RAM state was successfully restored**.

---

# 🔟 Verify Apache Web Server

Open the browser again:

```
http://PUBLIC-IP
```

The web page should still be accessible.

---

### Optional Verification

Check Apache status:

```bash
systemctl status httpd
```

---

# 🔍 What Happened Internally

During hibernation AWS performs the following:

```
RAM snapshot
        ↓
Saved to encrypted EBS
        ↓
Instance stopped
        ↓
Instance restarted
        ↓
EBS snapshot restored into RAM
```

---

# 📊 Hibernation Process Summary

```
Launch EC2
      ↓
Run uptime test
      ↓
Create RAM variable
      ↓
Hibernate instance
      ↓
RAM saved to EBS
      ↓
Start instance
      ↓
RAM restored
      ↓
Uptime continues
```

---

# ✅ Result

This lab proves that:

* EC2 instance **did not reboot**
* **RAM state was preserved**
* Applications resumed exactly where they stopped

This confirms **EC2 Hibernation works successfully**.

---

# 📚 Use Cases of EC2 Hibernation

* Dev environments
* Long-running memory applications
* Cost optimization
* Faster instance recovery

---

# 🧾 Author

DevOps Lab Documentation – AWS EC2 Hibernation
