# CloudFront Internal-Only Access (Option 1: Signed URLs / Signed Cookies) — Manual AWS Console Runbook

**Goal:** Make CloudFront content *cryptographically private* so that **only requests with valid CloudFront signatures** can download artifacts.  
**best way is to use WAF. No IP allowlists. however we dont know yet what the WAF cldr list of IPs is -- this is work in progress to find that info**    
**Origin:** Amazon S3 (private) protected by **Origin Access Control (OAC)**.

---

## Overview

CloudFront will be publicly reachable, but our objects are **not** publicly accessible because:

1. **S3 is private** and only CloudFront can fetch objects (via **OAC**).
2. CloudFront behaviors require **Signed URLs / Signed Cookies** using a **Key Group**.
3. A trusted internal system (app/Lambda/service) issues signed URLs/cookies to authorized users.

### Request flow

1. User/service requests a download from **source** (or your “signer” API).
2. You authorizes the request (customer entitlement / login / IAM / etc.).
3. You returns a **signed CloudFront URL** (or sets signed cookies).
4. User downloads from CloudFront using the signed URL/cookies.
5. CloudFront validates signature using **public key** and serves content; otherwise **403**.

---

## Prerequisites

- AWS access to manage: **S3**, **CloudFront**, **Secrets Manager** (optional but recommended), and whichever service signs URLs (e.g., **Lambda**).
- A bucket name for artifacts, e.g. `my-artifacts-bucket`.
- Decide which paths must be protected. Recommended:
  - Protect **everything** (Default behavior), or
  - Protect only specific prefixes (e.g. `/reports/*`, `/artifacts/*`).

---

## Step 1 — Create or verify the S3 bucket (private)

1. AWS Console → **S3** → **Create bucket**
2. Choose:
   - Bucket name: `my-artifacts-bucket`
   - Region: your preference
3. **Block Public Access**:
   - Keep **all** “Block Public Access” settings enabled
4. Create bucket
5. Confirm private settings:
   - S3 → Bucket → **Permissions**
   - Ensure **Block Public Access** is **ON**
   - Do **not** enable public ACLs or public bucket policy

✅ Result: Bucket is private, but CloudFront is not yet allowed.

---

## Step 2 — Generate an RSA key pair (for CloudFront signing)

Do this on a secure admin machine.

```bash
openssl genrsa -out cloudfront-private-key.pem 2048
openssl rsa -pubout -in cloudfront-private-key.pem -out cloudfront-public-key.pem
```

**Do not share the private key.**  
The private key is used only by the trusted signer service.

✅ Result: You have:
- `cloudfront-public-key.pem` (upload to CloudFront)
- `cloudfront-private-key.pem` (store securely, e.g., Secrets Manager)

---

## Step 3 — Create a CloudFront Public Key (Console)

1. AWS Console → **CloudFront**
2. Left nav → **Public keys**
3. Click **Create public key**
4. Enter:
   - Name: `internal-downloads-public-key`
   - Description: optional
   - **Encoded key**: paste contents of `cloudfront-public-key.pem`
5. Create

Record:
- **Public key ID** (CloudFront uses this internally)
- **Key Pair ID** (shown in the public key details; used by signed URLs)

✅ Result: CloudFront now has the public key needed to validate signatures.

---

## Step 4 — Create a Key Group (Console)

1. CloudFront → **Key groups**
2. Click **Create key group**
3. Enter:
   - Name: `internal-downloads-key-group`
4. Under **Public keys**, add `internal-downloads-public-key`
5. Create

✅ Result: You have a Key Group you can attach to behaviors to require signatures.

---

## Step 5 — Create an Origin Access Control (OAC) (Console)

1. CloudFront → Left nav → **Origin access**
2. Click **Create control**
3. Set:
   - Name: `my-s3-oac`
   - Origin type: **S3**
   - Signing behavior: **Always**
   - Signing protocol: **SigV4**
4. Create

✅ Result: This allows CloudFront to sign requests to S3 securely.

---

## Step 6 — Create the CloudFront Distribution (Console)

1. CloudFront → **Distributions** → **Create distribution**
2. **Origin**
   - Origin domain: select `my-artifacts-bucket`
   - Origin access: choose **Origin access control settings**
   - Select OAC: `my-s3-oac`
3. **Default cache behavior**
   - Viewer protocol policy: **Redirect HTTP to HTTPS**
   - **Important:** decide internal-only scope:
     - **Recommended internal-only:** set “Restrict viewer access” = **Yes** (we’ll finalize after behaviors exist)
4. Create distribution

Wait until Status = **Deployed**.

Record:
- Distribution **ID**
- Distribution **Domain name** (e.g., `dxxxxxxx.cloudfront.net`)
- Distribution **ARN** (used in S3 bucket policy)

✅ Result: CloudFront is created, but S3 still needs the bucket policy to allow CloudFront OAC access.

---

## Step 7 — Lock S3 so ONLY this CloudFront distribution can read objects

1. CloudFront → open the distribution → copy the **Distribution ARN**
   - Example: `arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE`
2. S3 → bucket → **Permissions** → **Bucket policy** → **Edit**
3. Paste and edit the following policy:

> Replace:
> - `my-artifacts-bucket`
> - account id / distribution id in `AWS:SourceArn`

```json
{
  "Version": "2025-xx-xx",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOACOnly",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-artifacts-bucket/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
        }
      }
    }
  ]
}
```

4. Save bucket policy

✅ Result: S3 objects can **only** be fetched through this CloudFront distribution (no origin bypass).

---

## Step 8 — Require Signed URLs/Cookies on CloudFront (internal-only)

### Option A (Recommended as of our immidiate NPI needs): Require signatures on **all paths**
1. CloudFront → Distribution → **Behaviors**
2. Select **Default behavior** → **Edit**
3. Set:
   - Viewer protocol policy: **Redirect HTTP to HTTPS**
   - **Restrict viewer access:** **Yes**
   - **Trusted key groups:** select `internal-downloads-key-group`
4. Save changes

✅ Result: Every request requires a signature (true internal-only).

### Option B: Require signatures only on a subset of paths. for future needs when we have more distribution lists
and we might still have public paths for openai etc. usecases.

1. CloudFront → Distribution → **Behaviors** → **Create behavior**
2. Path pattern examples:
   - `/artifacts/*`
   - `/reports/*`
3. In that behavior set:
   - **Restrict viewer access:** **Yes**
   - **Trusted key groups:** `internal-downloads-key-group`
4. Save changes

✅ Result: Only those paths require signatures; other paths may remain public.

---

## Step 9 — Store the private key securely (MUST)

**Do not store the private key in source code or client machines.**

### Store in AWS Secrets Manager
1. AWS Console → **Secrets Manager** → **Store a new secret**
2. Secret type: **Other type of secret**
3. Paste contents of `cloudfront-private-key.pem` (as plaintext PEM)
4. Secret name: `cloudfront/internal-downloads/private-key`
5. Save

✅ Result: A signer service can fetch the key at runtime.

---

## Step 10 — Create the “Signer” service (manually in UI pattern)

You need a trusted server-side component that:
- authenticates/authorizes the requester (customer entitlement, login, etc.)
- creates a signed URL/cookie
- returns it to the requester

### Recommended manual setup: Lambda + API Gateway (IAM or JWT auth)
High-level steps:
1. **Lambda**: Create function `cloudfront-url-signer`
   - Runtime: Python 3.11 (or Node.js)
   - Env vars:
     - `CLOUDFRONT_DOMAIN = dxxxxxxx.cloudfront.net`
     - `KEY_PAIR_ID = <from CloudFront Public Key details>`
     - `PRIVATE_KEY_SECRET_ARN = <Secrets Manager ARN>`
   - IAM for Lambda role:
     - `secretsmanager:GetSecretValue` on the secret ARN
     - CloudWatch Logs permissions
2. **API Gateway**: Create REST API
   - Resource: `/sign`
   - Method: `POST`
   - Integration: Lambda proxy to `cloudfront-url-signer`
   - Authorization: your choice
     - **AWS_IAM** for internal AWS workloads
     - **JWT authorizer** for customers (Cognito / OIDC), if needed

### What the signer endpoint should accept/return
**Request body**
```json
{ "path": "/artifacts/xxxxx.zip", "ttl_sec": 600 }
```

**Response body**
```json
{ "signed_url": "https://dxxxxxxx.cloudfront.net/artifacts/xxxxx.zip?Expires=...&Signature=...&Key-Pair-Id=..." }
```

✅ Result: Customers never learn keys—only receive a time-limited signed URL.

---

## Step 11 — Testing & validation

### 1) Verify direct S3 access is blocked
Try opening an S3 object URL:
- `https://my-artifacts-bucket.s3.amazonaws.com/artifacts/xxxxxx.zip`

Expected:
- **AccessDenied**

### 2) Verify CloudFront without signature is blocked
Try opening:
- `https://dxxxxxxx.cloudfront.net/artifacts/xxxxxx.zip`

Expected:
- **403 Forbidden**

### 3) Verify signed URL works
Call your signer and open the returned signed URL.

Expected:
- **200 OK** / download succeeds

### 4) Verify signature expiry
Wait until TTL expires and retry.

Expected:
- **403 Forbidden**

---

## Operational guidance

### TTL best practices
- Artifacts: 5–15 minutes recommended
- Large downloads: consider 30–60 minutes to avoid expiry mid-download
- If users need many downloads: consider **Signed Cookies** instead of many signed URLs

### Key rotation
- Create a **new** public/private key pair
- Add new public key to CloudFront
- Update Key Group to include new public key
- Update signer to use the new private key
- After rollout, remove old key from Key Group

### Logging (recommended)
- Enable **CloudFront standard logs** or **real-time logs**
- Enable **S3 access logs** (optional)
- Add API Gateway + Lambda logs for signer requests

---

## Common pitfalls

- **Forgetting to update S3 bucket policy** → CloudFront gets 403 from S3
- **Leaving a behavior public** → objects may be accessible without signatures
- **Storing private key in repo** → avoid; use Secrets Manager
- **TTL too short** for big downloads → users see expiry failures
- **Not locking S3 origin** → users can bypass CloudFront if S3 is public/allowed

---

## Appendix: When to use Signed Cookies instead of Signed URLs

Use Signed Cookies if:
- users download many artifacts in one session
- web app needs access to multiple paths without signing every URL

Signed URLs are best for:
- one-off downloads
- API/CLI usage
- artifact download links emailed to users

---
