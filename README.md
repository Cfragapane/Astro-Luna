# 🚀 Secure CI/CD Pipelines with AWS OIDC

This repository demonstrates how to securely deploy an **Astro website** to AWS S3 static hosting using **GitHub Actions** and **AWS IAM OIDC (OpenID Connect)**. The core objective of this architecture is to completely eliminate the need for long-lived, static AWS access keys inside GitHub Secrets.

---

## 🎯 Architecture Overview & The OIDC Approach

Traditional CI/CD pipelines often rely on permanent AWS Access Keys stored as GitHub Secrets. This project implements a modern **OIDC-based federated authentication** mechanism, offering significant security enhancements:

| Feature | The Classic Approach ⚠️ | The OIDC Approach ✅ |
| :--- | :--- | :--- |
| **Secret Storage** | Access key + secret stored in GitHub. | **No secrets** stored in GitHub – nothing to steal! |
| **Credential Lifetime** | Static and long-lived (no auto-rotation). | **Short-lived tokens** (minutes), issued automatically. |
| **Blast Radius** | If leaked: full access until manually revoked. | Trust policy binds access **exactly to repo + branch**. |
| **Auditability** | No direct link to "which workflow used this". | Full audit trail visible via **AWS CloudTrail**. |
| **Scalability** | Must be manually distributed to every account. | One-time setup per account, then "set and forget". |

### 🔄 The Core Deployment Flow
```text
[ Push to main ] ──► [ GitHub Actions Workflow ] ──► [ AWS STS Token Exchange ] ──► [ S3 Bucket Deploy ]
   (Developer)           (Requests ID Token)            (Checks Trust Policy)        (Serves Static Site)
```
> **How the Trust Policy Works:** AWS validates the incoming OpenID Connect token and dynamically checks: *"Does this token really come from exactly this repository and this specific branch?"* Temporary, short-lived credentials are issued **only** if this validation succeeds.

---

## 🧱 The Four Building Blocks

The security architecture of this deployment relies on four core components configured in AWS and GitHub:

1. **OIDC Identity Provider:** Registers GitHub as a trusted token issuer inside your AWS root account.
2. **Trust Policy:** Defines precisely *which* GitHub repository and *which* Git branch is legally allowed to assume the IAM role.
3. **Permission Policy:** Enforces the *Principle of Least Privilege* – restricting the workflow's permissions exclusively to write access on the target S3 bucket.
4. **Workflow File:** Configures the GitHub Actions runner to request the OIDC token and execute the secure handshake with AWS.

---

## 🛠️ Local Setup & Production Testing

Before deploying, you can build and test your Astro project locally to ensure environmental parity.

### 1. Initialize the Astro Project
```bash
# Create a new project using the latest Astro build template
npm create astro@latest

# Follow the interactive prompts to install dependencies and initialize git
cd your-project-folder

# Start the local development server
npm run dev
```
*Your development environment is now live at `http://localhost:4321`.*

### 2. Test the Production Build
> ⚠️ **Key Insight:** `npm run dev` is strictly for local engineering. Amazon S3 serves completely static assets compiled during the production build.

```bash
# Compile the static production assets
npm run build

# Preview the compiled production build locally
npm run preview
```
*The build command outputs all static distribution files into the local `dist/` directory. It is this folder that is synchronized onto your AWS S3 bucket.*

---

## 🛡️ Learning Goals & Practical Scenarios

This project serves as a real-world blueprint to master the following competencies:
* **Enterprise IAM Security Concepts:** Move beyond theory by applying role-based condition logic and strict least-privilege scoping.
* **GitHub Actions Hardening:** Master pipeline syntax, security scoping via workflow level `permissions`, and deep log analysis.
* **Federated Authentication:** Understand the underlying mechanics of token-exchange architecture—a pattern seamlessly transferable to other CI/CD systems like GitLab CI or CircleCI.

### 🧪 Advanced Practice Scenario (Stretch Goal)
To fully experience why scoping matters, deliberately loosen your AWS Trust Policy to allow all branches, trigger a workflow from a non-production branch, and then re-tighten it. You will see firsthand how AWS handles boundary violations.

---

## 🔍 Troubleshooting & Common Pitfalls

If your pipeline fails, verify these common configuration errors:

* **`An error occurred (AuthFailure) / Not authorized to perform sts:AssumeRoleWithWebIdentity`**
  * *Cause:* Your AWS IAM Trust Policy contains a typo or references the wrong GitHub repository organization, project name, or branch.
* **OIDC token isn't requested at all**
  * *Cause:* Your GitHub Action `.github/workflows/deploy.yml` is missing the explicit permission scope block:
    ```yaml
    permissions:
      id-token: write
      contents: read
    ```
* **`Access Denied` during S3 sync**
  * *Cause:* Your AWS IAM Permission Policy is too restrictive and does not grant all required actions (such as `s3:PutObject`, `s3:ListBucket`, or `s3:DeleteObject`).
* **Website shows `403 Forbidden` in the browser**
  * *Cause:* The static S3 bucket hosting is misconfigured. Ensure that *Block Public Access* is disabled or an appropriate Public Bucket Policy is applied.
