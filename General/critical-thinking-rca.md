[SUMMARY]
# Session: Root-Cause Analysis & Post-Mortems
**Completion Status:** All Layers (1-6) Completed.

## Layer 1: The Myth of the Single Root Cause & Blameless Culture
* **Key Concept:** "Human error" is a symptom, never a cause. You cannot scale "be more careful."
* **Takeaway:** Blameless culture is an engineering requirement, not an HR initiative. You need unvarnished truth to fix systems. Blaming humans hides the real architectural flaws.

## Layer 2: Constructing the Timeline
* **Key Concept:** Human memory during an incident is unreliable. Timelines must be built exclusively on system telemetry (metrics, logs, audit trails).
* **Takeaway:** Establish exact boundaries: Introduction, Start of Impact, Detection, Mitigation, and Resolution. This objectively measures MTTD and MTTR.

## Layer 3: Cause-and-Effect Chains (The 5 Whys)
* **Key Concept:** A forcing function to move from a localized trigger (the symptom) to the underlying systemic flaw.
* **Takeaway:** If your 5 Whys end with a person making a mistake, you stopped too early. It must end with missing architecture, tooling, or process guardrails.

## Layer 4: The Swiss Cheese Model of Failure
* **Key Concept:** Complex systems do not fail from a single event. Catastrophic failure requires multiple defensive layers (slices of cheese) to fail simultaneously.
* **Takeaway:** Distinguish between the **Trigger** (active failure that started it) and the **Holes** (latent bugs, missing tests, broken monitors) that allowed the failure to reach the customer.

## Layer 5: High-Leverage Action Items
* **Key Concept:** Action items exist on a leverage spectrum based on how much they rely on humans vs. machines.
* **Takeaway:** * *Low Leverage:* Process, training, checklists (manages symptoms, relies on humans).
    * *Medium Leverage:* Monitors, alerts (reduces MTTR).
    * *High Leverage:* Architectural constraints, automation (eliminates the class of error entirely, relies on deterministic systems).

## Layer 6: The Post-Mortem Lifecycle
* **Key Concept:** A perfect analysis is useless without execution. The document records history; the review meeting allocates resources.
* **Takeaway:** Use the review meeting to force the business to trade feature velocity for system stability, and cross-pollinate architectural fixes to other teams before they experience the same outage.

## Recommended Next Steps
To build on this foundation, consider exploring these related topics in future sessions:
1. **System Observability & Telemetry:** How to instrument systems so you don't fly blind during Layer 2 timeline construction.
2. **Circuit Breakers & Bulkheads:** Specific high-leverage architectural patterns for fault isolation.
3. **Progressive Delivery Strategy:** How to implement canary deployments and automated rollbacks to limit blast radius.