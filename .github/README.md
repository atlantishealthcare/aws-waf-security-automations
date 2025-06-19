# 🚀 AWS WAF Automation Deployment via GitHub Actions

This GitHub Actions workflow builds and deploys the [AWS WAF Security Automations](https://docs.aws.amazon.com/solutions/latest/aws-waf-security-automations/solution-overview.html) into a target AWS account using OpenID Connect (OIDC) authentication.

---

## 📋 Prerequisites

### 🔐 IAM Role for GitHub OIDC

You must configure an IAM role in the target AWS account that allows GitHub to assume it using OIDC:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::<account_id>:oidc-provider/token.actions.githubusercontent.com"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
    },
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:<owner>/<repo>:*"
    }
  }
}
````

The role (e.g., `GitHubDeployRole`) needs permissions to:

* Create/put to S3 buckets
* Create/update CloudFormation stacks
* Pass IAM roles if required by the template

---

## 🧱 Architecture Overview

### 📦 S3 Buckets Created per Deployment

```
[ GitHub Workflow Inputs ]
      ├── aws_region: eu-central-1
      ├── bucket_prefix: aws-waf-automation
      └── program_name: my-program

Generated Buckets:
────────────────────────────────────────────────────────────
1. Global template bucket
   └── aws-waf-automation-my-program
        └── [solution]/[version]/aws-waf-security-automations.template

2. Regional asset bucket
   └── aws-waf-automation-my-program-eu-central-1
        └── [solution]/[version]/...

3. Access log bucket
   └── aws-waf-automation-my-program-logs
        └── AWSLogs/...
```

---

### 🌩️ CloudFormation Stack Deployment Flow

```
+--------------------------+
|  GitHub Actions Runner   |
+-----------+--------------+
            |
            v
  Assume Role into AWS via OIDC
            |
            v
+--------------------------+
| Build & Package Solution |
+--------------------------+
            |
            v
+--------------------------+
|  Upload Assets to S3     |
+--------------------------+
      ├── Global → template bucket
      └── Regional → dist bucket
            |
            v
+------------------------------+
| Create/Update CloudFormation |
| Stack: waf-security-...     |
+------------------------------+
```

---

## 🚦 Triggering the Workflow

Triggered manually via **GitHub UI**.

### 🔧 Input Parameters

| Name            | Description                                        | Required | Default              |
| --------------- | -------------------------------------------------- | -------- | -------------------- |
| `aws_region`    | AWS region to deploy                               | ✅        | `eu-central-1`       |
| `account_id`    | Target AWS Account ID                              | ✅        |                      |
| `bucket_prefix` | Prefix for S3 buckets (e.g., `aws-waf-automation`) | ✅        | `aws-waf-automation` |
| `program_name`  | Suffix to uniquely identify this deployment        | ✅        |                      |

---

## 🏗️ What It Does

1. Authenticates with the target AWS account via OIDC.
2. Creates S3 buckets if they don’t exist.
3. Runs a custom `build-s3-dist.sh` script to package the solution.
4. Uploads global and regional assets to S3.
5. Deploys (or updates) a CloudFormation stack named `waf-security-automations`.

---

## 📂 Project Layout

```
.
├── .github/workflows/deploy-waf.yml  # GitHub Actions workflow
├── deployment/
│   ├── build-s3-dist.sh              # Packaging script
│   ├── global-s3-assets/             # CFN template & assets
│   └── regional-s3-assets/           # Lambda code, WAF configs
└── README.md                         # This file
```

---

## 🔎 Troubleshooting

### ❌ "Could not assume role with OIDC"

* Ensure `id-token: write` is set in `permissions`.
* Validate IAM trust policy for OIDC (audience and sub).
* Confirm the role name and repo match.

### ❌ S3 or CFN errors

* Confirm role has permissions: `s3:*`, `cloudformation:*`, and `iam:PassRole` if needed.
* Check if buckets already exist and are owned by your AWS account.

---

## 🧹 Cleanup

To delete the deployment:

```sh
aws cloudformation delete-stack --stack-name waf-security-automations
```

Then remove the S3 buckets manually or via script.

---

## 🧾 License

This repository builds on the official AWS WAF Security Automations solution. Refer to its [license](https://github.com/awslabs/aws-waf-security-automations/blob/master/LICENSE.txt) for details.

---
