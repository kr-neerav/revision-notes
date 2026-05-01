# Data Center Networking: From Physical Racks to Global Traffic

A complete architectural breakdown of modern, hyperscale data center networking.

## Layer 1: The Physical Foundation
* **The Unit:** The **Rack** is the fundamental unit of deployment and failure.
* **ToR (Top-of-Rack):** The first switch the servers connect to. If the ToR dies, the entire rack is isolated (Blast Radius).
* **Oversubscription:** Bandwidth from the rack to the core is usually shared (e.g., a 4:1 ratio). Massive simultaneous data shuffles can cause packet drops at the uplink if not managed.

## Layer 2: The Legacy Bottleneck (Three-Tier Architecture)
* **Topology:** Core -> Aggregation -> Access.
* **The Threat:** "Broadcast Storms" (infinite packet loops) that instantly crash the network.
* **The "Fix" (STP):** Spanning Tree Protocol (STP) prevents loops by physically disabling redundant links.
* **The Failure:** STP kills 50-75% of available bandwidth. It was designed for "North-South" (Internet to Server) traffic, but completely chokes on modern "East-West" (Server to Server) traffic.

## Layer 3: The Modern Fix (Spine-Leaf / Clos Architecture)
* **The Geometry:** Two tiers. Every Leaf (ToR) connects to every Spine. No Leaf-to-Leaf links. No Spine-to-Spine links.
* **Predictability:** Any two servers in the data center are exactly 2 switch-hops apart, guaranteeing flat, predictable latency.
* **ECMP (Equal-Cost Multi-Path):** Replaces STP. Because all paths are mathematically equal, ECMP safely load-balances traffic simultaneously across all Spine switches without loops.

## Layer 4: The Underlay (BGP at Scale)
* **Routing to the ToR:** Layer 3 (IP) routing is pushed all the way down to the rack level, completely eliminating Layer 2 STP domains.
* **Why BGP?** * Legacy *OSPF* is "link-state" (every switch maps the whole network). A single flapping cable spikes CPU across the entire data center.
  * *BGP* is "distance-vector" (gossip-based). It isolates failures, handles massive scale easily, and prevents cascading CPU crashes.

## Layer 5: The Overlay (VXLAN & EVPN)
* **The Goal:** Allow virtual machines (VMs) and containers to keep their IP and MAC addresses even if they physically migrate across the data center.
* **VXLAN (The Shipping Container):** Wraps the original Layer 2 server packet inside a new Layer 3 IP header to cross the physical Underlay.
* **EVPN (The Shipping Manifest):** A BGP-based control plane that tracks exactly which physical physical switch (Leaf) currently hosts which specific VM/container.

## Layer 6: Software-Defined Networking (SDN)
* **The Separation:** Physically rips the **Control Plane** (the brain/routing logic) out of the switch and puts it in a centralized software cluster. The switch becomes just a fast **Data Plane** (the brawn).
* **API-Driven:** Networks are configured programmatically via REST APIs rather than typing commands box-by-box.
* **Data Plane Resilience:** Forwarding rules are burned into the switch's hardware silicon. If the central SDN controller crashes, existing network traffic continues without interruption ("headless mode").

## Layer 7: Edge & Traffic Engineering
* **Anycast:** Using BGP to advertise the same IP address from multiple global data centers. The internet routes the user to the closest functional location.
* **Software Load Balancing:** Replacing fragile, expensive hardware appliances with massive clusters of cheap commodity servers.
* **Consistent Hashing (Maglev):** Solves the ECMP TCP problem. Ensures that if a single load-balancer server crashes, the remaining servers use the same math formula to route existing user sessions to the correct backend server, keeping connections alive.