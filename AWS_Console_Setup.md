# AWS Console Setup Guide 🌐

This document outlines the **Phase 1 Infrastructure Provisioning Steps** required to configure your multi-account AWS environment before deploying the application code.

**Multi-Account Reference Architecture Example**
To keep this guide production-secure while remaining easy to follow, we use the following dummy AWS Account IDs throughout the setup. Map these placeholders to your actual AWS Organization deployment IDs:

- **Master Management Account:** 111122223333
- **Child Account 1:** 444455556666
- **Child Account 2:** 777788889999
- **Child Account 3:** 123456789012

---

## SECTION 1 — MASTER ACCOUNT CONFIGURATION (111122223333)

Execute these steps while authenticated into your root **Master Management Account**.

### Step 1 — Centralized S3 Storage Configuration

1. Navigate to **Amazon S3** $\rightarrow$ Click **Create bucket**.
2. **Bucket name:** `iam-guardian-master-bucket`
3. **AWS Region:** `us-east-1`
4. **Block all public access:** Ensure this checkbox is **ON** (Checked).
5. Click **Create bucket**.

Once created, navigate to **S3** $\rightarrow$ **`iam-guardian-master-bucket`** $\rightarrow$ **Permissions** tab $\rightarrow$ **Bucket policy** $\rightarrow$ Click **Edit**, paste the following multi-tenant write policy, and click **Save changes**:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowChildAccountWrite",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::123456789012:role/IAMGuardianScanRole",
                    "arn:aws:iam::777788889999:role/IAMGuardianCollector-role-3h1tnig2",
                    "arn:aws:iam::777788889999:role/IAMGuardianScanRole",
                    "arn:aws:iam::444455556666:role/IAMGuardianScanRole",
                    "arn:aws:iam::123456789012:role/IAMGuardianCollector-role-n8133mdq",
                    "arn:aws:iam::444455556666:role/IAMGuardianCollector-role-038bmes6"
                ]
            },
            "Action": [
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "arn:aws:s3:::iam-guardian-master-bucket/*"
        }
    ]
}

```

### Step 2 — Provision the EC2 Server Control IAM Role

1. Navigate to **IAM** $\rightarrow$ **Roles** $\rightarrow$ Click **Create role**.
2. **Trusted entity type:** Select **AWS Service**.
3. **Use case:** Select **EC2**.
4. Click **Next** (Skip attaching managed permissions for now).
5. **Role name:** `IAMGuardianEC2Role`
6. Click **Create role**.

Select your newly created `IAMGuardianEC2Role` and add the following four **Inline Policies** by clicking **Add permissions** $\rightarrow$ **Create inline policy** $\rightarrow$ switching to the **JSON** tab:

#### Inline Policy 1: `AssumeChildRoles`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": [
                "arn:aws:iam::444455556666:role/IAMGuardianScanRole",
                "arn:aws:iam::777788889999:role/IAMGuardianScanRole",
                "arn:aws:iam::123456789012:role/IAMGuardianScanRole"
            ]
        }
    ]
}

```

#### Inline Policy 2: `MasterS3Access`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::iam-guardian-master-bucket",
                "arn:aws:s3:::iam-guardian-master-bucket/*"
            ]
        },
        {
            "Sid": "BedrockAccess",
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel",
                "bedrock:InvokeModelWithResponseStream"
            ],
            "Resource": "*"
        }
    ]
}

```

#### Inline Policy 3: `InvokeLambda`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "lambda:InvokeFunction",
            "Resource": [
                "arn:aws:lambda:us-east-1:444455556666:function:IAMGuardianCollector",
                "arn:aws:lambda:us-east-1:777788889999:function:IAMGuardianCollector",
                "arn:aws:lambda:us-east-1:123456789012:function:IAMGuardianCollector"
            ]
        }
    ]
}

```

#### Inline Policy 4: `ReadOwnIAM`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:ListRoles",
                "iam:ListUsers",
                "iam:ListGroups",
                "iam:ListPolicies",
                "iam:ListRolePolicies",
                "iam:ListUserPolicies",
                "iam:ListGroupPolicies",
                "iam:ListAttachedRolePolicies",
                "iam:ListAttachedUserPolicies",
                "iam:ListAttachedGroupPolicies",
                "iam:GetRole",
                "iam:GetUser",
                "iam:GetPolicy",
                "iam:GetPolicyVersion",
                "iam:GetRolePolicy",
                "iam:GetUserPolicy"
            ],
            "Resource": "*"
        }
    ]
}

```

### Step 3 — Launch the EC2 Host Server

1. Navigate to **EC2** $\rightarrow$ Click **Launch instance**.
2. **Name:** `iam-guardian-server`
3. **Application and OS Image:** Select **Ubuntu** $\rightarrow$ **Ubuntu Server 22.04 LTS**.
4. **Instance type:** Select **`t2.micro`** (Free Tier eligible).
5. **Key pair:** Select an existing key or click **Create new key pair** to download your entry `.pem` token file securely.
6. **Network settings (Security Group):** Configure inbound firewall pathways to allow the following traffic:
* **SSH (Port 22):** Remote CLI server management
* **HTTP (Port 80):** Web delivery layers
* **Custom TCP (Port 3000):** React Application Interface
* **Custom TCP (Port 8000):** FastAPI Backend Microservice API Gateway


7. Click **Launch instance**.

#### Attach the IAM Profile Engine to the Host Instance:

Navigate back to **EC2** $\rightarrow$ Select your running `iam-guardian-server` instance $\rightarrow$ Click **Actions** $\rightarrow$ **Security** $\rightarrow$ **Modify IAM role** $\rightarrow$ Select **`IAMGuardianEC2Role`** $\rightarrow$ Click **Update IAM role**.

> *Note down your EC2 instance's **Public IPv4 address** from the console summary—this value maps your automated frontend variables during Phase 2 deployment.*

---

## SECTION 2 — MEMBER CHILD ACCOUNT PROVISIONING

> **Prerequisite Execution Rule:** Repeat this section sequentially for each child member account partition inside your enterprise topology: **`444455556666`**, **`777788889999`**, and **`123456789012`**.

### Step 4 — Establish the Cross-Account Trust Security Role

1. Switch into your child member account target via your IAM Identity Center portal window.
2. Navigate to **IAM** $\rightarrow$ **Roles** $\rightarrow$ Click **Create role**.
3. **Trusted entity type:** Select **AWS account** $\rightarrow$ Select **Another AWS account**.
4. **Account ID:** Provide the Master Root ID: `111122223333`.
5. Click **Next**.
6. **Role name:** `IAMGuardianScanRole`
7. Click **Create role**.

Select your newly created `IAMGuardianScanRole` $\rightarrow$ Click **Add permissions** $\rightarrow$ **Create inline policy** $\rightarrow$ switch to the **JSON** tab, paste this configuration, and name it `IAMGuardianScanPolicy`:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ReadIAMPolicies",
            "Effect": "Allow",
            "Action": [
                "iam:ListRoles",
                "iam:ListUsers",
                "iam:ListGroups",
                "iam:ListPolicies",
                "iam:ListRolePolicies",
                "iam:ListUserPolicies",
                "iam:ListGroupPolicies",
                "iam:ListAttachedRolePolicies",
                "iam:ListAttachedUserPolicies",
                "iam:ListAttachedGroupPolicies",
                "iam:GetRole",
                "iam:GetUser",
                "iam:GetPolicy",
                "iam:GetPolicyVersion",
                "iam:GetRolePolicy",
                "iam:GetUserPolicy"
            ],
            "Resource": "*"
        },
        {
            "Sid": "SendToMasterS3",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "arn:aws:s3:::iam-guardian-master-bucket/*"
        },
        {
            "Sid": "CreateDemoRoles",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:PutObjectAcl",
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:PutRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:TagRole"
            ],
            "Resource": "arn:aws:iam::*:role/Demo-*"
        },
        {
            "Sid": "AllowInvokeCollectorAnyAccount",
            "Effect": "Allow",
            "Action": "lambda:InvokeFunction",
            "Resource": "arn:aws:lambda:*:*:function:IAMGuardianCollector*"
        }
    ]
}

```

### Step 5 — Deploy the Serverless Audit Data Collector

1. Navigate to **AWS Lambda** $\rightarrow$ Click **Create function**.
2. **Function name:** `IAMGuardianCollector`
3. **Runtime:** Select **Python 3.11**.
4. **Architecture:** Select **x86_64**.
5. **Permissions:** Choose **Create a new execution role** $\rightarrow$ Name it sequentially based on the account profile template (e.g., `IAMGuardianCollector-role-3h1tnig2`).
6. Click **Create function**.

#### Source Code Synchronization:

* In the Lambda developer context console workspace, navigate to the **Code** tab.
* Open `lambda_function.py`, purge default placeholder templates, and replace it with your clean repository code asset from `lambda/iam_collector_lambda.py`.
* Click **Deploy**.

#### Operational Optimization Tweaks:

* Navigate to **Configuration** tab $\rightarrow$ **General configuration** $\rightarrow$ Click **Edit**.
* Change **Timeout:** `5 minutes` (`300 seconds`).
* Change **Memory:** `256 MB`.
* Click **Save**.

#### Attach Policies to the Lambda Execution Profile:

1. Navigate to **Configuration** tab $\rightarrow$ **Permissions** panel $\rightarrow$ Click the active text link under the **Role name** header to open the role in the IAM console.
2. Click **Add permissions** $\rightarrow$ **Attach policies** $\rightarrow$ Search and add **`AWSLambdaBasicExecutionRole`**.
3. Click **Add permissions** $\rightarrow$ **Create inline policy** $\rightarrow$ Switch to the **JSON** tab, paste the following policy to permit local IAM auditing and central S3 delivery, and name it `LambdaCollectorPolicy`:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:ListRoles",
                "iam:ListUsers",
                "iam:ListGroups",
                "iam:ListPolicies",
                "iam:ListRolePolicies",
                "iam:ListUserPolicies",
                "iam:ListGroupPolicies",
                "iam:ListAttachedRolePolicies",
                "iam:ListAttachedUserPolicies",
                "iam:ListAttachedGroupPolicies",
                "iam:GetRole",
                "iam:GetUser",
                "iam:GetPolicy",
                "iam:GetPolicyVersion",
                "iam:GetRolePolicy",
                "iam:GetUserPolicy"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "arn:aws:s3:::iam-guardian-master-bucket/*"
        }
    ]
}

```

#### Enforce Cross-Account Resource Invocations (Resource-Based Policy):

Return to your **Lambda Function** workspace panel $\rightarrow$ **Configuration** tab $\rightarrow$ **Permissions** sidebar selection $\rightarrow$ Scroll down to **Resource-based policy statements** $\rightarrow$ Click **Add permission**:

* **Service boundary selection:** Select **AWS Account**.
* **Statement ID:** `AllowMasterAccountInvoke`
* **Principal ARN:** `arn:aws:iam::111122223333:role/IAMGuardianEC2Role`
* **Action:** `lambda:InvokeFunction`
* Click **Save**.

*(Note: The underlying resource JSON matching this structural configuration resolves as follows for Child Account 2)*:

```json
{
  "Version": "2012-10-17",
  "Id": "default",
  "Statement": [
    {
      "Sid": "AllowMasterAccountInvoke",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:role/IAMGuardianEC2Role"
      },
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:us-east-1:777788889999:function:IAMGuardianCollector"
    }
  ]
}

```

---

### Next Step

Once you have repeated Section 2 across all targeted member environments, your AWS backbone network layout is entirely stable. You can now transition directly to next step! [Check Readme.md]
