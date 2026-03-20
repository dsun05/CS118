## 1.0 Study Guide for the Final Exam (Overview)

### 1.1 Rules for Final Exam
The final examination is designed to test your independent mastery of the course material under strict conditions. 
*   **Closed book, closed notes**: No external course materials are allowed.
*   **No electronic devices**: The use of calculators, laptops, desktops, or Internet access is strictly prohibited.
*   **Allowed Resource**: You are permitted to bring **2 pages of "cheat sheets"**.
    *   **Page size**: Standard 8x11 inches.
    *   **Format**: Double-sided printing or writing is allowed.
    *   **Font size**: There are no restrictions on font size; you may write as small as you wish to maximize the information.

***

### 1.2 Materials to Be Covered
The exam will evaluate your understanding of the concepts covered in the lectures and the textbook (8th edition).

*   **High-priority topics (based on 8th edition textbook)**:
    *   **Chapter 4**: Data Plane (**4.1, 4.2, 4.3, 4.6**). Focus on the IP router architecture, the **IP protocol**, **DHCP** (Dynamic Host Configuration Protocol), **NAT** (Network Address Translation), the differences between **IPv4** and **IPv6**, and network transition strategies.
    *   **Chapter 5**: Control Plane (**5.1, 5.2, 5.3, 5.4, 5.6, 5.8**). Understand link-state vs. distance-vector routing, **BGP** (Border Gateway Protocol), and intra/inter-AS routing. *(Note: SDN & SNMP are excluded).*
    *   **Chapter 6**: Link Layer (6.1, 6.2, **6.3, 6.4**, 6.6, **6.7**). Focus heavily on MAC protocols, local area networks, switches, and ARP. *(Note: 6.5 MPLS is excluded).*
    *   **Chapter 7**: Wireless & Mobile Networks (**7.1, 7.2, 7.3.1~7.3.4, 7.5, 7.6.2**). Concentrate on CDMA, Wi-Fi MAC (CSMA/CA), mobility support, and mobile IP. *(Note: 2G/3G/4G materials are excluded).*
    *   **Chapter 8**: Security (*8.1, 8.2, 8.3, 8.4*). Understand basic security functions (encryption, message integrity) and how various authentication solutions defend against specific network attacks.
*   **Study Priority Order** (Allocate your review time accordingly):
    1.  **Lecture notes** (Primary focus; these represent the core examinable material).
    2.  Homework assignments.
    3.  Programming Project 2.
    4.  Textbook readings.

***

## 2.0 Chapter 4: Data Plane

### 2.1 Chapter 4
This section covers the foundational aspects of how network-layer packets are forwarded from a source to a destination.
*   **Internet service model**: The Internet provides a "best-effort" service model, meaning it guarantees neither packet delivery, order, nor timing.
*   **Virtual-circuit (VC) vs. datagram networks**: 
    *   *Virtual-circuit networks* require connection setup at the network layer, maintaining state at each router along the path.
    *   *Datagram networks* (like the Internet) route packets independently based on destination addresses without prior connection setup.
*   **Router next-hop forwarding decisions**: Routers use a forwarding table (FIB) to match the destination IP of an incoming packet to the longest prefix match to determine the appropriate output link.
*   **IP packet header field rationale**: Every field serves a specific purpose. For example, the `Time-to-Live (TTL)` prevents routing loops, `Header Checksum` detects bit errors in the header, and `Source/Destination IPs` identify the communicating hosts.
*   **Subnets, CIDR, and network masks**: 
    *   A **subnet** is a logical grouping of connected network devices. 
    *   **CIDR** (Classless Inter-Domain Routing) allows flexible allocation of IP addresses using a `/x` notation (e.g., `/24`).
    *   **Network masks** are used via bitwise AND operations with an IP address to extract the network portion.
*   **NAT Operations**:
    *   **Issues addressed by NAT**: It solves the exhaustion of the IPv4 address space by allowing an entire local network to share a single public IP address. It also provides basic security by hiding internal network topologies.
    *   **Step-by-step NAT procedures**: When a local host sends a packet, the NAT router replaces the local source IP and port with its own public IP and a unique port. It records this mapping in the NAT translation table. When the server replies to the public IP/port, the router uses the table to translate the destination back to the local host's IP/port.
    *   **NAT limitations and traversal**: NAT breaks the end-to-end argument of the Internet because routers manipulate layer 3 and 4 headers. Traversal solutions include UPnP, hole punching, or using relay servers (like TURN).
*   **DHCP Operations**:
    *   **Additional info**: Beyond assigning an IP address, DHCP provides the subnet mask, the IP of the first-hop router (default gateway), and the IP of the local DNS server.
    *   **4 DHCP messages**: Discover, Offer, Request, and ACK (commonly remembered as DORA).
    *   **Broadcast IPs**: A client uses `255.255.255.255` to reach all devices on the local subnet because it doesn't yet know the DHCP server's IP.
    *   **Reclaiming/Renewing**: IP leases expire. Clients can send a 2-message exchange (Request/ACK) directly to the server to renew their lease. If a client disconnects, the lease timer expires, and the server reclaims the IP.

***

### 2.2 Clarification on DHCP
*   **2-message vs. 4-message usage**: A full 4-message setup (Discover, Offer, Request, ACK) is used when a client is completely new to the network. The 2-message process (Request, ACK) is used simply to renew an existing lease before it expires.
*   **Use of ARP with DHCP**:
    *   DHCP works in tandem with **ARP** (Address Resolution Protocol) to prevent IP conflicts.
    *   Before formally claiming the IP address assigned by the DHCP server, the client broadcasts an ARP request for that specific IP. If another host replies, it means the IP is already in use, and the client will decline the DHCP offer.
*   * Visual Aid: Sequence diagram of DHCP discover, offer, request, and ACK messages between arriving client and DHCP server, showing source/destination IPs and ports. *

```text
+----------------+                                         +---------------+
| Arriving Client|                                         |  DHCP Server  |
| (IP: 0.0.0.0)  |                                         |(IP: 223.1.2.5)|
+----------------+                                         +---------------+
        |                                                          |
        | ----------------- DHCP discover -----------------------> |
        | src: 0.0.0.0, port 68                                    |
        | dest: 255.255.255.255, port 67                           |
        | yiaddr: 0.0.0.0, transaction ID: 654                     |
        | Broadcast: "Is there a DHCP server out there?"           |
        |                                                          |
        | <------------------ DHCP offer ------------------------- |
        | src: 223.1.2.5, port 67                                  |
        | dest: 255.255.255.255, port 68                           |
        | yiaddr: 223.1.2.4, transaction ID: 654                   |
        | Broadcast: "I'm a server! Here's an IP (223.1.2.4)"      |
        |                                                          |
        | ----------------- DHCP request ------------------------> |
        | src: 0.0.0.0, port 68                                    |
        | dest: 255.255.255.255, port 67                           |
        | yiaddr: 223.1.2.4, transaction ID: 655                   |
        | Broadcast: "OK. I would like to use this IP address!"    |
        |                                                          |
        | <-------------------- DHCP ACK ------------------------- |
        | src: 223.1.2.5, port 67                                  |
        | dest: 255.255.255.255, port 68                           |
        | yiaddr: 223.1.2.4, transaction ID: 655                   |
        | Broadcast: "OK. You've got that IP address!"             |
        |                                                          |
```

***

### 2.3 DHCP working with link layer
*   **Message passing through layered protocol stack**: The process of sending a DHCP message requires encapsulation down the entire protocol stack at the client.
    *   The application payload (`DHCP REQUEST`) is passed to the transport layer and **encapsulated** in a `UDP` segment.
    *   The `UDP` segment is passed to the network layer and encapsulated in an `IP` datagram.
    *   The `IP` datagram is passed to the link layer and encapsulated in an `Ethernet` frame.
*   The `Ethernet` frame is broadcast on the LAN using the destination MAC address `FF:FF:FF:FF:FF:FF` because the server's MAC is unknown. The router running the DHCP server receives this frame.
*   **Reverse process at DHCP server**: The server receives the frame and processes it bottom-up (Physical -> Ethernet -> IP -> UDP -> DHCP), decapsulating the data at each step to read the client's request.
*   The DHCP server formulates the `DHCP ACK`, packing it with the assigned IP, the router's IP, and the DNS server's IP.
*   **Demuxing**: Upon receiving the reply, the client decapsulates the message up the stack (**demuxed** frames), extracting the network configuration data to fully configure its network interface.
*   * Visual Aid: Diagrams showing a client, router with built-in DHCP server, and the layered encapsulation/decapsulation process across Physical, Ethernet, IP, UDP, and DHCP layers. *

```text
[ CLIENT ]                                         [ ROUTER w/ DHCP SERVER ]
Layer Stack                                        Layer Stack
-----------                                        -----------
|  DHCP   | ----- Encapsulation (Top-Down)         |  DHCP   |
|  UDP    |         |                        ^     |  UDP    |
|  IP     |         v                        |     |  IP     |
|  Eth    | -------------------------------------> |  Eth    | --- Demuxing (Bottom-Up)
|  Phy    |          Broadcast to FF:FF:...:FF     |  Phy    |

1. Client creates DHCP Request.
2. Encapsulates into UDP -> IP -> Ethernet.
3. Transmits over Physical medium (Phy).
4. Router receives via Phy.
5. Demuxes Ethernet -> IP -> UDP -> DHCP.
6. Server processes request and responds with ACK using same layer traversal.
```

***

### 2.4 Clarification on NAT
*   **Operation over network and transport layers**: NAT is unique because it modifies both the Network layer (IP address) and the Transport layer (Port number). 
*   This violation of the strict layered architecture is necessary to map multiple private IP addresses to a single public IP address using unique port numbers to distinguish between local hosts' connections.
*   * Visual Aid: Network diagram and NAT translation table showing a datagram's source address changing from a LAN IP/port to a WAN IP/port as it traverses the NAT router. *

```text
       [ LOCAL HOST ]
       IP: 10.0.0.1
             |
             | (1) Packet sent:
             | S: 10.0.0.1, Port 3345
             | D: 128.119.40.186, Port 80
             v
+-----------------------------+
|        NAT ROUTER           |
|  LAN IP: 10.0.0.4           |
|  WAN IP: 138.76.29.7        |
|                             |
| NAT TRANSLATION TABLE       |
| WAN Side            LAN Side|
| -----------------   --------|
| 138.76.29.7:5001 <-> 10.0.0.1:3345 |
+-----------------------------+
             |
             | (2) NAT router changes Source Address/Port:
             | S: 138.76.29.7, Port 5001
             | D: 128.119.40.186, Port 80
             v
       [ INTERNET ]
             |
             | (3) Reply arrives from Server:
             | S: 128.119.40.186, Port 80
             | D: 138.76.29.7, Port 5001
             v
       [ NAT ROUTER ] ---> (4) Router looks up table, forwards to Host:
                           S: 128.119.40.186, Port 80
                           D: 10.0.0.1, Port 3345
```

***

### 2.5 Chapter 4 (Continued)
*   **IPv4 vs. IPv6 fields**: 
    *   *Differences*: IPv6 uses 128-bit addresses (vs. 32-bit in IPv4). IPv6 removes fragmentation fields, the header checksum, and options fields from the main header to speed up processing.
    *   *Overlaps*: Both contain version numbers, source/destination addresses, hop limits (TTL in IPv4), and payload length indicators.
*   **Tunneling technique (IP-IP encapsulation)**: A transitional mechanism used because we cannot upgrade all routers globally at once. IPv6 packets are encapsulated entirely within the payload of an IPv4 datagram to traverse legacy IPv4 network segments.
*   **Incremental deployment**: Tunneling allows islands of new technology (IPv6) to communicate over the existing global Internet (IPv4) without requiring a simultaneous, universal upgrade.

***

## 3.0 Chapter 5: Control Plane

### 3.1 Chapter 5
*   **Routing algorithms**:
    *   **Link-state (LS) routing vs. distance-vector (DV) routing**: LS algorithms (like Dijkstra's) require global knowledge of the network topology, with every router broadcasting link costs to all others. DV algorithms (like Bellman-Ford) are decentralized; routers only know the costs to their direct neighbors and iteratively exchange distance estimates.
    *   **Information propagated/collected**: LS requires propagating $O(N \cdot E)$ messages (where N is nodes, E is edges) to build a complete map. DV only exchanges routing tables locally, significantly reducing initial message overhead but taking longer to converge.
    *   **Distance vector routing problems**: DV algorithms suffer from the "count-to-infinity" problem where a broken link causes routing loops and rapidly escalating distance metrics before the network converges.
*   **Intra-domain routing (OSPF vs. RIP)**:
    *   *OSPF (Open Shortest Path First)* uses Link-State routing. It is independent, highly scalable, and capable of calculating multiple same-cost paths for load balancing.
    *   *RIP (Routing Information Protocol)* uses Distance-Vector routing. It relies heavily on neighbors for computation and is historically susceptible to the count-to-infinity problem.
*   **Inter-domain routing (BGP basics)**:
    *   Hierarchical OSPF is for scaling within an AS, whereas BGP (Border Gateway Protocol) connects entirely different Autonomous Systems (ASes) together.
    *   **Route advertisement**: BGP advertises paths to subnets. A router may not accept all paths (due to internal policies) and will only propagate accepted paths downstream.
    *   **BGP path attributes**: The most important attributes are `AS-PATH` (the sequence of ASes a route traversed) and `NEXT-HOP` (the IP of the router that begins the AS-PATH).
    *   **Hot potato routing**: A strategy where a router hands off a packet to the closest local egress point (gateway) to get it out of its own AS as quickly as possible, minimizing internal cost.
*   **iBGP vs. eBGP operations**: 
    *   **eBGP** exchanges routes between distinct ASes. 
    *   **iBGP** distributes those learned external routes to all routers *within* the same AS.
*   **Routing loops in BGP**: Prevented primarily by the `AS-PATH` attribute. If a BGP router receives a route advertisement containing its own AS number in the `AS-PATH`, it rejects the route to prevent a loop.
*   **BGP with intra-AS routing**: BGP determines the egress gateway, and OSPF computes the physical shortest internal path to reach that gateway, populating the final forwarding table.

***

### 3.2 Clarification on count-to-infinity issue in distance vector routing
*   **Severity of count-to-infinity**: While it might seem like a minor issue requiring a few extra iterations, it is a severe flaw. 
*   **Practical failure scenarios**: Broken links (resulting in infinite link cost) are common in real-world networks. When a link drops, nodes continuously increment their cost estimates back and forth, attempting to find an alternative path through each other.
*   **Impact on reliability**: This creates a long convergence time, rendering the network effectively unresponsive and routing traffic in loops, drastically lowering fault tolerance.

***

### 3.3 Clarification on BGP related Notions
*   **peers**: Two BGP routers that establish a direct connection. They are established by manual configuration and communicate via a TCP session on port 179.
*   **iBGP (Internal BGP)**: BGP running between routers within the *same* Autonomous System.
*   **eBGP (External BGP)**: BGP running between routers in *different* Autonomous Systems.
*   **Border routers**: Gateways placed at the edge of an AS to communicate with other ASes via eBGP.
*   **Difference in route propagation**: 
    *   Routes learned from eBGP are advertised to all peers (both iBGP and eBGP).
    *   Routes learned from an iBGP peer are *not* forwarded to other iBGP peers (to prevent loops within the AS). Therefore, a full mesh topology is required among all iBGP peers inside an AS.
*   **Network Layer Reachability Information (NLRI)**: The payload of a BGP update message, indicating the specific IP prefix (e.g., `192.0.2.0/27`) that the advertising router can reach.

***

### 3.4 Clarifications on Intra- and Inter-AS routing
*   **OSPF/RIP**: Strictly concerned with finding the shortest physical path between internal routers within the boundaries of a single AS.
*   **BGP**: Used to learn how to reach subnets *outside* the AS by exchanging path vector advertisements.
*   **Next-hop router**: A term mostly used in OSPF/RIP, indicating the immediate next physical router on the interface path toward a destination.
*   **Next-hop path attribute**: A specific BGP term indicating the IP address of the border router that is the entry point into the next AS along the BGP route.

***

### 3.5 Next Hop Attribute in BGP advertised routes
*   When reaching external networks, the eBGP local network gateway is identified as the **Next Hop**.
*   This Next Hop attribute is updated when routes are passed between eBGP peers (from one AS to another).
*   **"Unreachable AS" issues**: When border routers pass the BGP route into their own network via iBGP, the `NEXT-HOP` attribute remains unchanged by default. Internal routers must therefore be able to route to that external interface IP. If OSPF hasn't been configured to reach the external subnet interface linking the two ASes, internal routers cannot forward traffic, leading to unreachable destinations.
*   * Visual Aid: Multiple network diagrams showing AS 100, 200, and 300 with routing tables indicating Network, Next-Hop, and Path attributes for BGP update messages. *

```text
[ AS 100 ]                      [ AS 200 ]                      [ AS 300 ]
160.10.0.0/16                   150.10.0.0/16                   140.10.0.0/16
       \                               /                               /
      (A)--- .1  192.20.2.0/30  .2 ---(B)--- .1 192.10.1.0/30 .2 ---(C)
                                       |
                                       |  BGP Table at AS 200 (Router B):
                                       |  Network        Next-Hop     Path
                                       |  ---------      ----------   ----
                                       |  150.10.0.0/16  192.10.1.1   200
                                       |  160.10.0.0/16  192.10.1.1   200 100

* BGP Update Messages are sent containing Network Prefixes, Next-Hop IP, and AS Path.
* Next Hop IP indicates the specific interface on the peer boundary router.
```

***

### 3.6 Internet Routing Example
Synthesizing how these protocols interact to route a packet globally:
*   **eBGP path vector advertisement**: Border routers broadcast AS-level paths to external networks.
*   **iBGP path vector propagation**: Border routers internally flood these external paths so all routers in the AS know which gateway to use.
*   **OSPF shortest path computation**: Internal routers calculate the physical hops required to reach the chosen internal gateway.
*   **Hot potato routing application**: If multiple gateways advertise the same external destination, the router picks the gateway with the lowest OSPF internal cost.

***

### 3.7 Example on Internet routing: eBGP
*   A gateway router may learn multiple possible paths to a destination subnet via different eBGP peers.
*   The selection of which path to use is based on local AS policy (e.g., favoring one ISP over another) and AS-PATH length. Once selected, this route is pushed internally.
*   * Visual Aid: Network topology spanning AS 1, AS 2, and AS 3 demonstrating eBGP path vectors being sent from gateway routers. *

```text
        [ AS 1 ]                 [ AS 2 ]                  [ AS 3 ]
       /        \               /        \                /        \
    (1b)        (1c) <======= (2a)       (2b)          (3a)        (3b)
   /                \           \        /               \         /  |
(1a)-----(1d)------- +          (2d)--(2c) <============ + ------(3d)(3c)
                     ^                    ^
             Receives AS2, AS3, X     Receives AS3, X
             Receives AS3, X
```

***

### 3.8 Example on Internet routing: iBGP & OSPF
*   Internal routers (like `1a`, `1d`) learn via **iBGP** from the border router (`1c`) that the best path to destination X is through `1c`.
*   To actually move a packet toward `1c`, the router relies on its **OSPF** intra-domain routing table, which provides the local link interface mapped to the shortest physical path to `1c`.
*   * Visual Aid: Network topology detailing local link interfaces, dest/interface routing tables, and intra-domain routing paths to reach external destinations. *

```text
[ AS 1 Internal Routing ]
        (1b)
       /    \
    1 /      \
    (1a)      (1c) ----> [ To AS 2 ]
    2 \      /
       \    / 1
        (1d)
          |
  Forwarding Table at 1d:
  +------+-----------+
  | Dest | Interface |
  +------+-----------+
  |  1c  |     1     |
  |  X   |     1     |  <-- To reach external X, forward out link 1 toward 1c
  +------+-----------+
```

***

### 3.9 Example on Internet routing: Hot potato routing
*   A router (e.g., `2d` in AS 2) may learn via iBGP that it can reach destination `X` through either border router `2a` or `2c`.
*   If both paths have the same BGP AS-PATH length, **hot potato routing** dictates choosing the gateway with the lowest intra-domain OSPF cost.
*   * Visual Aid: Diagram showing OSPF link weights and a router choosing a gateway based on the lowest cost rather than BGP path length. *

```text
         [ AS 2 ]
         
     (2b) --112-- (2c) ---> [ Gateway to AS 3 ]
      |            |
      |            | 263
      |            |
     (2a) --201-- (2d)
      |
      v
[ Gateway to AS 1 ]

* Router 2d wants to send to X.
* Cost to gateway 2a = 201.
* Cost to gateway 2c = 263.
* Hot Potato Routing Decision: 2d chooses to forward to 2a (lowest local cost), getting the packet out of the AS as cheaply as possible.
```

***

## 4.0 Chapter 6: Link Layer

### 4.1 Chapter 6
*   **Link-layer header rationale**: An IP header routes data end-to-end across multiple networks. A link-layer (frame) header is necessary to deliver the data hop-by-hop across a specific physical medium (e.g., Ethernet or Wi-Fi). They cannot be merged because IP addresses are logical and global, whereas MAC addresses are physical and local to the hardware.
*   **Error-detection algorithm accuracy**: No error detection algorithm (like CRC) provides 100% accuracy. They are designed to detect typical burst errors with extremely high probability, but mathematically, some bit-flip combinations can slip through undetected.
*   **MAC Protocols (Cons/Pros)**:
    *   *Channel partitioning MAC* (TDMA/FDMA/CDMA): Pros: Eliminates collisions, fair. Cons: Inefficient at low traffic loads due to idle reserved slots.
    *   *Random access MAC* (CSMA, ALOHA): Pros: Highly efficient at low loads (nodes transmit immediately). Cons: Collisions degrade performance at high traffic volumes.
    *   *Taking-turns (token-based) MAC*: Pros: Combines benefits of partitioning and random access, no collisions. Cons: Token overhead, point of failure if token is lost.
*   **CSMA and CSMA/CD operations**: Carrier Sense Multiple Access (listen before speaking). CSMA/CD includes Collision Detection (stop speaking if someone interrupts).
*   **Collisions in CSMA/CD**: Collisions can still occur due to propagation delay. Two nodes might sense an idle channel and transmit simultaneously because the signal from one hasn't reached the other yet.
*   **Binary exponential backoff**: Upon collision, nodes wait a random amount of time. If consecutive collisions happen, the collision window size doubles (exponentially), adapting to network congestion to prevent immediate re-collision.
*   **ARP operations and soft-state timers**: ARP maps IP addresses to MAC addresses. It relies on a "soft-state" model—entries in the ARP cache expire after a set timer (typically 20 minutes) if not refreshed, ensuring outdated hardware changes don't permanently break connectivity.
*   **Efficiency comparisons**: Slotted ALOHA is more efficient than pure ALOHA due to synchronized transmission boundaries. CSMA/CD is significantly more efficient than both because it aborts collided transmissions early, saving channel time that would otherwise be wasted transmitting corrupted frames.

***

### 4.2 Example of ARP Operations
*   **Resolving MAC addresses**: To send an IP datagram to an IP address on the same local subnet, the sender must wrap the datagram in an Ethernet frame requiring a destination MAC address.
*   **ARP cache lookup**: Host A checks its local ARP cache for Host B's IP address. If found, it uses the cached MAC.
*   **Broadcast ARP request**: If the cache misses, Host A constructs an ARP query ("Who has 192.168.0.55?") and broadcasts it to `FF:FF:FF:FF:FF:FF`.
*   **ARP response and caching**: Every node on the switch receives the query. Only Host B matches the IP and replies directly (unicast) to Host A with its MAC address. Both hosts cache this mapping for future use.

***

### 4.3 Chapter 6 (Continued)
*   **Packet delivery across subnets**: When sending to a different subnet, the sending host uses ARP to find the MAC address of its *default gateway router*, not the final destination. The frame header changes at every hop (source and destination MACs update), while the IP header remains largely unchanged (except for decrementing TTL).
*   **DHCP as a soft-state protocol**: DHCP leases IP addresses temporarily. If a client doesn't renew, the state expires, allowing dynamic reallocation of resources.
*   **ARP in point-to-point links**: ARP is generally unneeded in point-to-point links (like PPP) because there is only one possible destination on the wire, unlike broadcast mediums like Ethernet where nodes must be uniquely identified.
*   **Switches vs. Routers**: 
    *   *Switches (Layer 2)*: Connect devices within the *same* subnet. They isolate collision domains but forward broadcasts to all ports (same broadcast domain).
    *   *Routers (Layer 3)*: Interconnect *different* subnets. They isolate broadcast domains.
*   **Isolating collision domains**: A switch provides dedicated bandwidth per port, completely eliminating collisions on full-duplex links.
*   **Self-learning algorithm**: Switches build their switching tables dynamically by observing the source MAC address of incoming frames and recording which port they arrived on.
*   **Protocol synthesis**: An action like browsing a webpage requires DNS (UDP port 53) to resolve the hostname, DHCP (UDP ports 67/68) to obtain an initial IP, ARP to find the gateway MAC, and finally HTTP/TCP (Port 80/443) to retrieve the content.

***

### 4.4 Another Synthesis Example
*   **Protocol stack integration**: The process requires interaction spanning all layers: Application (HTTP/DNS), Transport (TCP/UDP), Network (IP, OSPF/BGP), and Link (Ethernet/802.11).
*   **Video watching scenario**: A student attaches a tablet to Campus Wi-Fi and browses YouTube. This seemingly simple action kicks off a cascade of network protocols detailed below.

***

### 4.5 A day in the life: scenario
*   * Visual Aid: High-level network topology showing a mobile browser connecting to Wi-Fi, passing through a Comcast network, reaching YouTube streaming servers, and resolving via DNS and KingCDN. *

```text
  [ Mobile Client ]                 [ Campus Wi-Fi Router ]
         |  (Browser)                        | (DHCP/DNS/Gateway)
         |~~~~~~~~~~ (Wireless) ~~~~~~~~~~~~ |
                                             |
                                             | 
                                      [ ISP Network ] (e.g., Comcast)
                                             |
                 +---------------------------+---------------------------+
                 |                           |                           |
                 v                           v                           v
         [ DNS Servers ]             [ YouTube Subnet ]          [ CDN Subnet ]
         (Root/Authoritative)        (Web/Streaming Servers)     (KingCDN caching)
```

***

### 4.6 A day in the life: connecting to the Internet
*   To start, the mobile client needs network parameters: IP address, first-hop router IP, and DNS server IP. It uses DHCP.
*   The DHCP request is **encapsulated** (`UDP` -> `IP` -> `802.11 Wi-Fi MAC`).
*   The Access Point (AP) receives the wireless frame and bridges it to the wired LAN, broadcasting it to the router.
*   The router receives the frame and processes it up the stack. It is **demuxed** at the server to reach the DHCP application.
*   The DHCP server formulates the ACK. As the ACK travels back through the switch, **switch learning** occurs, allowing the switch to record the client's new MAC address to port mapping.
*   * Visual Aid: Step-by-step encapsulation and demultiplexing visual of a DHCP request traveling from a laptop to a DHCP router. *

```text
[ LAPTOP ]                                [ ROUTER WITH DHCP ]
  DHCP  | Request                             ^  | DHCP
  UDP   v                                     |  | UDP
  IP    | Encapsulation               Demux   |  | IP
  WiFi  v                                     |  | Eth
  Phy   --------> [ ACCESS POINT ] --------> Phy |
                     (WiFi -> Eth)
```

***

### 4.7 A day in the life… ARP (before DNS, before HTTP)
*   The user types `www.youtube.com`. Before sending an HTTP request, the client must resolve the IP address using DNS.
*   To send the DNS query (which must go to the DNS server via the default gateway router), the client needs the router's MAC address.
*   The client broadcasts an ARP query (`FF:FF:FF:FF:FF:FF`).
*   The router responds with an ARP reply. The client caches the MAC and can now transmit the frame containing the DNS query.
*   * Visual Aid: Diagram detailing the ARP request/reply cycle between a client and the first-hop router. *

```text
[ CLIENT ]                                          [ ROUTER (Gateway) ]
    |                                                      |
    | ------- ARP Broadcast: "Who has IP of Gateway?" ---> |
    |                                                      |
    | <------ ARP Unicast: "I do. My MAC is AA:BB..." ---- |
    |                                                      |
    v                                                      v
(Client updates ARP cache)                        (Gateway processes request)
```

***

### 4.8 A day in the life… using DNS
*   With the router's MAC known, the DNS query is encapsulated in a UDP/IP datagram and sent to the first-hop router.
*   The router forwards the datagram into the ISP network toward the local DNS server.
*   The local DNS server receives the packet, demuxes it, and processes the lookup.
*   * Visual Aid: Path of the DNS UDP/IP datagram across the network to the local DNS server and back. *

```text
[ CLIENT ] --(Eth)--> [ GATEWAY ROUTER ] --(IP Routing)--> [ LOCAL DNS SERVER ]
    ^                                                             |
    |                                                             |
    +--------------------- DNS IP Reply --------------------------+
```

***

### 4.9 A day in the life… Iterative DNS
*   If the local DNS server does not have the record cached, it performs iterative DNS resolution, querying the Root, TLD, and Authoritative DNS servers.
*   This traffic is routed globally via OSPF, IS-IS, and BGP routing protocols.
*   Because YouTube utilizes Content Delivery Networks (CDNs), the authoritative server might redirect the resolution to `www.KingCDN.com`.
*   The iterative replies pass back to the local DNS server, which finally replies to the mobile client with the target CDN IP.
*   * Visual Aid: Diagram of iterative DNS resolution communicating with Comcast, KingCDN, and YouTube DNS servers. *

```text
[ LOCAL DNS ] ---> 1. Query to Root Server
              <--- 2. Referral to .com TLD
              ---> 3. Query to .com TLD
              <--- 4. Referral to YouTube Authoritative DNS
              ---> 5. Query to YouTube DNS
              <--- 6. CNAME alias to KingCDN.com
              ---> 7. Query to KingCDN DNS
              <--- 8. IP Address of nearby CDN cache server (e.g., 64.233.160.15)
```

***

### 4.10 A day in the life…TCP connection carrying HTTP
*   Now possessing the CDN server's IP, the client opens a TCP socket.
*   The client initiates the **TCP 3-way handshake** by sending a `TCP SYN` segment.
*   The segment is routed through the Internet to the CDN server, which responds with a `TCP SYNACK`.
*   The client replies with an `ACK`, successfully establishing the connection.
*   * Visual Aid: Diagram showing the TCP SYN/SYNACK handshake process establishing a connection with a CDN server. *

```text
[ CLIENT ]                                          [ CDN SERVER ]
    |                                                      |
    | ------------------- TCP SYN (Step 1) --------------> |
    |                                                      |
    | <---------------- TCP SYNACK (Step 2) -------------- |
    |                                                      |
    | ------------------- TCP ACK (Step 3) --------------> |
    |                                                      |
   (TCP Connection Established)
```

***

### 4.11 A day in the life… HTTP request/reply
*   With the TCP socket open, the client sends the `HTTP GET` request for the video page.
*   The request is packaged into IP datagrams, routed to the KingCDN server.
*   The server processes the request and streams the HTTP reply containing the video data back to the client, utilizing adaptive streaming protocols like DASH.
*   * Visual Aid: Final diagram showing the HTTP request payload traveling to the CDN and returning the media data to the client. *

```text
[ CLIENT ]                                          [ CDN SERVER ]
    |                                                      |
    | -------- HTTP GET Request (inside TCP socket) -----> |
    |                                                      |
    | <------- HTTP Reply (Web page / DASH Video data) --- |
    v                                                      v
  Play Video                                            Serve Media
```

***

## 5.0 Chapter 7: Wireless & Mobile Networks

### 5.1 Chapter 7
*   **CDMA (Code Division Multiple Access)**: A channel partitioning MAC protocol. Unlike time or frequency partitioning, CDMA allows multiple transmitters to send simultaneously over the same frequency. It works by assigning a unique mathematical code to each transmitter. Receivers use the designated code to extract the intended signal from the combined interference.
*   **CSMA/CA detailed operations**: Wireless networks (802.11) use CSMA/CA (Collision *Avoidance*). Before transmitting, a node listens. If the channel is idle for a specific time (DIFS), it transmits. If busy, it uses a random backoff timer.
*   **Collision avoidance vs. Collision detection**: 802.11 cannot use CSMA/CD because wireless radios cannot easily listen while transmitting (the transmission signal drowns out incoming signals). Furthermore, the hidden terminal problem makes collision detection across the whole network impossible.
*   **Link-layer acknowledgments**: Because collisions can't be perfectly avoided and wireless channels suffer from fading and high error rates, 802.11 implements explicit Link-Layer ACKs. If an ACK isn't received, the sender retransmits. TCP ACKs operate at a higher layer and take much longer; MAC ACKs provide rapid, localized error recovery.
*   **Hidden terminal mechanisms (RTS/CTS)**: Two nodes may be able to transmit to a central Access Point but cannot hear each other, causing collisions at the AP. 802.11 solves this using short Request-To-Send (RTS) and Clear-To-Send (CTS) control frames to reserve channel time and notify hidden terminals to stay quiet.
*   **Handling mobility**:
    *   *Same Subnet*: Mobility is handled seamlessly by Layer 2 switches updating their forwarding tables as a host moves between Access Points.
    *   *Different Subnets*: Requires Mobile IP.
*   **Routing to mobile hosts (Mobile IP)**:
    *   The host has a permanent IP in its **home network** managed by a **home agent**.
    *   When moving, it receives a temporary Care-Of-Address (COA) from a **foreign agent**.
    *   Traffic sent to the permanent IP is intercepted by the home agent and sent via **tunneling** to the foreign agent, which delivers it to the mobile host.
*   **Avoiding triangle routing**: Standard mobile IP creates inefficient triangle routing (Sender -> Home Agent -> Foreign Agent -> Mobile Host). This is optimized by allowing the sender to query the home agent for the host's COA and then route packets directly to the visited network.
*   **Location tracking and updating**: The mobile host registers its new location (COA) with its home agent immediately upon joining a foreign network.

***

## 6.0 Chapter 8: Security

### 6.1 Chapter 8
*   **Public key vs. symmetric key-based encryption**: 
    *   *Symmetric Key*: Both parties share a single, identical secret key used for both encryption and decryption (e.g., AES). It is computationally fast but requires secure initial key exchange.
    *   *Public Key*: Uses a mathematical key pair (Public Key and Private Key). What one key encrypts, only the other can decrypt (e.g., RSA). It is slower but solves the key distribution problem.
*   **Using public/private keys**:
    *   *Encryption*: Sender encrypts data using the receiver's **Public Key**. Only the receiver can decrypt it using their **Private Key**.
    *   *Authentication*: Sender encrypts a challenge nonce using their **Private Key**. The receiver uses the sender's **Public Key** to decrypt it, proving the sender's identity.
    *   *Digital signature*: Sender encrypts a document hash with their **Private Key** to prove authorship and non-repudiation.
    *   *Message integrity*: Sending a digital signature alongside the message guarantees the contents haven't been altered.
*   **Network attacks and security defense mechanisms**:
    *   *Data sniffing & interception*: Passive eavesdropping. Defended by **Encryption** (turning plaintext into ciphertext).
    *   *IP address spoofing*: Faking the source IP. Defended by **Endpoint Authentication** (verifying cryptographic identity, not just IP).
    *   *Replay attack*: Intercepting a valid packet and re-transmitting it later. Defended by using unique **Nonces** (numbers used once) or sequence numbers.
    *   *Man in the middle (MITM) attack*: An attacker secretly intercepts and relays communications. Defended by using **Digital Certificates** (issued by trusted CAs) to bind public keys to actual entities.
    *   *Email spam / Illegal access*: General threats defended by robust authentication and perimeter security policies.
*   *(Excluded topics: SSH, IPSec, and VPN functions/operations are not on the exam).*

***

## Key Terms to Define
*   **ARP** (Address Resolution Protocol): A link-layer protocol used to discover the physical MAC address associated with a known network-layer IP address within a local subnet.
*   **peers**: In BGP, two routers that have been manually configured to establish a direct TCP connection (port 179) to exchange routing information.
*   **iBGP** (Internal BGP): The protocol used to propagate external BGP route advertisements to all internal routers *within* the same Autonomous System.
*   **eBGP** (External BGP): The protocol used to exchange routing information *between* border routers of different Autonomous Systems.
*   **Border routers**: Gateway routers positioned at the edge of an Autonomous System tasked with handling eBGP communications with external networks.
*   **Network Layer Reachability Information (NLRI)**: The core payload of a BGP update message containing the IP prefix (subnet) that the advertising router is declaring it can reach.
*   **Next-hop router**: An OSPF/RIP term indicating the immediate next physical router interface on the shortest path to a destination.
*   **Next-hop path attribute**: A BGP attribute that indicates the IP address of the border router serving as the entry point into the next Autonomous System along a BGP path.
*   **hot potato routing**: An inter-domain routing strategy where a router passes traffic to the closest available egress gateway out of the AS, minimizing intra-domain routing costs regardless of the external path length.
*   **encapsulated**: The process of taking a data packet from a higher layer and wrapping it inside the header and payload of a lower-layer protocol (e.g., wrapping an IP datagram inside an Ethernet frame).
*   **demuxed** (demultiplexed): The process at the receiving end where a lower-layer protocol strips its header and uses field identifiers to correctly pass the payload up to the appropriate upper-layer protocol.
*   **switch learning**: The automated process where an Ethernet switch builds its MAC forwarding table by observing the source MAC addresses of incoming frames and associating them with the port on which they arrived.
