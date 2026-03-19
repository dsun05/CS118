## 1.0 Link Layer

### 1.1 Link layer: introduction

The **Link Layer** is the second layer of the OSI network model, responsible for node-to-node data transfer between directly connected devices. It acts as the critical intermediary between the physical hardware and the network layer (like IP). Understanding the principles behind link-layer services involves several key concepts:

*   **Data framing:** The process of encapsulating network layer datagrams into packets called "frames." This adds necessary header and trailer information to delineate where a message starts and ends.
*   **Error detection, correction:** Because physical channels are susceptible to noise and interference, the link layer uses mechanisms like **CRC (Cyclic Redundancy Check)** to detect if a frame's bits were flipped or corrupted in transit. If an error is found, the frame is usually dropped or, in some protocols, corrected.
*   **Sharing a broadcast channel:** When multiple devices share a single communication medium (like air in Wi-Fi or a shared physical wire), the link layer must manage who gets to "speak" to avoid simultaneous transmissions that scramble signals. This is handled by **Multiple Access Control (MAC)** protocols.
*   **Link layer addressing:** Unlike IP addresses that dictate end-to-end routing across the globe, MAC addresses are used at the link layer to identify specific network interfaces on a local segment. 

These concepts are practically implemented in Local Area Networks (LANs) through technologies like **Ethernet** and Virtual LANs (VLANs).

***

### 1.2 Delivering packet at link layer

At the link layer, communication between devices generally falls into three basic delivery models based on the intended audience of the packet:

*   **Unicast:** A one-to-one communication model where a single source sends a frame to a single, specific destination MAC address. 
*   **Broadcast:** A one-to-all communication model. The source sends a single frame addressed to a special broadcast MAC address (e.g., `FF-FF-FF-FF-FF-FF`), and every node on the local subnet receives and processes it.
*   **Multicast:** A one-to-many communication model. A source sends a frame to a specific group of interested receivers, rather than everyone. 

*Why do we need these?* Unicast is efficient for direct communication (like fetching a web page). Broadcast is necessary for discovery protocols (like asking "Who has this IP address?"). Multicast is vital for efficient streaming to multiple users without overwhelming the network with duplicate unicast streams.

* Visual Aid: Diagram illustrating Unicast (one-to-one), Broadcast (one-to-all), and Multicast (one-to-many) communication models using colored nodes and directional arrows. *
```text
      UNICAST                   BROADCAST                   MULTICAST
   (One-to-One)               (One-to-All)                (One-to-Many)
        
        [D]                        [D]                         [D]
       /                          / | \                       / |  
     /                          /   |   \                   /   |   
   /                          /     |     \               /     |     
 [S] - - [ ]                [S] - -[D]- - [D]           [S] - -[ ]- - [ ]
   \                          \     |     /               \     |     
     \                          \   |   /                   \   |   
       \                          \ | /                       \ |  
        [ ]                        [D]                         [D]

Legend: [S] = Source Node, [D] = Destination Node, [ ] = Uninvolved Node
```

***

### 1.3 Broadcast

When broadcasting data across a network, there are two primary philosophical approaches:
1.  **Replicate at source:** The sending node creates a separate copy of the packet for every single node in the network. This is highly inefficient and requires the source to know every destination.
2.  **In-network replication:** The source sends one packet, and the network infrastructure (switches/routers) replicates the packet out of all other ports. This is how modern link-layer broadcast works.

**Flooding (FYI):** Flooding is a brute-force broadcast strategy where every node simply forwards a received packet out of all connected links (except the one it arrived on). To prevent endless routing loops, networks use techniques like:
*   **Reverse Path Forwarding (RPF):** Only forwarding a broadcast packet if it arrived on the link that represents the shortest path back to the sender.
*   **Spanning Tree:** A protocol that mathematically disables redundant links in a network to create a loop-free logical topology.

***

### 1.4 Multicast (FYI)

Multicast efficiently delivers data to a specific group of subscribers. 
*   **Address range:** In IP networks, multicast relies on Class-D IP addresses, which range from `224.0.0.0` to `239.255.255.255`. Link layer MAC addresses have corresponding mappings for these IPs.
*   **Protocols:** Managing who is in a multicast group requires specialized protocols such as IGMP (Internet Group Management Protocol) for local group membership, and routing protocols like DVMRP or PIM for determining paths across networks.

***

### 1.5 Data framing (FYI)

To separate continuous streams of bits into discrete "frames," the link layer uses stuffing techniques to mark the start and end of a packet without confusing payload data for a boundary marker.
*   **HDLC Byte Stuffing:** A bit-oriented protocol. If the payload naturally contains five `1` bits in a row (which might look like a boundary flag), the transmitter "stuffs" an extra `0` bit. The receiver removes it. This results in a 20% worst-case overhead.
*   **PPP Byte Stuffing:** A byte-oriented protocol (Point-to-Point Protocol). It uses a special escape byte to indicate that the following byte is data, not a control flag. This can result in up to a 200% worst-case overhead if every payload byte requires escaping.
*   **COBS (Consistent Overhead Byte Stuffing):** A byte-oriented technique that guarantees a fixed, trivial overhead (usually <1%) by encoding the payload such that the boundary byte value never appears in the data itself.

***

### 1.6 Medium Access Links and Protocols

Links generally fall into two categories: **point-to-point** (a direct wire between two specific machines) and **broadcast** (a shared medium, like a single wire shared by multiple computers, or shared airspace in Wi-Fi).

When a **broadcast** channel is shared by multiple hosts, collisions will occur if two nodes speak at once. 
*   *Pros and cons of a broadcast channel:* It is highly efficient for discovery and group messaging, and requires less wiring. However, the major con is the necessity for complex rules to coordinate access and prevent overlapping signals from garbling the data.

To solve this, we use **Multiple Access Control (MAC) protocols**. They are divided into three main classes:
1.  **Channel partitioning:** Dividing the channel resource into smaller, dedicated "pieces." Examples include **FDMA** (Frequency Division Multiple Access), **TDMA** (Time Division Multiple Access), and **CDMA** (Code Division Multiple Access).
2.  **Random access:** Nodes transmit whenever they want and handle collisions if they occur. Examples include **ALOHA**, **CSMA/CD**, and modern **Ethernet**.
3.  **Taking turns:** Nodes pass a distinct permission token around to determine who gets to speak. Examples include Token ring or Token passing.

***

### 1.7 Random Access MAC Protocols

Random access protocols are decentralized and highly efficient at low traffic volumes. Their basic functions dictate that a node should:
1.  Send a frame when it has data ready.
2.  Retransmit the frame at a later, usually random, time if a collision happens.

The most historically significant and widely used protocols in this category are **ALOHA** (and its variant **Slotted ALOHA**) and **CSMA/CD (Carrier Sense Multiple Access with Collision Detection)**, which powers Ethernet.

***

### 1.8 Random access: slotted ALOHA

**Slotted ALOHA** was an early improvement over basic ALOHA that discretized time into "slots" to reduce the window of vulnerability for collisions.

*   **Assumptions:** All frames are the exact same size; time is divided into equal-sized slots exactly matching the time it takes to transmit one frame; nodes are perfectly synchronized; nodes are only allowed to start transmission at the exact beginning of a slot; and if two nodes transmit in the same slot, a collision is immediately detected by all.
*   **Efficiency logic:** 
    *   Let $p$ be the probability that a node transmits in a given slot, and $N$ be the number of active nodes.
    *   The probability that a *specific* node succeeds is the probability it transmits ($p$) multiplied by the probability that no other nodes transmit: $p(1-p)^{N-1}$.
    *   The probability that *any* of the $N$ nodes succeeds in a given slot is: $N \times p(1-p)^{N-1}$.
    *   To find the maximum possible efficiency, we calculate the limit of this expression as $N$ approaches infinity. This mathematically yields $1/e$, which is approximately $0.37$ (or 37%). This means at maximum efficiency, only 37% of the slots successfully carry data.

***

### 1.9 Random access: ALOHA efficiency

The efficiency limits demonstrate why better protocols were needed:
*   **Slotted ALOHA max efficiency = 0.37:** By forcing nodes to only start at slot boundaries, collisions only occur if nodes pick the exact same slot.
*   **Unslotted (Pure) ALOHA max efficiency = 0.18:** Because nodes can transmit at any precise millisecond, a transmission can collide with the end of a previous transmission *or* the beginning of a future one. This doubles the vulnerability window, dropping efficiency to $1/2e \approx 0.18$ (18%).

***

### 1.10 Quick question (Slotted ALOHA)

*Scenario:* Consider a network with exactly four nodes using slotted ALOHA. The probability that any single node transmits at a given time is $p$.

*   **Probability of all four nodes transmitting simultaneously:** Since each node is independent, we multiply their individual probabilities: $p \times p \times p \times p = p^4$. The probability that the *other* nodes do not transmit (though not applicable for a 4-node total transmission scenario) is mathematically $(1-p)$.
*   **Probability of a specific node succeeding:** To succeed, the specific node must transmit ($p$), and the remaining three nodes must *not* transmit $(1-p)^3$. Therefore, $P_{success} = p(1-p)^3$. *(Note: The outline references $p(1-p)^8$, which would correspond to an environment with 9 nodes where 1 transmits and 8 defer).*

***

## 2.0 CSMA/CD

### 2.1 CSMA (carrier sense multiple access)

**CSMA (Carrier Sense Multiple Access)** improves upon ALOHA by introducing the "listen before transmit" philosophy. It acts on the polite human principle: "don’t interrupt others!"
*   **Idle channel:** If the node listens and hears no other transmissions, it transmits its entire frame.
*   **Busy channel:** If the channel is sensed busy, the node defers its transmission. 

How does a node behave when it finds the channel busy? There are three strategies:
*   **1-persistent CSMA:** The node continuously listens and retries immediately the millisecond the channel becomes idle.
*   **p-persistent CSMA:** The node retries immediately with probability $p$, and defers with probability $1-p$.
*   **Non-persistent CSMA:** The node goes to sleep for a random interval before checking the channel again.

**Collision issues:** Even with carrier sensing, collisions happen due to propagation delay (Node B starts transmitting before Node A's signal reaches it) and the **Hidden Terminal Problem** (where two transmitting nodes cannot physically hear each other, but their destination in the middle can hear both, causing a collision at the receiver).

* Visual Aid: Diagram depicting a spatial layout of four computer nodes and a space-time graph showing a signal collision between two transmitting nodes over time. *
```text
Spatial Layout:
  [Node A]          [Node B]          [Node C]          [Node D]

Space-Time Graph (Propagation Delay causing Collision):

Time |
 t0  |  [A transmits] 
     |    \                                            / [D transmits]
     |     \                                          /
     |      \                                        /
     |       \       <SIGNAL PROPAGATION>           /
     |        \                                    /
     |         \                                  /
     |          \                                /
 t_c |           \     [COLLISION HAPPENS!]     /
     |            \XXXXXXXXX[BOOM]XXXXXXXXXXXXX/
     |             \XXXXX GARBLED DATA XXXXXXX/
     V              \                        /
```

***

### 2.2 CSMA/CD (collision detection)

**CSMA/CD (Carrier Sense Multiple Access with Collision Detection)** is an enhancement to CSMA used primarily in wired Ethernet networks. It not only listens before talking but also listens *while* talking.

*   **The Principle:** If a node detects a collision mid-transmission, it immediately aborts sending the rest of the frame. This rapidly reduces channel wastage compared to basic CSMA, which would blindly finish transmitting the corrupted frame.
*   **Wired vs Wireless:** Collision detection is easy in wired LANs because transceivers can simultaneously measure transmitted and received signal voltages. It is extremely difficult in wireless LANs (Wi-Fi) because a radio's own local transmission signal overwhelmingly drowns out any faint incoming signals from other devices.
*   **The Algorithm Steps:**
    1. Sense channel; wait if necessary; when idle, transmit and monitor.
    2. If a collision is detected, immediately abort transmission and send a **Jam Signal** (a short burst of data to ensure all other nodes detect the collision).
    3. Update the collision counter ($n++$).
    4. Execute **Binary Exponential Backoff**: Delay for a duration of $K$ slot times (where 1 slot = 512 bits transmission time). The node chooses $K$ randomly from a window of $\{0, 1, 2, ..., 2^n - 1\}$. 
    5. As collisions mount, the random wait window grows exponentially, reducing traffic congestion.

***

### 2.3 Quick question (CSMA/CD)

*Scenario:* Two nodes are trying to transmit frames at the exact same time $t=0$ on a 512kb/s link. Assume propagation delay is negligible, and the delay in detecting a collision and aborting is $t_0$.

*   **Probability of simultaneous failure:** Both nodes detect a collision at $t=0$, send jam signals, and abort by $t_0$. At $t_0$, each node chooses $K$ from $\{0, 1\}$ (since $n=1$). The probability they both pick the same number (both 0 or both 1) and collide again is $1/4 + 1/4 = 1/2$. The probability that two nodes *both fail* to transmit immediately (meaning they pick different backoff times, but we are looking at total failure rate of the medium at that instant) is related to their collision matrix. 
*   **Max waiting time after the second collision:** After the second collision ($n=2$), the backoff window is $\{0, 1, 2, 3\}$. The maximum choice is $K=3$. At a 512kb/s rate, transmitting 512 bits takes exactly 1ms. Thus, the max waiting time is $3 \times 1ms = 3ms$.

***

## 3.0 LAN & Switch

### 3.1 Ethernet

**Ethernet** is the dominant wired LAN technology in the world. 
*   **Connectionless and unreliable:** Ethernet does not establish a handshake before sending, and it does not send acknowledgments. It relies on upper-layer protocols (like TCP) to guarantee reliable data transfer. If an Ethernet frame is dropped, Ethernet itself does not recover it.
*   **MAC Protocol:** It uses CSMA/CD combined with binary exponential backoff.
*   **Wireless Networks:** We cannot easily use standard CSMA/CD in wireless networks due to the hidden terminal problem and the inability of a radio antenna to listen for faint remote collisions while blasting out its own signal. (Wireless uses CSMA/CA instead).
*   **Switch-based Ethernet:** Modern Ethernet uses switches rather than shared hubs. This fundamentally changes the network: there is no longer a true shared "broadcast channel" (avoiding physical collisions entirely). Instead, the switch relies on a **Self-learning algorithm** to achieve true **Plug-and-play (PnP)** functionality.

*Key distinction:*
*   **Routing table:** Maps IP addresses to next-hop routers (Network Layer).
*   **Switch table:** Maps MAC addresses to physical switch ports (Link Layer).
*   **ARP table:** Maps IP addresses to MAC addresses (Bridge between Network and Link layers).

***

### 3.2 MAC address

Every network interface card (NIC) is given a unique identifier called a **MAC Address** (Media Access Control address).
*   **Allocation:** These are centrally allocated and managed by the IEEE to ensure global uniqueness.
*   **Flat Address Space:** Unlike IP addresses which are hierarchical (changing based on what subnet you move to), MAC addresses are "flat" and permanently burned into the hardware. This promotes device portability.
*   **Format:** A 48-bit address typically represented as six hexadecimal pairs (e.g., `AA-BB-CC-DD-EE-FF`).
*   **Broadcast Address:** The special address `FF-FF-FF-FF-FF-FF` targets every node on the local LAN.

***

### 3.3 Ethernet Frame

The structure of an **Ethernet Frame** is strictly defined to ensure proper synchronization and collision detection.

*   **Min frame size (64 Bytes):** Required to ensure that a transmitting node is still sending data by the time a collision signal propagates back from the furthest end of the network. If frames were smaller, the node might finish sending and assume success before the collision report arrived.
*   **Max frame size (1514 Bytes):** Enforced to ensure fair media sharing so that one node cannot monopolize the shared channel by sending an infinitely long frame.

* Visual Aid: Block diagram of an Ethernet Frame highlighting the Preamble (8 bytes), Dest Address (6), Source Address (6), Frame Type (2), Data Payload (46-1500), and CRC (4). *
```text
 +---------------+---------------+---------------+--------+----------------------+-------+
 | Preamble      | Dest Address  | Source Address| Frame  | Data Payload         | CRC   |
 | (8 Bytes)     | (6 Bytes)     | (6 Bytes)     | Type   | (46 - 1500 Bytes)    | (4 B) |
 |               |               |               | (2 B)  |                      |       |
 +---------------+---------------+---------------+--------+----------------------+-------+
                 <----------------------- Header -------------------------------->
```

***

### 3.4 Ethernet CSMA/CD

The full lifecycle of an Ethernet transmission using CSMA/CD is as follows:
1.  The NIC receives a datagram from the network layer (IP) and creates an Ethernet frame.
2.  The NIC senses the channel. If idle, it starts transmission immediately. If busy, it defers and waits until the channel is clear.
3.  If the NIC successfully transmits the entire frame without detecting any other signal, transmission is complete.
4.  If the NIC detects another transmission (a collision) while sending, it instantly aborts and sends a jam signal.
5.  After aborting, the NIC enters the binary exponential backoff phase, waits its randomly calculated time, and then returns to step 2.

***

### 3.5 Switch

A **Switch** is an intelligent link-layer device acting as the central nexus for a modern LAN. 
*   **Function:** It examines the destination MAC address of every incoming frame and selectively forwards it *only* out of the specific physical port leading to that destination, isolating traffic and preventing collisions.
*   **Mechanism:** It uses **Store-and-forward**, meaning it completely buffers the incoming frame into memory to check its integrity (via CRC) before pushing it out to the destination link.
*   **Switch Table:** This internal database dictates forwarding rules. It is built dynamically via a **self-learning algorithm**. When a frame arrives, the switch records the *source* MAC address, the incoming port interface, and a timestamp. If the destination MAC is unknown, it floods the frame; as devices reply, the switch learns their locations.

***

### 3.6 ARP: address resolution protocol

Because the Network layer (IP) and Link layer (MAC) use entirely different addressing schemes, devices need a translation mechanism. This is the **ARP (Address Resolution Protocol)**.
*   **Purpose:** To determine a node's physical MAC address when only its logical IP address is known.
*   **ARP Table:** Every host and router maintains this cache, storing `<IP address; MAC address; TTL (Time to Live)>` mappings.
*   **Characteristics:** It operates invisibly in the background, making it **Plug-and-play (PnP)**. It utilizes a **soft-state design**, meaning cached entries automatically expire and delete themselves after their TTL times out unless they are refreshed by new traffic, ensuring outdated mappings don't break the network when hardware is swapped.

***

### 3.7 ARP: send an IP packet in the same subnet

When Host A needs to send a packet to Host B on the exact same local network, and A does not have B's MAC address:
1.  Host A broadcasts an ARP query to the entire LAN. The frame's destination MAC is set to `FF-FF-FF-FF-FF-FF`. The payload asks, "Who has IP_B? Tell IP_A."
2.  Every node on the network receives and processes this query.
3.  Host B recognizes its own IP address in the query. B constructs an ARP reply containing its MAC address and sends it as a *unicast* frame directly back to Host A's MAC.
4.  Host A receives the reply, caches the IP-to-MAC pair in its ARP table, and proceeds to send its actual data payload.

***

### 3.8 ARP: send an IP packet across subnets

Sending data across different networks (e.g., to the internet) requires **Subnet Routing** and the use of a router (default gateway).

* Visual Aid: Network topology diagram showing Host A sending a packet to Host B across a Router (R). It details the IP/Ethernet/Physical layer encapsulations and the specific IP and MAC address pairs required at each hop. *
```text
                 [Router R]
               /            \
    IP: 111...110          IP: 222...220
    MAC: E6-E9...          MAC: 1A-23...
     /                        \
    /                          \
[Host A]                    [Host B]
IP: 111...111               IP: 222...222
MAC: 74-29...               MAC: 49-BD...

HOP 1 (A to Router):
Src IP: 111...111 (A)   | Dest IP: 222...222 (B)
Src MAC: 74-29... (A)   | Dest MAC: E6-E9... (R's Left Interface)

HOP 2 (Router to B):
Src IP: 111...111 (A)   | Dest IP: 222...222 (B)
Src MAC: 1A-23... (R)   | Dest MAC: 49-BD... (B)
```

**Process overview:**
1.  Host A checks its IP routing table and realizes Host B is on a different subnet.
2.  Because it's a gateway path, A uses ARP to look up the MAC address of its *Default Gateway (Router)*, not Host B.
3.  Host A creates an Ethernet frame addressed to the Router's MAC, but the inner IP packet remains addressed to Host B's IP.
4.  The Router receives the frame, removes the Link-layer Ethernet header, and inspects the Network-layer IP destination.
5.  The router realizes the packet is not for itself, checks its routing table, and determines the outbound interface. It then performs an ARP lookup for Host B on the destination subnet, creates a *new* Ethernet frame with its own outbound MAC as the source and B's MAC as the destination, and forwards the payload.

***

### 3.9 Router vs. Switch

Both routers and switches connect network segments, but they operate at fundamentally different abstraction layers.
*   **Commonalities:** Both are store-and-forward devices that use buffers and tables to make routing/switching decisions.
*   **Switches:** Link layer devices. They only examine Ethernet MAC headers and do not understand IPs.
*   **Routers:** Network layer devices. They examine IP headers and connect entirely different subnets or networks together.

This layered approach defines network design philosophies:
*   **Circuit-switched networks (e.g., legacy telecom):** Establishes a rigid, dedicated path before sending data. Processing is simple ($O(1)$ complexity) based on connection labels, but it is highly **vulnerable to link/node failures** because the entire path drops if one wire breaks.
*   **Packet-switched networks (e.g., the Internet):** Connectionless routing based on IP headers. Requires longest prefix matching at every router ($O(N)$ complexity), but is incredibly **robust to link/node failures** because individual packets can dynamically route around damage.

***

### 3.10 Tables we learnt

To recap the logical databases used by network devices:
*   **Routing table:** Resides in routers (and hosts); uses IP addresses to determine the outbound interface/next-hop router.
*   **Forward (Switch) table:** Resides in switches; uses MAC addresses to determine which physical port to send a frame down.
*   **ARP table:** Resides in routers and hosts; maps logical IP addresses to physical MAC addresses.

***

### 3.11 Network devices

A quick classification of network equipment based on the OSI model layer they operate on:
*   **Repeater:** Operates at the Physical (PHY) layer. A "dumb" device that simply receives analog signals (bits in) and amplifies/repeats them out to all other ports. It has no software intelligence.
*   **Hub:** A multi-port repeater. It operates at the physical layer (sometimes conceptually considered primitive link layer). Packets coming in one port are mindlessly broadcasted out to all other ports, leading to massive collision domains.
*   **Switch:** Operates at the Link layer. Intelligently forwards frames based on MAC addresses, entirely replacing hubs in modern networks.
*   **Router:** Operates at the Network layer. Connects separate LANs over vast geographic distances based on IP routing protocols. 

***

### 3.12 FYI

For deep technical understanding, advanced courses may require mathematical derivations of protocol efficiency. Students should review the mathematical proofs for:
*   Derivation of slotted ALOHA efficiency ($1/e \approx 37\%$).
*   Derivation of unslotted (pure) ALOHA efficiency ($1/2e \approx 18\%$).
*   Derivation of CSMA/CD efficiency models.
