# Comprehensive Revision Notes: Data Depersonalization & Privacy Architecture

## Layer 1: The Threat Model & Data Types

* **Direct Identifiers:** Data that explicitly points to an individual (e.g., Email, SSN, Phone Number).
* **Quasi-Identifiers (QIDs):** Data that is harmless alone but dangerous combined (e.g., Zip Code, Age, Gender). 
* **Linkage Attack:** An attacker combines QIDs from your database with external public datasets (like voter registries) to re-identify anonymized users.

---

## Layer 2: Protecting Direct Identifiers

### 1. Hashing (One-Way Destruction)
Destroys the original value mathematically. Use when you never need the raw data back.
* **Salt (Row-Level):** A unique random value added to each row before hashing. Prevents frequency analysis (stops identical inputs from producing identical hashes).
* **Pepper (Global-Level):** A highly guarded secret key stored in a KMS. Prevents offline dictionary/rainbow-table attacks if the database is stolen.
* **The Deterministic Tradeoff:** To preserve SQL `GROUP BY` and `JOIN` capabilities, you must drop the random row-level Salt and rely solely on the Pepper (HMAC). This restores analytical utility but re-introduces vulnerability to frequency analysis.

### 2. Encryption (Two-Way Lockbox)
Locks the data so it can be reversibly unlocked by downstream operational systems (e.g., billing).
* **Standard:** AES-256 with Envelope Encryption (A Data Encryption Key wrapped by a Key Management System Key).
* **The Analytical Tradeoff:** Encryption uses random Initialization Vectors (IVs), meaning every ciphertext is unique. You cannot run `JOIN`, `GROUP BY`, or aggregations on encrypted columns.

### 3. Tokenization (Infrastructure Swapping)
Swaps real data for random UUIDs using a physically isolated Token Vault microservice.
* **Advantages:** Immune to cryptographic attacks (tokens are random noise), deterministic (preserves analytics), and enables **Crypto-Shredding** (instant GDPR compliance by deleting the token-to-data mapping in the Vault).
* **Disadvantages:** Introduces network latency to ingestion pipelines.

---

## Layer 3: Protecting Quasi-Identifiers

Uses Generalization (bucketing data, like Age 34 $\rightarrow$ 30-39) and Suppression (deleting outliers) to hide individuals in a crowd.

| Guarantee | Mathematical Rule | Vulnerability | Impact on ML Utility |
| :--- | :--- | :--- | :--- |
| **$k$-Anonymity** | Every combination of QIDs matches $\ge k$ rows. | **Homogeneity Attack** (The entire group shares the same sensitive secret). | Low/Medium |
| **$l$-Diversity** | Every $k$-group has $\ge l$ distinct sensitive values. | **Semantic/Probabilistic Attack** (Values mean the same thing, or skew heavily from baseline). | High |
| **$t$-Closeness** | The sensitive distribution in every $k$-group matches the global dataset distribution. | Mathematically airtight. | Severe (often destroys dataset correlations) |

---

## Layer 4: Differential Privacy (DP)

Abandons generalization entirely. Protects QIDs by injecting mathematically calibrated **cryptographic noise** into the data, providing users with plausible deniability while preserving accurate aggregate statistical trends.

* **Constraint:** Only works on numerical aggregates and categorical counts. Cannot be applied to unstructured text or operational data (e.g., stack traces).
* **$\epsilon$ (Epsilon) Privacy Budget:** Determines the amount of noise. High $\epsilon$ = low noise/high utility. Low $\epsilon$ = high noise/strict privacy.

### DP Architectures
* **Local DP:** Noise is injected at the edge/client before hitting the network. Extremely high privacy. **Tradeoff:** Requires massive sample sizes (Law of Large Numbers) to be accurate. Fails at micro-segmentation.
* **Global DP:** Database holds raw, sensitive data. Analysts query through a DP Proxy. The proxy calculates the exact answer, injects noise based on the remaining $\epsilon$ budget, and returns a fuzzy aggregate.

---

## Layer 5: Architectural Implementation (Defense in Depth)

Enterprise data pipelines require applying these techniques at different stages to manage the blast radius:

1.  **Ingest (Shift-Left):** Intercepting data before it hits the database.
    * *Techniques:* Tokenization, Local DP.
    * *Best for:* Radioactive Direct Identifiers (SSNs, Credit Cards) that should never touch analytical storage.
2.  **At-Rest (Storage):** Securing the physical disk/database.
    * *Techniques:* Hashing (Salt + Pepper), Encryption.
    * *Best for:* Standard PII (Emails, Phone numbers). Protects against DB dumps.
3.  **Read-Time (Compute):** Securing data on-the-fly when queried.
    * *Techniques:* Global DP Proxies, RBAC Dynamic Masking, Generalization Views.
    * *Best for:* Internal analysts and ML teams. Highly flexible, but relies on the underlying storage remaining perfectly secured.