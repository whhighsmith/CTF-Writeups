# TryHackMe - Complimentary Writeup
**Event:** Byte Lotus  
**Room:** Complimentary  
**Difficulty:** Beginner/Intermediate  
**Tools Used:** Browser Developer Tools, AWS CLI  
**Platform:** TryHackMe AttackBox  

---

## Introduction

Complimentary is a CTF challenge from the TryHackMe Byte Lotus event. The scenario involves a wellness app called "Byte Lotus Wellness" that offers complimentary access with no login required. The objective was to discover how the app authenticates users behind the scenes, leverage those credentials to access a cloud database, and retrieve a flag hidden in another guest's data.

The core vulnerabilities exploited were:
- **Exposed AWS Cognito credentials** leaked through unauthenticated identity pools
- **Insecure Direct Object Reference (IDOR)** via misconfigured DynamoDB table permissions

---

## Reconnaissance

Upon loading the app, I opened **Browser Developer Tools** (`F12`) and navigated to the **Network tab** to monitor outgoing requests.

I immediately noticed a request going out to:
```
cognito-identity.amazonaws.com
```

This revealed that the app was using **AWS Cognito** to silently issue temporary AWS credentials to unauthenticated users — no login required.

---

## Extracting AWS Credentials

By inspecting the **Response** of the Cognito network request in Developer Tools, I found three temporary AWS credentials:

```
AccessKeyId:     <REDACTED>
SecretKey:       <REDACTED>
SessionToken:    <REDACTED>
```

I also noted the following details from the request:
- **Region:** `us-east-1`
- **Cognito Identity Pool ID:** `us-east-1:4d571309-b086-c37a-eafc-2bc4c07558d1`

---

## Configuring AWS CLI

I exported the credentials as environment variables to authenticate the AWS CLI:

```bash
export AWS_ACCESS_KEY_ID=<AccessKeyId>
export AWS_SECRET_ACCESS_KEY=<SecretKey>
export AWS_SESSION_TOKEN=<SessionToken>
```

I verified the credentials were working with:
```bash
aws sts get-caller-identity
```

This confirmed I was authenticated as:
```
arn:aws:sts::332173347248:assumed-role/complimentary-cognito-unauth-role/CognitoIdentityCredentials
```

---

## Enumerating DynamoDB

I attempted to list all DynamoDB tables:
```bash
aws dynamodb list-tables --region us-east-1
```

This returned an `AccessDeniedException` — the unauthenticated Cognito role didn't have permission to list tables. However, this didn't mean I couldn't access a specific table directly.

I went back to the app's source code in Browser Developer Tools under the **Sources tab** and searched for references to `dynamodb` and `table`. I located the target table name hardcoded in the application's JavaScript.

---

## Dumping the DynamoDB Table

With the table name in hand, I ran a full table scan:
```bash
aws dynamodb scan --table-name <tablename> --region us-east-1
```

This returned records for **all guests** in the database — not just my own. This is the IDOR vulnerability: the unauthenticated Cognito role had scan permissions on the entire table with no per-user access controls in place.

To quickly locate the flag I filtered the output:
```bash
aws dynamodb scan --table-name <tablename> --region us-east-1 | grep -i "flag\|THM\|secret"
```

---

## Flag

The flag was found embedded in another guest's record in the DynamoDB table.

---

## Key Takeaways

- **AWS Cognito unauthenticated identity pools** can expose temporary AWS credentials to anyone who opens the app — even without a login. These credentials should be tightly scoped to only what is absolutely necessary.
- **Hardcoded resource names** in client-side JavaScript (like DynamoDB table names) give attackers a roadmap to your backend infrastructure.
- **DynamoDB table permissions** should enforce per-user access controls. A user should only ever be able to read their own record, not scan the entire table.
- **IDOR vulnerabilities exist at the cloud infrastructure level**, not just in web applications. Misconfigured IAM roles are one of the most common real-world cloud security issues.
- **Browser Developer Tools** are an incredibly powerful reconnaissance tool — network requests can reveal entire backend architectures.

---

## Tools Reference

| Tool | Purpose |
|---|---|
| Browser Developer Tools | Intercepting network requests and extracting credentials |
| AWS CLI | Authenticating with extracted credentials and querying DynamoDB |

---

## Attack Chain Summary

```
App loads → Cognito issues temporary credentials → Credentials intercepted in Network tab
→ AWS CLI configured with credentials → Table name found in JS source
→ DynamoDB scan dumps all guest records → Flag retrieved from another guest's data
```

---

*Writeup by Will | TryHackMe: willyh
