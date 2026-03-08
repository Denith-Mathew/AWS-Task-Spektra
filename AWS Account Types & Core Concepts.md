# AWS Account Types & Core Concepts

> A beginner-friendly guide to understanding AWS account hierarchy, IAM, SCPs, GuardDuty, CloudTrail, and Budgets.

---

## Table of Contents

- [AWS Account Hierarchy](#aws-account-hierarchy)
- [1. Standalone Account](#1-standalone-account)
- [2. AWS Organization](#2-aws-organization)
- [3. Management Account](#3-management-account)
- [4. Member Account](#4-member-account)
- [IAM Users in Member Accounts](#iam-users-in-member-accounts)
- [Cross-Account Access](#cross-account-access)
- [SCP — Service Control Policy](#scp--service-control-policy)
- [GuardDuty & CloudTrail](#guardduty--cloudtrail)
- [AWS Budgets](#aws-budgets)
- [Full Comparison Table](#full-comparison-table)
- [Real World Example](#real-world-example)

---

## AWS Account Hierarchy

```
Level 1:  Standalone Account        ← Just you, alone
Level 2:  AWS Organization          ← A "company" grouping of accounts
            ├── Management Account  ← The "CEO" of the organization
            └── Member Account      ← The "employees/departments"
```

---

## 1. Standalone Account

### What is it?
A **normal AWS account** that you create by signing up at aws.amazon.com. It is **not part of any organization**. It stands alone by itself.

> 💡 Think of it as a **single person** renting their own apartment. They pay their own bills, manage everything themselves, no one above them.

### What you can do:
- Launch EC2, S3, RDS, Lambda — any AWS service
- Create IAM users for your team
- Manage your own billing
- Full control over everything inside

### Use Cases:

| Use Case | Example |
|---|---|
| Personal learning | You learning AWS, doing tutorials |
| Solo developer | A freelancer building their own app |
| Small startup | A tiny team, one account is enough |
| Quick testing | Trying out AWS services temporarily |

### Limitations:
- ❌ No central billing if you have multiple accounts
- ❌ No central governance or policies
- ❌ No control over other accounts
- ❌ Hard to manage as your team grows

---

## 2. AWS Organization

### What is it?
AWS Organizations is **not an account** — it is a **container/grouping** that holds multiple AWS accounts together under one roof.

> 💡 Think of it as a **company** with multiple departments. The company has one HQ (management account) and many departments (member accounts).

### Why do you need it?
When your business grows and you need:
- Multiple accounts for different teams/projects
- One single bill for all accounts
- Central security policies across all accounts

```
Without Organization:           With Organization:

Account A → Bill A              Management Account
Account B → Bill B    vs            ├── Member Account A ─┐
Account C → Bill C                  ├── Member Account B ─┼─ One combined bill
                                    └── Member Account C ─┘
```

---

## 3. Management Account

### What is it?
The account that **creates the AWS Organization**. It becomes the **owner and controller** of all other accounts.

> 💡 Think of it as the **CEO / Head Office** of a company. Makes rules, pays bills, controls everyone under it.

### What you can do:
- Create new member accounts
- Invite existing standalone accounts to join
- Apply **SCPs (Service Control Policies)** — rules that restrict what member accounts can do
- See and pay the **combined bill** of all accounts
- Enable AWS services (GuardDuty, CloudTrail) across all accounts at once

### Use Cases:

| Use Case | Example |
|---|---|
| Central billing | Pay one bill for 50 accounts |
| Security governance | Block all accounts from disabling CloudTrail |
| Account factory | Create new accounts automatically for new teams |
| Cost management | See which department is spending how much |

> ⚠️ **Golden Rule: Never run your applications or workloads in the management account. Use it ONLY for billing and governance.**

---

## 4. Member Account

### What is it?
Any account that is **inside the organization**, under the management account. This is where you **actually run your work**.

> 💡 Think of it as a **department** in a company (HR, Engineering, Finance). Each has its own space, own people, but follows company-wide rules set by HQ.

### What you can do:
- Launch all AWS services (EC2, S3, Lambda, RDS etc.)
- Create IAM users specific to that account
- Manage resources independently
- But **cannot override SCP rules** set by management account

### Use Cases:

| Account Name | Purpose |
|---|---|
| Dev Account | Developers build and test here |
| Staging Account | Testing before going live |
| Production Account | Live application for real users |
| Security Account | Centralized logs, GuardDuty, auditing |
| Shared Services Account | Common tools like CI/CD, DNS |

---

## IAM Users in Member Accounts

Yes! Every member account has its **own IAM system**, completely separate from other accounts.

### Two Common Patterns

**Pattern 1 — IAM Users per Account (Simple)**

```
Dev Account  → IAM User (john) with dev permissions
Prod Account → IAM User (john) with limited prod permissions
```

**Pattern 2 — Centralized Access via IAM Roles (Recommended)**

```
Central Identity Account
└── IAM User: john
        ↓  assumes role
Dev Account  → Role: DevAdminRole       ← john switches into this
Prod Account → Role: ProdReadOnlyRole   ← john switches into this
```

> ✅ **Best Practice Today:** Use **IAM Identity Center (SSO)** — one login, access to all accounts.

---

## Cross-Account Access

> Nothing is automatic. Access only works when you **explicitly set up cross-account roles**.

```
Management Account User  ──❌──▶  Member Account   (NOT automatic)
Member Account User      ──❌──▶  Management Account  (NOT automatic)
```

### Access Rules:

| Scenario | Allowed? | Notes |
|---|---|---|
| Mgmt user → Member account | ✅ Yes | Needs explicit cross-account role |
| Member user → Mgmt account | ⚠️ Technically yes | Needs role, but strongly discouraged |
| Member user → Another member account | ⚠️ Technically yes | Needs role in target account |
| Automatic access anywhere | ❌ No | Nothing is automatic in AWS |

---

## SCP — Service Control Policy

### What is it?
SCP is a **policy/rulebook** created in the **Management Account** that controls **what actions are allowed or denied** in member accounts.

> 💡 Think of it as **company rules given by the CEO to all departments.**
> Even if a department head wants to do something, if the CEO's rulebook says NO — it's NO.

### The AND Rule — Most Important Concept

```
A user can do an action ONLY IF:
✅ SCP allows it
        AND
✅ IAM permission allows it

If EITHER one blocks it → Action is DENIED
```

### Visual Example:

```
Action: Disable CloudTrail

SCP says → ❌ Denied
IAM says → ✅ Allowed (even Admin)
Result   → ❌ CANNOT disable  ← SCP wins
```

### Real SCP Examples

**Block deleting CloudTrail:**
```json
{
  "Effect": "Deny",
  "Action": [
    "cloudtrail:DeleteTrail",
    "cloudtrail:StopLogging"
  ],
  "Resource": "*"
}
```

**Allow only specific regions:**
```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["ap-south-1", "us-east-1"]
    }
  }
}
```

**Block expensive EC2 instances:**
```json
{
  "Effect": "Deny",
  "Action": ["ec2:RunInstances"],
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "StringNotEquals": {
      "ec2:InstanceType": ["t2.micro", "t3.micro"]
    }
  }
}
```

### Why SCP is Used in Member Accounts:

| Reason | Example |
|---|---|
| **Security** | Block disabling GuardDuty, CloudTrail |
| **Cost Control** | Only allow cheap EC2 instance types |
| **Region Restriction** | Only use India/US regions for compliance |
| **Service Restriction** | Dev accounts cannot touch production services |
| **Compliance** | Meet legal requirements across all accounts |

> 💡 **SCP = The CEO's rulebook that every department must follow — even the admin cannot break these rules.**

---

## GuardDuty & CloudTrail

### CloudTrail — What is it?

CloudTrail **records every action/API call** made in your AWS account.

> 💡 Think of it as a **CCTV camera** — it records who did what, when, and from where.

```
Who?    → IAM user "john"
Did?    → Deleted an S3 bucket
When?   → 2024-01-15 10:30 AM
Where?  → IP address 192.168.1.1
```

### Why Enable from Management Account?

```
❌ Bad — Each account manages own CloudTrail:
Dev Account admin → can disable CloudTrail → no recording → danger!

✅ Good — Management account enables for ALL:
→ Member accounts CANNOT disable it (SCP blocks them)
→ All logs go to one central S3 bucket
→ Full audit trail always maintained
```

### GuardDuty — What is it?

GuardDuty is an AWS **threat detection service** that automatically monitors for **suspicious/malicious activity**.

> 💡 Think of it as a **security guard + AI** that raises an alarm if something looks dangerous.

### What it detects:
- 🚨 Brute force attacks on EC2 instances
- 🚨 IAM user making API calls from unusual country
- 🚨 Cryptocurrency mining on your EC2 (hacker planted it)
- 🚨 S3 bucket data being stolen
- 🚨 Unusual spikes in API calls (possible attack)

### Why Enable from Management Account?

```
✅ Management Account enables GuardDuty for ALL member accounts
→ ALL threats from ALL accounts come to ONE place
→ Security team sees everything in a single dashboard
→ Member accounts CANNOT disable it
```

### Summary:

| Service | Purpose | Why from Management Account |
|---|---|---|
| **CloudTrail** | Records every action in AWS | So no member account can hide actions by turning it off |
| **GuardDuty** | Detects threats and attacks | So security team sees all threats across all accounts in one place |

---

## AWS Budgets

### What is it?
AWS Budgets lets you **set a spending limit** on your AWS account and **get alerts** when you are close to or exceeding that limit.

> 💡 Think of it like a **monthly household budget** — get warned before you overspend.

### Without Budgets vs With Budgets:

```
❌ Without Budgets:
Forget to stop EC2 → runs 30 days → Surprise bill of $5,000 😱

✅ With Budgets:
Set $100 limit → Alert at $80 → You stop resources → No surprise bill 😊
```

### Types of AWS Budgets:

| Type | What it tracks |
|---|---|
| **Cost Budget** | How much money you spend |
| **Usage Budget** | How much of a service you use (e.g. EC2 hours) |
| **Savings Plans Budget** | Whether your savings plan is being utilized |
| **Reservation Budget** | Whether your reserved instances are being used |

### Budget in AWS Organization:

```
Management Account
│
├── Member Account (Dev)      → Budget: $500/month
├── Member Account (Staging)  → Budget: $300/month
└── Member Account (Prod)     → Budget: $5,000/month
```

### AWS Budgets vs AWS Cost Explorer:

| Feature | AWS Budgets | AWS Cost Explorer |
|---|---|---|
| Purpose | Set limits & get alerts | Analyze past spending |
| Direction | Future focused | Past focused |
| Alerts | ✅ Yes | ❌ No |
| Think of it as | Speed limit warning | Fuel consumption report |

> ⚠️ **Golden Rule: Always set a budget the moment you create an AWS account — even if it's just $10.**

---

## Full Comparison Table

| Feature | Standalone | Management Account | Member Account |
|---|---|---|---|
| Part of Organization | ❌ No | ✅ Yes (creates it) | ✅ Yes (joins it) |
| Runs workloads | ✅ Yes | ⚠️ Not recommended | ✅ Yes |
| Controls other accounts | ❌ No | ✅ Yes | ❌ No |
| Has its own billing | ✅ Yes | Pays for all accounts | Rolls up to mgmt |
| Affected by SCPs | ❌ No | ❌ No | ✅ Yes |
| Count per org | — | Exactly 1 | Unlimited |

---

## Real World Example

```
TechCorp AWS Setup
│
├── Management Account (techcorp-master)
│     → Only used for billing, creating accounts, applying policies
│
├── Member Account: Security (techcorp-security)
│     → Stores all logs, runs GuardDuty, CloudTrail
│
├── Member Account: Development (techcorp-dev)
│     → Developers build features here
│     → Budget: $500/month
│
├── Member Account: Staging (techcorp-staging)
│     → QA team tests here before release
│     → Budget: $300/month
│
└── Member Account: Production (techcorp-prod)
      → Real users access your app from here
      → Strictest SCPs applied
      → Budget: $5,000/month
```

---

## Quick Summary

| Concept | One Line Explanation |
|---|---|
| **Standalone Account** | Single AWS account, alone, good for individuals/small teams |
| **AWS Organization** | A container grouping multiple accounts under one roof |
| **Management Account** | The boss account — handles billing and governance only |
| **Member Account** | Worker accounts — where actual apps and workloads run |
| **IAM User** | A user identity inside an AWS account with permissions |
| **SCP** | CEO's rulebook — restricts what member accounts can do |
| **CloudTrail** | CCTV — records every action across all accounts |
| **GuardDuty** | Security guard — detects threats across all accounts |
| **AWS Budgets** | Spending limit — alerts you before bills get too high |

---

> 📌 **Growth Path:** Start with Standalone → As you grow → Create an Organization → Management account controls → Member accounts do the actual work
