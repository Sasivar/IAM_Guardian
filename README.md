# IAM Guardian 

**IAM Guardian** is an enterprise-grade cloud security posture management platform designed to automate multi-account AWS Identity and Access Management (IAM) auditing. Built as an AI Hackathon prototype, the platform orchestrates decentralized serverless data collectors across member accounts and processes complex permissions vectors through a specialized parallel **Claude 3.5 Sonnet** AI engine to deliver instant risk mitigation blueprints.

---

## Core Features

* **Multi-Account Topology Control Panel:** A unified, responsive dashboard providing a high-level operational glance at registered Master and Member accounts.
* **Decentralized Serverless Collection:** Uses low-footprint, event-driven AWS Lambda components (`IAMGuardianCollector`) in member accounts to read local configurations securely without ongoing compute costs.
* **Automated RAG Triage Matrix:** Automatically assesses and categorizes identities into explicit **RED** (Critical), **AMBER** (Moderate), and **GREEN** (Low Risk) status layers.
* **AI-Engineered Self-Healing Blueprints:** Evaluates complex JSON constructs using Claude to provide human-readable risk justifications and dynamically written, syntactically accurate **Least-Privilege JSON replacements**.
* **Executive Report Generation:** Features a dedicated server-side compiler that builds standardized, page-optimized PDF roadmaps containing visual summaries for cross-functional stakeholders.

---

## User Interface Dashboard

As captured in the below image, the platform provides a streamlined **AWS Account Scanner** control panel. Administrators can select from any registered account in the AWS Organization topology and initiate a zero-trust compliance assessment with a single click:

<img width="1916" height="589" alt="image" src="https://github.com/user-attachments/assets/201749fb-4b57-4aea-93c3-480defabfbca" />

---

## Scan Results & AI Analysis

When a scan finishes, IAM Guardian displays deep-dive modal cards (as shown in `image_f2a3da.png`) detailing the exact risks identified by our specialized engine. The platform provides natural language risk assessments, clear risk factors, and copy-paste remediation rules.

![Uploading image.png…]()


### 💡 Core Components of an AI Audit Card:
*   **Target Scope Metadata:** Instantly maps the entity type, name, and operational context (e.g., tracking that the role has **Never** been used actively).
*   **Generative Risk Assessment:** Claude 3.5 Sonnet contextualizes the technical debt—explaining exactly why combining wildcard actions (`ec2:*`) with wildcard resources (`*`) on a dormant credential invites significant privilege escalation surfaces.
*   **Granular Risk Factors:** Translates JSON statements into plain-English operational warnings (e.g., flags missing MFA conditions for high-impact actions like instance termination).

---

## Architectural Blueprint

The platform implements a highly isolated, cross-account extraction execution model across an AWS Organization topology:

```
[ Web UI (React) ] 
       │  ▲
       │  │ (4-Second Async Polling Loop)
       ▼  │
[ FastAPI Backend (EC2) ]
       │
       ├─► (Secure Invocation Handshake) ──► [ Member Account (AWS Lambda) ]
       │                                                     │
       │                                                     ▼ (Reads Local IAM Metadata)
       │                                           [ Collects JSON Policies ]
       │                                                     │
       │                                                     ▼ (Cross-Account Push)
[ Ingests JSON Payload ] ◄──────────────────────── [ Master Amazon S3 Bucket ]
       │
       ▼ (Async Semaphore Queue: Limit 3)
[ Claude 3.5 Sonnet API ]
       │
       ├─► Validates Strict JSON Structure Response Schema
       ├─► Populates State Machine & RAG Matrix Dashboard
       └─► Renders High-Fidelity PDF Execution Report

```

### Infrastructure & Operational Boundaries

1. **Account A (Master Root):** Hosts the decoupled React web client and FastAPI server on an optimized Ubuntu instance profile environment. Manages centralized S3 storage repositories and external LLM communication brokers.
2. **Account B (Target Partitions / Members):** Hosts the passive data source infrastructure. The execution role has **absolutely zero write permissions** over the child environment, enforcing a strict read-only security sandbox (`iam:List*`, `iam:Get*`).

---

## Automated Deployment & Setup (EC2)

We provide a fully automated initialization script to provision dependencies, build the frontend package ecosystem, and launch your microservices on a fresh **Ubuntu 22.04** cloud node.

### Prerequisites & Security Requirements

* **Host OS:** Fresh AWS EC2 instance running **Ubuntu 22.04 LTS** (e.g., `t2.micro` or `t3.small`).
* **IAM Instance Profile:** The EC2 host instance must be assigned an AWS IAM Role providing read/write permissions over your centralized deployment bucket (`iam-guardian-master-bucket`).
* **Inbound Security Group Rules:**
* `Port 22` (SSH Access)
* `Port 3000` (React UI Dashboard Interface)
* `Port 8000` (FastAPI Production Endpoint Gateway)



### Step-by-Step Execution Guide

1. **Access Server & Create Script File:**
Connect via SSH to your clean EC2 instance and spawn a deployment script file:
```bash
sudo nano installation.sh

```


*Paste the complete bash automation script contents directly into the file window, save (`Ctrl+O`), and exit (`Ctrl+X`).*
2. **Verify Target Environment Flags:**
Ensure lines 33–34 match your exact active AWS resource layout definitions:
```bash
GITHUB_REPO_URL="https://github.com/Sasivar/IAM_Guardian.git"
MASTER_BUCKET="iam-guardian-master-bucket"
AWS_REGION="us-east-1"

```


3. **Elevate Permissions & Launch:**
Provide executable execution flags and invoke the automated sequence as root:
```bash
chmod +x installation.sh
sudo ./installation.sh

```


4. **Provide Runtime Token:**
The installation sequence will safely pause and wait for your input. Input your **Anthropic API Secret Key**:
```text
====================================================
               ANTHROPIC API KEY                    
====================================================

Enter your ANTHROPIC_API_KEY: sk-ant-api03-...

```



---

### Under-The-Hood Deployment Sequence

Once credentials are inputs, the script redirects logs to `/var/log/iam-guardian-setup.log` and completely automates the remaining steps:

1. **Dependency Ingestion:** Running system upgrades and provisioning core execution packages (`python3-pip`, `python3-venv`, `git`, `curl`).
2. **Node Engine Bootstrapping:** Installs **Node.js v20** and matching NPM packages via NodeSource distribution hooks.
3. **Environment Isolation:** Provisions an isolated Python virtual environment (`venv`) and installs production modules (`fastapi`, `uvicorn`, `boto3`, `python-dotenv`, `reportlab`, `anthropic`).
4. **Code Ingestion & Dynamic Asset Compilation:** Clones the code tree to `/home/ubuntu/iam-guardian`, runs a custom inline python script to update hardcoded API target lines inside client modules (`App.js`) with the newly fetched public IP, and compiles production UI distributions (`npm run build`).
5. **Daemon Management:** Deploys services using persistent `nohup` processes:
* **FastAPI application layer** is mapped to port `8000`.
* **React static bundle build** is served via an unblocked Python HTTP thread on port `3000`.



---

### Monitoring & Log Locations

Verify active operational runtimes or parse error loops using the following system targets:

* **Platform Dashboard Console:** `http://<YOUR_EC2_PUBLIC_IP>:3000`
* **Swagger API Documentation Hub:** `http://<YOUR_EC2_PUBLIC_IP>:8000/docs`
* **Infrastructure Installation Logs:** `tail -f /var/log/iam-guardian-setup.log`
* **FastAPI Execution Diagnostics:** `tail -f /home/ubuntu/backend.log`
* **Frontend Network Traffic Logs:** `tail -f /home/ubuntu/frontend.log`

---

## License

Distributed under the MIT License. See `LICENSE` for more information.
