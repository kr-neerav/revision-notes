# AWS IAM: Production Revision Notes

## 1. The API Boundary and Principals
Every action in AWS is an authenticated HTTP request evaluated against a **Request Context** (Principal, Action, Resource, Conditions).

* **Principals (The "Who"):** Only three exist—Root, IAM Users, and IAM Roles.
* **The Group Trap:** IAM Groups are administrative buckets, *not* cryptographic identities. You cannot use a Group as a Principal in a resource-based policy.

## 2. Policy Evaluation Logic
IAM relies on a strict, deterministic evaluation engine. 

| Evaluation Order | Rule | Impact |
| :--- | :--- | :--- |
| **1. Implicit Deny** | Default state of AWS. | If no policy says "Allow", the request drops. |
| **2. Explicit Allow** | Must be explicitly granted. | Enables the requested action on the resource. |
| **3. Explicit Deny** | **The Golden Rule.** | Instantly and permanently overrides *any* Explicit Allow. |

## 3. IAM Roles and STS
Do not use static long-term credentials (IAM Users) for compute. Use IAM Roles for temporary, auto-expiring credentials via the Security Token Service (STS).

* **Permission Policy:** Dictates what the role can *do* (e.g., read S3).
* **Trust Policy:** Dictates *who* or *what* can assume the role (e.g., EC2, API Gateway).
* **The Confused Deputy:** When allowing AWS services (like EventBridge or API Gateway) to assume a role, always use `Condition` blocks (`aws:SourceAccount`, `aws:SourceArn`) to prevent other AWS accounts from triggering your resources. *Exception:* EC2 uses Instance Profiles, natively enforcing the account boundary.

## 4. Resource-Based vs. Identity-Based Policies
* **Identity-Based:** Attached to the principal ("What can I do?").
* **Resource-Based:** Attached to the resource itself, like an S3 bucket or KMS key ("Who can access me?").

**The Cross-Account Handshake:**
| Scenario | Identity Policy (Acct A) | Resource Policy (Acct B) | Result |
| :--- | :--- | :--- | :--- |
| **Same Account** | Explicit Allow | No Policy (or vice versa) | **Authorized** |
| **Cross-Account** | Explicit Allow | No Policy | **AccessDenied** |
| **Cross-Account** | Explicit Allow | Explicit Allow | **Authorized** |

*Note: Cross-account access requires a strict two-way handshake. Both the sending account and the receiving account must explicitly allow the action.*

## 5. Permission Boundaries and SCPs
These **do not grant access**. They define the *maximum possible permissions* an identity can have.

* **Service Control Policies (SCPs):** Applied at the AWS Organization or Account level. Acts as an invisible ceiling. Even the Root user of an account is blocked by an SCP `Deny`.
* **Permission Boundaries:** Attached directly to an IAM Role or User. Acts as a filter. If a role has AdministratorAccess, but its boundary only allows S3, the maximum permission is S3.
* **Evaluation:** For a request to succeed, the Identity/Resource policy, the Boundary, and the SCP must *all* allow the action (a perfect Venn diagram overlap), with zero Explicit Denys.

## 6. Federation and OIDC
Used to grant external systems (GitHub Actions, Okta, Kubernetes) access to AWS without generating static IAM Access Keys.

* **How it Works:** The external Identity Provider (IdP) hands AWS STS a cryptographically signed token (SAML or OIDC JWT). STS verifies the signature, checks the Trust Policy, and issues temporary credentials.
* **The GitHub Actions Trap:** Because GitHub shares a single OIDC provider globally, you *must* use a `Condition` block in your Trust Policy to restrict the `sub` (subject) claim to your specific repository (`repo:YourOrg/YourRepo:*`). Without this, any GitHub user on the planet could assume your role.