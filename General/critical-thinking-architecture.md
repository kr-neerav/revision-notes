## Structural Tradeoff Analysis: Revision Notes

**Core Concept:**
In distributed systems, there are no perfect architectures, only N-dimensional tradeoffs. Optimizing one metric (like availability or throughput) never eliminates complexity; it merely shifts that complexity to another part of the system or onto another team.

### Layer 1: The Dimensionality of Tradeoffs
* **The Matrix:** Tradeoffs are not binary (e.g., Consistency vs. Availability). They affect secondary metrics like downstream developer velocity, customer support load, and infrastructure cost.
* **The Hidden Tax:** The true cost of an architectural decision is the compensating mechanisms you must build to handle its failure modes (e.g., writing Distributed Sagas when moving to event-driven architectures).

### Layer 2: Hard Constraints vs. Soft Preferences
* **Hard Constraints (Immutable):** Physics (latency limits), regulatory compliance (data sovereignty), and hard budget caps. You cannot negotiate with the speed of light.
* **Soft Preferences (Negotiable):** SLAs, feature requirements, and uptime targets. 
* **The Rule:** When a Soft Preference conflicts with a Hard Constraint, the Soft Preference must be dropped or modified.

### Layer 3: Blast Radius and Failure Domains
* **Assume Failure:** Design for *when* things fail, not *if*.
* **Blast Radius:** The maximum potential impact of a single localized failure.
* **Failure Domains:** Deliberate isolation boundaries (Process -> Node -> AZ -> Region -> Cell) used to contain the blast radius. 
* **The Trap:** Beware the shared centralized resource (a global connection pool, a single queue). A single point of failure (SPOF) instantly expands your blast radius to the entire system, regardless of how much redundant compute you deploy.

### Layer 4: State, Consistency, and Anomalies
* **The Physics of State:** Data existing in two places at once inherently creates a consistency problem.
* **Specific Anomalies:** Eventual consistency is not a monolith. You must define what you tolerate:
    * *Stale Reads / Read-Your-Writes Violation:* User cannot see the data they just saved.
    * *Monotonic Read Violation:* Data appears to travel backward in time across refreshes.
* **Routing Mitigations:** You can often solve consistency anomalies without changing the database engine by using routing logic (e.g., sticky sessions to pin a user's reads to the node they just wrote to).

### Layer 5: The Operational Tax
* **The Human Cost:** Every architectural choice incurs an ongoing tax paid in cognitive load, onboarding time, MTTR, and on-call burnout.
* **The Trap of "Scale":** Optimizing for technical isolation (e.g., 100 microservices for a 15-person team) often bankrupts developer velocity.
* **The Alternative:** Coarse-grained modular monoliths with bounded contexts often provide the best balance of scaling runway and operational sanity for small-to-mid-sized teams.