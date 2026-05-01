# Networking Fundamentals - Revision Notes

## Layer 1: The Local Network (L2)
* **The Problem:** Computers need a way to identify and talk to each other on the same physical network without shouting to everyone at once.
* **MAC Address:** A permanent, 48-bit physical identifier stamped onto network hardware at the factory. It provides the hardware's identity but has no concept of location.
* **The Switch:** A local network device that builds a "MAC Table" in memory by silently observing the source MAC addresses of incoming frames. It routes traffic point-to-point rather than broadcasting.
* **Broadcast Storms:** A failure mode where a faulty device floods the network with broadcast messages, forcing the switch to copy them to all ports, which can overwhelm and crash healthy servers.

## Layer 2: The Global Network (L3)
* **The Problem:** MAC addresses lack geographic topology. To route traffic across the world, we need an addressing system that represents location.
* **IP Address:** A logical address (like a postal code) that defines topology. It changes based on where the hardware is physically located.
* **The Router:** A Layer 3 gateway device. It uses software-defined "Routing Tables" to inspect IP addresses and forward packets hop-by-hop toward external networks.
* **Subnetting:** Nodes use their IP and subnet mask configuration to determine if a destination is local (send directly via Switch) or external (send to the Router's MAC address).

## Layer 3: Transport & Reliability (L4)
* **The Problem:** IP is a "best-effort" protocol that drops packets without warning. We need guaranteed delivery and a way to direct traffic to specific applications on the same server.
* **Ports:** A 16-bit number (0-65535) acting like an "apartment number" to identify the specific application (e.g., Port 443 for HTTPS, Port 5432 for PostgreSQL) receiving the packet.
* **TCP (Transmission Control Protocol):** A stateful connection that guarantees delivery and strict ordering. If a packet drops, TCP pauses to fetch it. Best for documents, queries, and financial data.
* **UDP (User Datagram Protocol):** A stateless, fire-and-forget protocol prioritizing raw speed over reliability. Best for real-time streams (gaming, VoIP) where stale data is useless.
* **Head-of-Line Blocking:** A TCP failure mode where a single dropped packet causes the receiving buffer to stall, potentially leading to application memory exhaustion.

## Layer 4: Naming & Application (L7)
* **The Problem:** Humans cannot easily memorize IP addresses, and applications need a structured, shared language to exchange data.
* **DNS (Domain Name System):** The distributed phonebook that translates human-readable domains (google.com) into IP addresses using fast UDP queries.
* **HTTP (Hypertext Transfer Protocol):** The structural language (Methods, Paths, Status Codes) used by clients and servers to exchange data once a TCP connection is established.
* **Failure Isolation:** A "Cannot resolve host" error means DNS failed, and the request never reached the network. An HTTP 500 error means the network routed perfectly, but the destination application crashed.

## Layer 5: Scaling the Traffic
* **The Problem:** A single backend server has finite compute and memory, limiting how many concurrent TCP connections it can hold.
* **The Load Balancer:** A middleman appliance that receives incoming TCP connections and distributes them across a massive pool of backend servers.
* **NAT (Network Address Translation):** The mechanism where the Load Balancer intercepts packets, swaps the destination public IP for a backend private IP, and vice versa when replying. The client never knows the backend exists.
* **SNAT Port Exhaustion:** A scaling failure mode where a Load Balancer runs out of its 65,535 local ports needed to proxy connections to the backend, causing it to drop requests even if CPU/Memory is healthy.

## Layer 6: Network Security
* **The Problem:** Implicit trust on an open network allows attackers to easily compromise backend infrastructure if a single perimeter defense fails.
* **Firewalls & VPCs:** Firewalls act as strict bouncers (Layer 4/7 rules). VPCs physically/logically isolate infrastructure into Public (internet-facing) and Private (internal-only) subnets.
* **The Perimeter Flaw:** Assuming a database is safe just because a firewall rule limits access to an internal web server IP. If an attacker breaches the web server via the internet, they automatically gain access to the database.
* **Zero Trust:** The modern security paradigm dictating that network location (being "inside") grants zero implicit trust. Every single request must be independently and cryptographically authenticated.