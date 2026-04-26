# Architecture Notes: Control Plane vs. Data Plane

## 1. The Core Distinction (The Brain vs. The Muscle)
* **Data Plane (The Muscle):** Handles the actual user traffic. Optimized exclusively for raw speed, reliability, and volume.
* **Control Plane (The Brain):** Manages the rules. Calculates configurations, monitors health, and tells the data plane what to do. Never touches user traffic.
* **Why Separate?** Offloading complex decision-making from the critical path ensures that heavy calculations don't slow down or drop user requests.

## 2. The Data Plane (Fast Paths & Forwarding)
* **Mechanism:** Uses a "Match-Action Pipeline". Matches an identifier (e.g., IP, header) against a local table, then executes a predefined action (forward, rewrite, drop).
* **Constraint:** Must operate at **line-rate** (microseconds/nanoseconds). 
* **Rule:** The data plane cannot rely on external dependencies (databases, external APIs) to process a packet. It relies purely on its local forwarding table.

## 3. The Control Plane (State & Calculation)
* **Mechanism:** Operates in a continuous loop:
    1.  **Discovery:** Monitors global state and telemetry.
    2.  **Calculation:** Runs complex logic (routing algorithms, policy evaluation).
    3.  **Distribution:** Compiles logic into flat rules and pushes them to the data plane.
* **Tradeoff:** Pushing state introduces eventual consistency. The data plane is always operating on a slightly stale version of the truth, which is necessary for scale.

## 4. Failure Modes & Blast Radii
* **Data Plane Failure:** Highly localized blast radius. A dead router drops the specific packets it was holding, but the control plane quickly routes around it.
* **Control Plane Failure:** Massive potential blast radius. Mitigated by **Static Stability**: if the control plane dies, the data plane survives by continuing to use its last known good configuration. Existing traffic keeps flowing.
* **State Drift:** If the control plane stays dead, the data plane cannot react to new hardware failures and will route traffic into black holes.
* **The Poison Pill:** The worst-case scenario. The control plane calculates a bad rule (e.g., "drop all traffic") and distributes it globally, taking down the entire data plane instantly.

## 5. Modern Manifestations
* **Software-Defined Networking (SDN):** Cloud providers (like AWS VPCs) moved the control plane out of physical switches and into a central cluster, leaving virtual network interfaces as dumb data planes.
* **Service Mesh (e.g., Istio + Envoy):** Applied the pattern to microservices. 
    * *Envoy (Data Plane):* A sidecar proxy attached to every service, executing routing rules on HTTP requests.
    * *Istio (Control Plane):* Central cluster that calculates routing percentages, retries, and mTLS policies, distributing them to the Envoys.