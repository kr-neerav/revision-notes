# Revision Notes: Access Control in Batch Data Architecture

These notes summarize the architectural patterns and operational standards for securing decoupled batch data environments (e.g., S3/GCS paired with Spark/Trino).

---

## 1. Storage-Level vs. Compute-Level Authorization
In batch architectures, physical storage and query compute are separated. Security must reflect this split to prevent users from bypassing logical access controls.

* **The Golden Rule:** The compute engine (e.g., Trino, Spark) must be the *only* entity with storage-level read access (cloud IAM) to raw data buckets. 
* **Human Access:** Human users are granted access *solely* at the compute/logical layer, where policies can be actively enforced.

| Feature | Monolithic Database | Decoupled Batch |
| :--- | :--- | :--- |
| **Data Custodian** | Database engine | Object storage (S3/GCS) |
| **Granularity** | Logical (Tables, rows, columns) | Split: Physical (Files) + Logical (Tables) |
| **Primary Risk** | SQL injection, weak passwords | Bypassing compute to read raw storage files |

---

## 2. Identity & Service Principals (Workload Identity)
Hardcoding static credentials for machines creates massive blast radii and operational bottlenecks. 

* **Compute-Bound Identity:** Bind access rights to the lifecycle of the compute resource itself (e.g., EC2 Instance Profiles), not to humans or orchestrators like Airflow.
* **Ephemeral Tokens:** Machines dynamically authenticate to the cloud provider to receive short-lived, expiring tokens (STS) rather than using infinite-lifespan static keys.
* **Orchestrator Security:** Airflow should trigger jobs, but should never hold the keys to the underlying data. 

---

## 3. Metadata Catalogs & Role-Based Access Control (RBAC)
Managing physical IAM policies for hundreds of datasets hits cloud limits and drains engineering time. 

* **Logical Translation:** Metadata catalogs (e.g., Unity Catalog, AWS Lake Formation) map physical S3 file paths to logical structures (Databases, Schemas, Tables).
* **SQL-Based Governance:** Data stewards manage access via standard SQL (`GRANT SELECT ON TABLE...`) instead of modifying JSON cloud infrastructure policies.
* **Decoupling:** Shifts access control from DevOps/Cloud Infrastructure teams to Data teams.

---

## 4. Fine-Grained Access Control (FGAC)
Creating physically distinct copies of data for different compliance requirements (e.g., a "masked" table and a "raw" table) causes cost overruns and metric drift.

* **Zero Duplication:** Maintain a single physical source of truth and apply dynamic policies at query time.
* **Column-Level Security (CLS):** Dynamically masks, hashes, or nullifies specific sensitive columns (like PII) based on the user's role.
* **Row-Level Security (RLS):** Transparently appends `WHERE` filters to queries, ensuring users only see authorized rows (e.g., doctors only seeing their own patients).

---

## 5. Attribute-Based Access Control (ABAC)
RBAC fails at enterprise scale because it requires manual policy creation for every new table. ABAC automates this using metadata tags.

* **Tag-Driven:** Policies target metadata tags (e.g., `tag: PII` or `tag: GDPR`) rather than physical table names.
* **Global Enforcement:** A single policy ("Mask `PII` for all users without clearance") applies instantly across the entire data lake, protecting all current and future data bearing that tag.
* **Scalability:** Eliminates the human bottleneck during data ingestion. Security scales automatically with the pipeline.

---

## 6. Auditing & Lineage
When an incident occurs or an auditor requests proof of compliance, physical storage logs are insufficient.

* **Logical Auditing (The "Who"):** S3 logs only show that the compute engine read a file. You must rely on Catalog/Compute logs to capture the human User ID, the exact SQL executed, and the specific masking policies that were applied during the query.
* **Data Lineage (The "How"):** Tracks the flow of data across complex transformations (DAGs).
* **Automated Tag Propagation:** If lineage is maintained, tags applied to raw upstream tables (e.g., `PII`) automatically flow down to all downstream derivative tables and dashboards, ensuring ABAC policies follow the data indefinitely.