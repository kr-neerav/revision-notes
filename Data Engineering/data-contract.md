# Data Contracts: Revision Notes

## 1. The Core Problem: The Upstream-Downstream Divide
* **The Root Cause of Breakages:** Software engineering teams treat databases as mutable, internal application state. Data engineering teams treat those exact same databases as immutable, public APIs for the business.
* **Implicit vs. Explicit APIs:** Currently, the interface between software and data systems is implicit. When upstream teams make localized, non-breaking changes to their apps (e.g., changing an ENUM), it causes silent, downstream analytical outages.
* **Misaligned Incentives:** Software teams optimize for feature velocity; data teams optimize for analytical consistency. Without guardrails, communication is the only fallback, and communication does not scale.

## 2. Core Definition of a Data Contract
* **What it is:** An enforceable, version-controlled API definition for data. It is a physical file (usually YAML/JSON) that acts as a firewall against bad data.
* **Where it lives:** In the **upstream software engineering team's repository**, not the data team's repository. This integrates it directly into the deployment lifecycle.
* **Proactive vs. Reactive:** A `dbt` test is a reactive autopsy—if it fails, the bad data is already in the warehouse. A data contract is proactive—it prevents bad data from ever being merged into production.

## 3. Anatomy of a Contract
A mature data contract defines three distinct layers:
1. **Schema (Physical Layer):** The exact structure and data types (e.g., `user_id` must be a UUID, `email` must be a string).
2. **Semantics (Logical/Business Layer):** The meaning and constraints of the data (e.g., `status` can only contain `['active', 'suspended', 'deleted']`).
3. **SLAs & Metadata (Operational Layer):** The terms of service (e.g., data ownership, 15-minute freshness guarantees, and 30-day deprecation notice mandates).

## 4. Enforcement: CI/CD vs. Schema Registries
Data contracts and schema registries (like Confluent/Kafka) operate at different phases of the lifecycle.
* **Data Contracts (Deploy-Time):** Run in CI/CD. Act as a hard gate that fails a build if a PR introduces a backward-incompatible schema or semantic change.
* **Schema Registries (Run-Time):** Validate physical bytes (Avro/Protobuf) on the message broker to ensure consumers don't crash. They do not validate business logic.
* **The Workflow:** The Data Contract in CI/CD acts as the source of truth and automatically generates/pushes the physical schema to the Schema Registry upon a successful merge.

## 5. Versioning & Migration (Schema Evolution)
Contracts cannot freeze application development; they must support safe evolution.
* **Semantic Versioning:**
  * **Minor (Non-Breaking):** Adding a column or loosening a constraint (e.g., v1.1 to v1.2). Auto-approved in CI/CD.
  * **Major (Breaking):** Dropping a column, renaming, or tightening a constraint (e.g., v1.0 to v2.0). Blocked by CI/CD unless migrated safely.
* **The Expand/Contract Pattern:** The required sequence for making breaking changes without downtime.
  1. **Expand:** Write to both the old and new columns/formats simultaneously (Minor version update).
  2. **Deprecate:** Trigger an automated SLA window (e.g., 30 days) for downstream consumers to migrate their queries.
  3. **Contract:** Once downstream consumers have migrated, delete the old column and drop it from the contract (Major version update).

## 6. Organizational Dynamics (Shift-Left Ownership)
Data contracts are fundamentally a cultural transformation masquerading as an engineering project.
* **Shift-Left Ownership:** The producer of the data (software engineering) becomes accountable for the analytical quality of that data.
* **Division of Labor:**
  * **Consumers (Data Teams):** Define the requirements, SLAs, and semantics.
  * **Producers (Software Teams):** Own the implementation, the repository, and the deployment pipeline.
* **Executive Mandate:** To overcome organizational friction, leadership (CTO/VP) must mandate that a feature is not "Done" until its data contract is passing in CI/CD.