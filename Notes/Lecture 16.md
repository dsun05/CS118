# Chapter 6: The Link Layer and LANs

## 1.0 Ethernet

### 1.1 Basics

**Ethernet** stands as the "dominant" wired Local Area Network (LAN) technology in the world today. Its ubiquitous nature is largely due to its first-mover advantage—being the first widely deployed high-speed LAN technology—combined with its architectural simplicity and extremely low cost compared to early alternatives like Token Ring or ATM. 

Over the decades, Ethernet has impressively kept up with the "speed race" of modern networking. While original implementations ran at 10 Mbps, modern standards support speeds ranging up to 800 Gbps, with projections of reaching 1.6 Tbps by 2026. Furthermore, modern Ethernet hardware is highly integrated; a single chip, such as the `Broadcom BCM5761`, can support multiple speeds, auto-negotiating the fastest common rate between devices.

```text
Metcalfe's Original Ethernet Sketch (Conceptual)
  
      TRANSCEIVER      STATION
         |               |
 TAP  +--V--+        +---+---+
 ====>|  X  |        | I   C |
      +--+--+        +-------+
         | INTERFACE     ^
         | CABLE         |
         |               |
   +-----+-----+         |
   |   |   |   |=========+
   +---+---+---+ 
     THE ETHER (Coaxial Cable Bus)       TERMINATOR =>
```

***

### 1.2 Physical Topology

The physical layout of Ethernet networks has fundamentally shifted over time:

*   **Bus Topology (Legacy):** Popular through the mid-1990s, early Ethernet relied on a single coaxial cable acting as a shared bus. In this topology, all nodes exist within the same **collision domain**, meaning if two nodes transmit simultaneously, their signals collide and interfere with each other.
*   **Switched Topology (Modern):** This architecture prevails today. It utilizes an active link-layer device known as a **switch** placed in the center of a star topology. Each host ("spoke") connects to the switch via a dedicated link running a separate Ethernet protocol instance. Because each link is isolated, nodes do not physically collide with one another.

```text
       Bus Topology                       Switched Topology
       (Coaxial Cable)                    (Twisted Pair / Star)
       
   PC   PC   PC   PC                        PC       PC
   |    |    |    |                          \       /
======================                        \     /
     Shared Media                              +---+
                                         PC ---|SW |--- PC
                                               +---+
                                              /     \
                                             /       \
                                            PC       PC
```

***

### 1.3 Ethernet Frame Structure

When a transmitting interface receives data from the network layer (typically an IP datagram), it encapsulates that packet within an **Ethernet frame** for physical transmission.

*   **Preamble:** This 8-byte field is crucial for physical-layer synchronization. It consists of 7 bytes of alternating `10101010` bits followed by a single byte of `10101011` (known as the Start Frame Delimiter). It allows the receiving adapter to lock onto the sender's clock rates before the actual frame data arrives.
*   **Addresses (MAC):** The frame includes 6-byte source and destination **MAC addresses**. When a Network Interface Card (NIC) receives a frame, it checks the destination MAC. If it matches the NIC's physical address, or if it is the broadcast address (`FF:FF:FF:FF:FF:FF` used for protocols like ARP), the data is extracted and passed up the protocol stack. Otherwise, the adapter simply discards the frame.
*   **Type:** This 2-byte field acts as a multiplexing mechanism. It indicates the higher-layer protocol encapsulated within the payload. While this is almost always IPv4 or IPv6 today, it supports other protocols like Novell IPX or AppleTalk.
*   **CRC (Cyclic Redundancy Check):** A 4-byte checksum located at the tail of the frame. The receiver recalculates the CRC based on the received bits. If an error is detected due to noise or interference, the frame is silently dropped.

```text
+-----------------------------------------------------------------------+
| Preamble | Dest. Address | Source Address | Type | Payload Data | CRC |
| (8 bytes)|   (6 bytes)   |   (6 bytes)    |(2B)  | (46 - 1500B) |(4B) |
+-----------------------------------------------------------------------+
```

***

### 1.4 Unreliable, Connectionless Service

Ethernet provides a streamlined, lightweight service model:
*   **Connectionless:** There is no handshaking between the sending and receiving NICs. A transmitter simply builds the frame and puts it onto the wire.
*   **Unreliable:** The receiving NIC does not send acknowledgments (ACKs) or negative acknowledgments (NAKs) back to the sender. If a frame fails the CRC check and is dropped, Ethernet itself makes no attempt to recover it. Any data recovery must be handled by higher-layer reliable data transfer (RDT) protocols, such as TCP.

Historically, over shared media, Ethernet managed access using **CSMA/CD** (Carrier Sense Multiple Access with Collision Detection) combined with **binary exponential backoff**. Nodes would "listen" to the wire, transmit if idle, and if a collision occurred, they would back off for a randomized period of time that doubled after each consecutive collision.

***

### 1.5 802.3 Ethernet Standards: Link & Physical Layers

The IEEE 802.3 working group defines *many* different Ethernet standards. While the MAC protocol and the frame format remain strictly uniform across all variations, the physical layer implementations change to accommodate different speeds and transmission media (e.g., copper twisted pair vs. fiber optics).

```text
 +---------------+
 |  Application  |
 +---------------+
 |   Transport   |
 +---------------+
 |    Network    |
 +---------------+             MAC Protocol and Frame Format
 |     Link      | . . . . . . . . . . . . . . . . . . . . . . . . .
 +---------------+        [100BASE-TX] [100BASE-T4] [100BASE-T2] (Copper)
 |   Physical    | . . . .[100BASE-FX] [100BASE-SX] [100BASE-BX] (Fiber)
 +---------------+
```

***

### 1.6 Ethernet Switch Basics

A switch is an active, **link-layer** device tasked with storing and forwarding Ethernet frames. Unlike a legacy hub, a switch examines the incoming frame's destination MAC address and *selectively* forwards the frame out of the specific interface required to reach the destination. 

Switches operate **transparently**, meaning end-hosts are completely unaware of their presence; hosts address frames to other hosts, not to the switch. Furthermore, switches are **plug-and-play** and **self-learning**—they require virtually no manual configuration by network administrators to begin passing traffic.

***

### 1.7 Multiple Simultaneous Transmissions

In modern switched networks, hosts maintain a dedicated, direct connection to the switch. The switch possesses internal buffers to queue packets if necessary. Because the protocol is running on dedicated links in **full duplex** mode (separate channels for transmit and receive), collisions are physically impossible. Therefore, CSMA/CD is no longer invoked.

This isolation means each link is its own discrete collision domain, permitting **switching** of multiple frames simultaneously. For example, host A can send to host A' at the exact same time host B sends to host B' without interference. However, if A and C both attempt to send to A' simultaneously, the switch will buffer one of the frames and forward them sequentially.

```text
                 A                      B
                  \                    /
                   \                  /
                    +----------------+
             C ---- |   1        2   | ---- C'
                    |        X       |
            B' ---- |   6        3   | ---- A'
                    |   5        4   |
                    +----------------+
                   /                  \
                  /                    \
                 A'                     B'

Simultaneous Paths: [A -> 1 -> Switch -> 4 -> A'] AND [B -> 2 -> Switch -> 5 -> B']
```

***

### 1.8 Switch Forwarding Table

To intelligently route frames, each switch maintains a **switch table** (also known as a MAC address table or CAM table). 

Conceptually similar to a network-layer routing table, a switch table entry contains:
1.  **MAC address of the host:** The physical address of the device on the network.
2.  **Interface:** The specific switch port to which that host is connected.
3.  **Time to Live (TTL) / Time stamp:** A countdown timer to age out stale entries if a device is disconnected.

***

### 1.9 Switch Self-Learning & Forwarding

A switch populates its forwarding table automatically. The self-learning process works by observing the *source* MAC address of incoming frames. When a frame arrives at an interface, the switch notes the source MAC and the interface it arrived on, recording this mapping in its table.

When making a forwarding decision:
*   If the destination MAC address is **unknown** (not in the table), the switch must **flood** the frame out of all interfaces (except the one it arrived on).
*   If the destination MAC address is **known**, the switch will **selectively send** the frame only out the specific link mapped in the table.

```text
Scenario: Host A sends frame to Host A' (Location currently unknown)

Switch Table (Initially Empty):
+----------+-----------+-----+
| MAC Addr | Interface | TTL |
+----------+-----------+-----+
|          |           |     |
+----------+-----------+-----+

1. Frame arrives from A on Port 1.
2. Switch LEARNS: MAC A is on Port 1.
3. Switch Table updates:
+----------+-----------+-----+
| MAC Addr | Interface | TTL |
+----------+-----------+-----+
|    A     |     1     |  60 |
+----------+-----------+-----+
4. Switch FLOODS frame out ports 2,3,4,5,6 to find A'.
```

***

### 1.10 Frame Filtering/Forwarding Logic

The switch executes a precise algorithm every time a frame is received.

```text
when frame received at switch:
1. record incoming link, MAC address of sending host
2. index switch table using MAC destination address
3. if entry found for destination then {
      if destination on segment from which frame arrived then
          drop frame // No need to forward, it's already on that segment
      else 
          forward frame on interface indicated by entry
   } 
   else {
      flood // forward on all interfaces except arriving interface
   }
```

***

### 1.11 Interconnecting Switches

Self-learning switches can be daisy-chained or connected together hierarchically. The self-learning mechanism scales seamlessly across multiple switches without any required modification to the protocol. If Host A sends a frame to Host G attached to a distant switch, intermediate switches simply learn Host A's MAC address relative to the trunk ports connecting the switches.

```text
      A ---+                           +--- I
           |                           |
      B ---+--- [S1]           [S3] ---+--- H
           |      \             /      |
      C ---+       \           /       +--- G
                  [S4 Core Switch]
                   /
      D ---+      /
           |     /
      E ---+-- [S2]
           |
      F ---+
```

***

### 1.12 Small Institutional Network Architecture

In a real-world institutional setting, switches and routers operate together to form an **IP subnet**. Access switches connect end-user workstations; these connect to a central distribution switch, which also connects critical infrastructure like web and mail servers. Finally, an edge router connects the entire institutional subnet to the external internet.

```text
                      To External Network
                              |
                         +--------+
                         | Router |
                         +---+----+
                             |
                     +-------+-------+
                     |  Core Switch  | --------- [Mail Server]
                     +---+---+---+---+ --------- [Web Server]
                         |   |   |
          +--------------+   |   +--------------+
          |                  |                  |
    +-----+----+       +-----+----+       +-----+----+
    | Access SW|       | Access SW|       | Access SW|
    +--+--+--+-+       +--+--+--+-+       +--+--+--+-+
       |  |  |            |  |  |            |  |  |
     [Host Computers across various physical offices]
```

***

### 1.13 Switches vs. Routers

Both switches and routers are fundamentally store-and-forward networking devices, but they operate at different layers of the OSI model:

*   **Routers:** Are **network-layer** (Layer 3) devices. They examine IP headers and compute forwarding tables using complex routing algorithms (like OSPF or BGP). They route packets based on IP addresses.
*   **Switches:** Are **link-layer** (Layer 2) devices. They examine Ethernet headers and learn forwarding tables reactively through flooding and MAC address observation.

```text
   Router Stack                     Switch Stack
+---------------+               
|  Application  |
+---------------+
|   Transport   |
+---------------+                +---------------+
|  Network (IP) | <---Routes---> |  Network (IP) | (If Layer 3 switch)
+---------------+                +---------------+
|   Link (MAC)  | <-Forwards->   |   Link (MAC)  |
+---------------+                +---------------+
|   Physical    |                |   Physical    |
+---------------+                +---------------+
```

***

### 1.14 Virtual LANs (VLANs): Motivation

As a LAN grows, placing all hosts in a single broadcast domain introduces significant problems. 
1.  **Scaling Constraints:** All layer-2 broadcast traffic (ARP requests, DHCP queries, unknown MAC floods) must traverse the entire physical LAN, wasting bandwidth and CPU cycles on end hosts.
2.  **Administrative & Security Issues:** Without logical isolation, privacy is compromised. Furthermore, if a Computer Science (CS) user physically moves their office into an Electrical Engineering (EE) building, they will be attached to the EE switch but likely still need to be logically grouped with the CS subnet for access rights.

```text
Physical Topology dictates logical grouping (Without VLANs)

      [ Router ]
          |
    +-----+-----+
    |Core Switch|
    +--+----+---+
       |    |
       |    +-------------+
       |                  |
 +-----+----+       +-----+----+
 | CS Switch|       | EE Switch|
 +----------+       +----------+
```

***

### 1.15 Port-based VLANs

**VLANs (Virtual Local Area Networks)** solve this by allowing switches to define *multiple virtual LANs* over a single physical LAN infrastructure.

In a **port-based VLAN**, network administrators group specific switch ports via management software. A single 16-port switch could be partitioned so that ports 1-8 act as an isolated "EE Switch" and ports 9-16 act as an isolated "CS Switch".

*   **Traffic Isolation:** Frames originating in ports 1-8 cannot reach ports 9-16 at Layer 2. 
*   **Dynamic Membership:** Ports can be dynamically reassigned without moving cables.
*   **Inter-VLAN Forwarding:** Because the VLANs are isolated broadcast domains, they act as separate IP subnets. Moving traffic *between* VLANs requires a router, just as communicating between distinct physical switches would.

```text
Single Physical Switch acting as multiple virtual switches:

    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    | 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14|15|16|
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
     \_____________________/ \_____________________/
       VLAN 10 (EE Dept)        VLAN 20 (CS Dept)
             |                        |
             +--------[ Router ]------+ (For Inter-VLAN Routing)
```

***

### 1.16 VLANs Spanning Multiple Switches

When VLANs stretch across multiple physical switches (e.g., the EE department spans two buildings), switches must be connected using a **trunk port**. 

A trunk link carries frames belonging to multiple VLANs. Because a standard Ethernet frame does not contain any information about which VLAN it belongs to, a mechanism is needed to identify the frame's origin so the receiving switch knows which virtual LAN to broadcast it to.

```text
   Switch 1 (Bldg A)                      Switch 2 (Bldg B)
+-------------------+                  +-------------------+
|EE|EE|EE|CS|CS| CS |                  |EE|CS|EE|EE|CS| CS |
+-++-++-++-++-++-T--+                  +-T+-++-++-++-++----+
                 |                       |
                 +=======================+
                 TRUNK LINK (Carries both EE & CS traffic)
```

***

### 1.17 802.1Q VLAN Frame Format

To solve the trunking issue, the IEEE **802.1Q** protocol modifies the standard 802.1 Ethernet frame before it traverses a trunk port. It inserts a 4-byte VLAN tag directly after the Source MAC address.

This addition consists of a 2-byte **Tag Protocol Identifier** (value `0x8100`, indicating an 802.1Q frame) and a 2-byte **Tag Control Information** block (which houses a 12-bit VLAN ID and a 3-bit priority field). Because the frame size and contents change, the switch must recompute the CRC before placing the tagged frame onto the trunk.

```text
Standard 802.1 Frame:
+----------+---------------+----------------+------+----------------+-----+
| Preamble | Dest. Address | Source Address | Type | Payload Data   | CRC |
+----------+---------------+----------------+------+----------------+-----+

802.1Q Tagged Frame:
+----------+---------------+----------------+------|------+---------+-----+
| Preamble | Dest. Address | Source Address | TAG  | Type | Payload | CRC*|
+----------+---------------+----------------+------|------+---------+-----+
                                              |
      +---------------------------------------+
      |
      v
+-------------+-------------+
| TPID (8100) | TCI (VLANID)|  *CRC is recomputed by the switch
+-------------+-------------+
```

***

## 2.0 Datacenter Networks

### 2.1 Datacenter Networks Challenges

Modern data centers—powering massive e-business operations (Amazon), content delivery (YouTube, Akamai), and search engines (Google)—house tens to hundreds of thousands of hosts in close physical proximity. 

The primary challenges in these environments include:
*   Supporting multiple applications simultaneously, each serving massive numbers of concurrent clients.
*   Ensuring ultra-high **reliability**; component failure is an hourly occurrence at this scale.
*   Managing and load-balancing traffic to prevent processing, networking, and data bottlenecks.

```text
[ Microsoft 40-ft Container Data Center Conceptual ]
|| Server Rack || Server Rack || Server Rack ||
|| Server Rack || Server Rack || Server Rack ||
|| Server Rack || Server Rack || Server Rack ||
[ Highly dense, closely coupled infrastructure ]
```

***

### 2.2 Datacenter Network Elements

To handle these challenges, datacenters deploy a rigid, hierarchical switch topology (often utilizing Clos or Fat-Tree architectures) rather than a flat network:
*   **Server Racks:** Physical cabinets holding 20-40 server blades (hosts).
*   **Top of Rack (TOR) switch:** A switch located in every rack connecting the server blades via 40-100Gbps Ethernet.
*   **Tier-2 switches:** Aggregation switches connecting to approximately 16 TOR switches below them.
*   **Tier-1 switches:** Core switches connecting to the Tier-2 switches below.
*   **Border routers:** Devices interfacing the datacenter network with the external internet.

```text
                 [ Border Routers ]
                   /             \
            [ Tier-1 ]         [ Tier-1 ]
            /       \           /      \
      [Tier-2]     [Tier-2] [Tier-2]   [Tier-2]
        /  \         /  \     /  \       /  \
     [TOR][TOR]   [TOR][TOR][TOR][TOR] [TOR][TOR]
      | |  | |     | |  | |  | |  | |   | |  | |
     [Racks of Server Blades ...................]
```

***

### 2.3 Multipath Routing

Notice the highly meshed topology in the diagram above. Datacenters require **rich interconnection** among switches and racks. This multipath routing provides:
1.  **Increased throughput:** Using protocols like ECMP (Equal-Cost Multi-Path), a flow between rack 1 and rack 11 can utilize multiple distinct routes to avoid congestion on a single link.
2.  **Increased reliability:** Redundancy ensures that if a Tier-2 switch fails, a disjoint path through a different Tier-2 switch remains available to reach the destination.

```text
Path 1: Rack 1 -> TOR A -> Tier-2 A -> Tier-1 A -> Tier-2 C -> TOR Z -> Rack 11
Path 2: Rack 1 -> TOR A -> Tier-2 B -> Tier-1 B -> Tier-2 D -> TOR Z -> Rack 11
(Two completely disjoint, redundant paths highlighted in the topology)
```

***

### 2.4 Application-layer Routing

Traffic entering the data center from the outside internet typically passes through a **load balancer**. Operating at the application layer (Layer 7) or transport layer (Layer 4), a load balancer:
1.  Receives external client requests directed at a public IP.
2.  Evaluates current server loads and directs the workload to an optimal internal server node.
3.  Returns the result to the external client, completely hiding the complex internal IP structure of the datacenter from the outside world.

```text
[Client] ---> (Internet) ---> [Load Balancer]
                                    |
            +-----------------------+-----------------------+
            |                       |                       |
      [Web Server A]          [Web Server B]          [Web Server C]
```

***

### 2.5 Protocol Innovations

Traditional Ethernet and TCP/IP were not built for datacenter-level scale. Innovations are occurring at every layer:
*   **Link Layer:** Technologies like **RoCE** (Remote Direct Memory Access over Converged Ethernet) allow servers to read/write memory bypassing the CPU, drastically reducing latency.
*   **Transport layer:** **ECN** (Explicit Congestion Notification) is heavily utilized in specialized congestion control variants like DCTCP (Data Center TCP) to react to network pressure before packets are dropped.
*   **Routing & Management:** **SDN** (Software-Defined Networking) is heavily leveraged to place related services and data physically close to one another (e.g., in the same rack) to minimize expensive Tier-1 and Tier-2 network traversal.

***

## 3.0 Synthesis: A Day in the Life of a Web Request

### 3.1 Overview
To put all networking concepts together, we will trace a seemingly simple scenario: A student attaches a laptop to a campus network and requests a webpage (`www.google.com`). We will identify the protocols involved across all layers of the stack.

### 3.2 Scenario Topology
Our environment involves four major network domains:
1.  **School Network:** Where the laptop resides (`68.80.2.0/24`).
2.  **Comcast Network:** The intermediate ISP routing traffic (`68.80.0.0/13`).
3.  **Google's Network:** The destination network hosting the web server (`64.233.160.0/19` and specific IP `64.233.169.105`).
4.  **DNS Server:** A local or ISP-provided server resolving domain names.

```text
[Browser/Laptop]
       |
  (School Switch) ---- [School Router/Gateway]
                               |
                        (Comcast Network) --------- [DNS Server]
                               |
                       (Google Network)
                               |
                          [Web Server]
```

***

### 3.3 Connecting to the Internet (DHCP)

Before the laptop can request a webpage, it needs an IP address, the address of its default gateway (first-hop router), and the address of a DNS server. It utilizes **DHCP** (Dynamic Host Configuration Protocol) for this.

1.  The laptop generates a DHCP Request. This is encapsulated in **UDP**, then **IP**, and finally **802.3 Ethernet**.
2.  Because the client lacks an IP address, the Ethernet frame's destination MAC is set to the broadcast address (`FF:FF:FF:FF:FF:FF`).
3.  The LAN switch broadcasts this frame. The school router, running a DHCP server, receives it.
4.  The router extracts the payload (demuxing Ethernet -> IP -> UDP -> DHCP).
5.  The DHCP server formulates a **DHCP ACK** containing the client's new IP address, the IP of the first-hop router, and the DNS server's IP.
6.  The router encapsulates this ACK and sends it back to the client via switch learning.

```text
Laptop DHCP Request Encapsulation:
+------------------------------------------+
| MAC (Broadcast) | IP (0.0.0.0) | UDP | DHCP |
+------------------------------------------+
```

***

### 3.4 ARP (Before DNS, Before HTTP)

Now the laptop has an IP address. To access `www.google.com`, it must first translate the hostname to an IP address using DNS. However, the DNS server is outside the local subnet. To send the DNS packet out of the local network, the laptop must send the frame to the *default gateway* (the first-hop router). 

To send an Ethernet frame to the router, the laptop needs the router's physical MAC address. It uses **ARP** (Address Resolution Protocol).
1.  The laptop broadcasts an **ARP query** ("Who has the IP address of the router?").
2.  The router receives the query and replies with an **ARP reply** containing its MAC address.
3.  The laptop now maps the router's IP to the router's MAC address and can proceed.

```text
[Laptop] ---- ARP Broadcast (Who is 68.80.2.1?) ----> [Switch] ---> [Router]
[Laptop] <--- ARP Reply (My MAC is 00:1A:2B...) ----- [Switch] <--- [Router]
```

***

### 3.5 Using DNS

With the router's MAC address obtained, the laptop creates the DNS query for `www.google.com`.
1.  The DNS query is encapsulated in UDP, then IP, then an Ethernet frame addressed to the *router's MAC address* (but destined for the *DNS server's IP address*).
2.  The router receives the frame, strips the Ethernet header, looks at the destination IP, and forwards the IP datagram into the Comcast network.
3.  The packet is routed through intermediate networks using routing algorithms (**RIP, OSPF, IS-IS, and/or BGP**).
4.  The DNS server receives the query, demuxes it, and replies to the client with the IP address of Google's web server (`64.233.169.105`).

```text
Laptop DNS Query Packet:
Dest MAC: [Router MAC] | Dest IP: [DNS Server IP] | UDP | DNS Query: www.google.com
```

***

### 3.6 TCP Connection Carrying HTTP

With the destination IP address known, the client prepares the HTTP request. HTTP relies on a reliable transport layer, so it opens a **TCP socket**.

1.  The laptop sends a **TCP SYN** segment (Step 1 of the 3-way handshake) destined for Google's IP address. This is routed inter-domain across the internet.
2.  Google's web server receives the SYN and replies with a **TCP SYNACK** segment (Step 2).
3.  The laptop receives the SYNACK and responds with an ACK (Step 3). The **TCP connection is established!**

```text
TCP 3-Way Handshake Flow:
[Client] ------ SYN (Synchronize) -----> [Google Web Server]
[Client] <--- SYN-ACK (Acknowledge) ---- [Google Web Server]
[Client] ------ ACK (Acknowledge) -----> [Google Web Server]
```

***

### 3.7 HTTP Request/Reply

Finally, the data exchange can occur.
1.  The laptop sends the **HTTP GET request** into the established TCP socket.
2.  The IP datagram containing the HTTP request is routed over the Comcast network to the Google data center.
3.  Google's web server generates the webpage data and responds with an **HTTP reply**.
4.  The IP datagram containing the HTTP payload is routed back to the client.
5.  The browser parses the HTML and the web page is *finally* displayed to the student.

```text
Laptop HTTP Payload Routing:
+-------------------------------------------------------+
| Router MAC | Google Server IP | TCP | HTTP GET / HTTP |
+-------------------------------------------------------+
```

***

## 4.0 Chapter 6: Summary

### 4.1 Key Takeaways
We have completed our journey down the protocol stack. The Link layer is responsible for the actual node-to-node transfer of frames across a single physical medium or switched network. 

The principles behind data link layer services include critical functions like **error detection and correction** (via CRCs), sharing a broadcast channel via **multiple access protocols** (like CSMA/CD), and **link layer addressing** (MAC addresses) which are distinct from network-layer IP addresses.

We explored dominant link-layer technologies, focusing heavily on **Ethernet** evolution, the operations of self-learning **switched LANs**, and logical isolation using **VLANs**. Finally, synthesizing these components via the "Day in the Life of a Web Request" demonstrates how DHCP, ARP, DNS, TCP, HTTP, and physical switching seamlessly interact to provide the modern internet experience.
