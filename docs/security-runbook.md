# Security Runbook — IAM Least-Privilege Architecture with ABAC

**Author:** Muhammad Ahmad  
**Date:** June 2026  
**Account Type:** AWS Free Tier  
**Project:** Cloud Security & IAM — Project 01 of 7

---

## 1. Objective

Design and implement a zero-trust IAM architecture on AWS that:
- Enforces least-privilege access for all identities
- Uses attribute-based access control (ABAC) for scalable authorization
- Prevents privilege escalation via permission boundaries
- Enforces organization-wide guardrails via SCPs
- Automatically detects policy misconfigurations via IAM Access Analyzer

---

## 2. Identity Design Decisions

### 2.1 Why Groups Instead of Direct Policy Attachment?

Policies are attached to **groups**, not users directly. This means:
- Adding a new developer = add them to `dev-team`, permissions follow automatically
- Removing access = remove from group, no per-user cleanup needed
- Audit is simple: group membership = permission set

**Exception:** The `admin-muhammad` user has `AdministratorAccess` attached directly. This is intentional — it is a personal admin account, not a job-function user. One person, one account, explicit attachment.

### 2.2 Why No Wildcard Actions?

Every policy in this project uses **explicit actions** (e.g. `s3:GetObject`, `s3:ListBucket`) rather than `s3:*`. This is the core of least-privilege:

- `s3:*` grants 100+ S3 actions including `s3:DeleteBucket`, `s3:PutBucketPolicy`, etc.
- Our dev team only needs to read and write objects — so that is all they get
- This limits blast radius if credentials are compromised

---

## 3. ABAC Design

### 3.1 Why ABAC Over RBAC?

Traditional RBAC (role-based) requires a new policy for every new team/environment combination:
- `DevPolicy`, `StagingPolicy`, `ProdPolicy`, `DevStagingPolicy`... 

ABAC uses **tags** as the authorization mechanism:
```
User tag: team=dev
Resource tag: team=dev
Policy condition: resource.team == principal.team → ALLOW
```

Adding a new team = tag their users and resources with `team=newteam`. No new policies needed.

### 3.2 Tag Schema

| Tag Key | Values | Purpose |
|---|---|---|
| `team` | dev, ops, security | Primary ABAC attribute |
| `env` | dev, staging, prod | Environment isolation |
| `cost-center` | cc-100, cc-200 | Billing attribution |

---

## 4. Permission Boundaries — Threat Model

**Threat:** A developer with IAM permissions could create a new role and attach `AdministratorAccess` to it, escalating their own privileges.

**Defense:** `DeveloperPermissionBoundary` is set on all developer roles. Even if a policy with `AdministratorAccess` is attached, the **effective permissions are the intersection** of the policy and the boundary.

```
Effective = AttachedPolicy ∩ PermissionBoundary
```

The boundary only allows S3, EC2 Describe, and CloudWatch. A developer can never escape this ceiling.

---

## 5. SCP Design

### 5.1 Why These Two Denies?

The `PreventSecurityToolsTampering` SCP targets the two most common attacker moves after gaining initial access to an AWS account:

1. **Disable CloudTrail** — attackers delete audit logs to hide their activity
2. **Disable GuardDuty** — attackers turn off threat detection to operate undetected

By denying these at the **organization level**, even a fully compromised account admin cannot cover their tracks.

### 5.2 SCP Evaluation Logic

```
Final Decision = SCP_Allows AND IAM_Allows AND NOT SCP_Denies
```

An SCP Deny always wins. There is no way to override it from within the account.

---

## 6. IAM Access Analyzer

### 6.1 What It Does

Access Analyzer continuously scans resource-based policies (S3, IAM roles, KMS keys, etc.) and generates a **finding** whenever a resource is accessible from outside the zone of trust (our account).

### 6.2 Test Performed

Created a public S3 bucket policy granting `s3:GetObject` to `Principal: "*"`. Access Analyzer flagged it within ~5 minutes as an **active finding** with `Public access: Allow`.

After removing the public policy and re-enabling Block Public Access, the finding resolved automatically.

### 6.3 Why This Matters

Access Analyzer essentially acts as a **continuous policy audit** — it catches mistakes that humans miss, like accidentally leaving a bucket public or creating an overly permissive cross-account role.

---

## 7. Cleanup Checklist

- [x] Delete test S3 bucket (`abac-test-muhammad`)
- [x] Re-enable Block Public Access on all S3 buckets
- [x] Confirm Access Analyzer finding resolved
- [ ] Remove test IAM users if no longer needed
- [ ] Keep SCP and Access Analyzer active (free, low maintenance)

---

## 8. Interview Talking Points

**Q: How do you prevent privilege escalation in IAM?**  
A: Permission boundaries on all developer roles. The boundary defines the absolute maximum — even if someone attaches AdministratorAccess, the effective permissions are capped.

**Q: What's the difference between ABAC and RBAC in AWS?**  
A: RBAC uses static role/group membership. ABAC uses tags evaluated at request time. ABAC scales better — no policy explosion as teams grow.

**Q: How do you prevent an attacker from covering their tracks in AWS?**  
A: SCPs at the organization level that deny CloudTrail deletion and GuardDuty disablement. These cannot be overridden even by account admins.

**Q: What does IAM Access Analyzer do?**  
A: Continuously scans resource-based policies and flags any that allow access from outside the account. Think of it as automated policy audit running 24/7.

---

*This runbook is part of the 7-project AWS Cloud Security series.*
