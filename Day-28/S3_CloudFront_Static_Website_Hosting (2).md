# 🌐 Static Website Hosting on S3 + CloudFront

---

## 📸 Screenshots Reference

### Screenshot 1 — Restaurant Website (Static Website Result)
![Restaurant Login Page](../user-data/uploads/Screenshot_2026-03-11_153141.png)
> The static restaurant website hosted on S3 and served via CloudFront — showing the login page with a restaurant background image.

---

### Screenshot 2 — S3 Bucket Policy (OAC Policy Applied)
![S3 Bucket Policy](../user-data/uploads/Screenshot_2026-03-11_153349.png)
> S3 bucket `denithmatehffyueh32384` with the CloudFront OAC policy applied under the Permissions tab.

---

### Screenshot 3 — S3 Bucket Objects
![S3 Bucket Objects](../user-data/uploads/Screenshot_2026-03-11_153844.png)
> Bucket contains 2 objects:
> - `Outlook-eiwwmuzm.jpg` — background image (5.9 KB)
> - `resto.html` — main website HTML file (1.5 KB)

---

## 🏗️ Architecture Overview

```
User → CloudFront (CDN) → S3 Bucket (Origin)
```

CloudFront serves as the CDN layer, caching your S3 content globally for fast delivery.

---

## 📋 Step-by-Step Setup

### Step 1: Create an S3 Bucket

1. Go to **AWS Console → S3 → Create Bucket**
2. Enter a **bucket name** (e.g., `denithmatehffyueh32384`)
3. Choose your **region** (e.g., `us-east-1`)
4. **Uncheck** "Block all public access" *(temporarily, for setup)*
5. Click **Create Bucket**

---

### Step 2: Upload Your Website Files

1. Open your bucket → Click **Upload**
2. Upload your files:
   - `resto.html` (main HTML file)
   - `Outlook-eiwwmuzm.jpg` (background image)
3. Click **Upload**

---

### Step 3: Enable Static Website Hosting

1. Go to bucket → **Properties** tab
2. Scroll to **Static website hosting** → Click **Edit**
3. Select **Enable**
4. Set:
   - **Index document:** `resto.html`
   - **Error document:** `error.html`
5. Click **Save changes**

---

### Step 4: Create CloudFront Distribution

1. Go to **AWS Console → CloudFront → Create Distribution**
2. Configure:

| Setting | Value |
|---|---|
| **Origin Domain** | `denithmatehffyueh32384.s3-website-us-east-1.amazonaws.com` |
| **Origin Protocol** | HTTP only |
| **Viewer Protocol Policy** | Redirect HTTP to HTTPS |
| **Allowed HTTP Methods** | GET, HEAD |
| **Default Root Object** | `resto.html` |
| **Origin Access** | Origin Access Control (OAC) |

3. Click **Create Distribution**
4. Wait ~5–10 minutes for status → **Deployed**

---

### Step 5: Apply OAC Bucket Policy

After creating the distribution, apply this policy to your S3 bucket under **Permissions → Bucket Policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::denithmatehffyueh32384/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::493162619993:distribution/E4YBCCOWMKKOQ"
        }
      }
    }
  ]
}
```

> ⚠️ Replace `denithmatehffyueh32384`, `493162619993`, and `E4YBCCOWMKKOQ` with your actual bucket name, account ID, and distribution ID.

---

### Step 6: Block Public S3 Access

After applying the OAC policy:

1. Go to S3 → **Permissions → Block public access**
2. **Enable all** — S3 is now private ✅
3. Only CloudFront can access the bucket

---

### Step 7: Access Your Website

Use your CloudFront domain:

```
https://d2cfpawiymwoxn.cloudfront.net/resto.html
```

Find it at: **CloudFront → Distributions → Domain Name column**

---

## 🔐 Policy Explained

| Field | Meaning |
|---|---|
| `Principal: cloudfront.amazonaws.com` | Only CloudFront service can access S3 |
| `Action: s3:GetObject` | Allows reading/fetching objects |
| `Resource: arn:aws:s3:::BUCKET-NAME/*` | Applies to all files in bucket |
| `AWS:SourceArn` | Restricts to your specific CloudFront distribution only |

---

## 🛡️ Security Model

```
Without OAC:          With OAC (Current Setup):
S3 = Public ❌        S3 = Private ✅
Anyone can access     Only CloudFront can access
via S3 URL            Direct S3 URL is blocked
```

---

## 📂 Your Bucket Details

| Property | Value |
|---|---|
| **Bucket Name** | `denithmatehffyueh32384` |
| **Region** | `us-east-1` (N. Virginia) |
| **Account ID** | `493162619993` |
| **Distribution ID** | `E4YBCCOWMKKOQ` |
| **CloudFront URL** | `https://d2cfpawiymwoxn.cloudfront.net/resto.html` |
| **Files** | `resto.html`, `Outlook-eiwwmuzm.jpg` |

---

## ✅ Summary Checklist

- [x] S3 bucket created
- [x] Website files uploaded (`resto.html` + image)
- [x] Static website hosting enabled
- [x] CloudFront distribution created
- [x] OAC policy applied to S3 bucket
- [ ] Block public S3 access (recommended)
- [ ] Custom domain via Route 53 + ACM (optional)
