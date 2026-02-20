## 1.0 Network Layer: Data Plane (Router Internals & Queueing)

### 1.1 Topic Overview
The Network Layer is fundamentally divided into two interacting components: the **data plane** and the **control plane**. 
*   **Data Plane**: This is the local, per-router function responsible for forwarding datagrams from a router's input link to the correct output link. It operates at an extremely fast, nanosecond timeframe and is typically implemented in hardware.
*   **Control Plane**: This is the network-wide logic that determines the end-to-end path a packet takes from the source to the destination. It operates in a millisecond timeframe and is typically implemented in software.

This section explores what resides inside a router, including input ports, switching fabrics, output ports, buffer management, and scheduling. It also covers the core concepts of the **`IP`** (Internet Protocol), generalized forwarding paradigms such as **`SDN`** (Software-Defined Networking), and the role of middleboxes in modern networks.

***

### 1.2 Router architecture overview

```text
                        +----------------------+
                        |  Routing Processor   |  <--- Control Plane (Software, ms)
                        +----------+-----------+
                                   |
===================================|=======================================
                                   v
+---------------+       +----------------------+       +---------------+
|               |       |                      |       |               |
|  Input Ports  | ====> | High-Speed Switching | ====> | Output Ports  |  <--- Data Plane (Hardware, ns)
|               |       |       Fabric         |       |               |
+---------------+       +----------------------+       +---------------+
```

The router architecture separates the high-level decision-making from the high-speed execution of those decisions. 
*   **Routing Processor**: This is the brain of the router. It runs the routing protocols, maintains routing tables and attached link state information, and computes the forwarding table.
*   **Router Input Ports**: These receive the physical signal, process link-layer framing, and perform the crucial lookup function to determine which output port a packet should be directed to.
*   **High-Speed Switching Fabric**: The internal network of the router that physically moves packets from input ports to output ports.
*   **Router Output Ports**: These store packets received from the switching fabric and transmit them on the outgoing link.

***

### 1.3 Input port functions

```text
   Physical Layer        Link Layer                Decentralized Switching
 +----------------+   +--------------+   +--------------------------------------+
 | Line           |   | Protocol     |   | Lookup, Forwarding, & Queueing       |
 | Termination    |-->| (e.g.        |-->|                                      |--> To Switch Fabric
 | (Bit reception)|   |  Ethernet)   |   | (Match + Action using local memory)  |
 +----------------+   +--------------+   +--------------------------------------+
```

Input ports perform several sequential functions to process incoming traffic at line speed:
*   **Physical Layer**: Handles bit-level reception of the incoming signal.
*   **Link Layer**: Handles framing and error checking (e.g., Ethernet protocols).
*   **Decentralized Switching**: To prevent the routing processor from becoming a bottleneck, input ports maintain a shadow copy of the forwarding table. 
    *   They use a **`match plus action`** paradigm: using header field values to look up the appropriate output port in the local memory.
    *   The primary goal is to complete this processing at "line speed" (the physical transmission rate of the link).
    *   **Input Port Queuing**: If packets arrive and are processed faster than the switching fabric can move them to the output ports, they must be temporarily queued in the input buffers.
*   **Destination-based Forwarding**: The traditional method where the forwarding decision is made based *only* on the packet's destination IP address.
*   **Generalized Forwarding**: A modern approach (used in SDN) where forwarding decisions can be made based on *any* set of header field values (e.g., source IP, MAC address, port numbers).

***

### 1.4 Destination-based forwarding

```text
Destination Address Range                                       Link Interface
---------------------------------------------------------       --------------
11001000 00010111 00010000 00000000 through
11001000 00010111 00010000 00000100                                   0

11001000 00010111 00010000 00000111 through
11001000 00010111 00011000 11111111                                   3

11001000 00010111 00011001 00000000 through
11001000 00010111 00011111 11111111                                   2

otherwise                                                             3
```

Because IPv4 addresses encompass over 4 billion possibilities, routers do not maintain a direct mapping for every single IP address. Instead, they group destination addresses into ranges and map these ranges to specific outbound link interfaces.

***

### 1.5 Longest prefix matching

```text
Forwarding Table Example:
Prefix Match                                      Link Interface
------------------------------------------        --------------
11001000 00010111 00010*** ********                     0
11001000 00010111 00011000 ********                     1
11001000 00010111 00011*** ********                     2
otherwise                                               3

Packet IP:  11001000 00010111 00011000 10101010  --> Matches Prefix 1 AND 2.
Decision:   Uses Prefix 1 (Link Interface 1) because it is the LONGEST match.
```

When IP address ranges overlap, routers use **longest prefix matching**. The router compares the packet's destination IP against the table and selects the entry with the longest continuous matching bit pattern (prefix).

To perform this at hardware line speeds, routers utilize **Ternary Content Addressable Memories (TCAMs)**. 
*   **Content Addressable**: Unlike standard RAM where you provide an address to get data, you present the data (the IP address) to the TCAM, and it retrieves the matching memory address (the forwarding rule) in a single clock cycle, regardless of how large the routing table is.
*   High-end routers (like the Cisco Catalyst series) can store upwards of 1 million routing entries in their TCAMs for instantaneous lookups.

***

### 1.6 Switching fabrics

```text
     N Input Ports                        N Output Ports
   +---------------+                    +---------------+
==>|               |   +------------+   |               |==>
==>| Queue / Match |-->| High-Speed |-->| Queue / Buff  |==>
==>|               |   | Switching  |   |               |==>
   +---------------+   | Fabric     |   +---------------+
                       +------------+
```

The switching fabric is the actual mechanism that physically transfers a packet from a specific input link to the appropriately determined output link. 
*   **Switching Rate**: This is the rate at which packets can be transferred across the fabric. A well-designed router aims for a switching rate that is $N$ times the line rate (where $N$ is the number of input ports), ensuring the fabric never becomes a bottleneck even under maximum load.

There are three primary generations/types of switching fabrics: Memory, Bus, and Interconnection networks.

***

### 1.7 Switching via memory

```text
+--------------+                                +--------------+
|  Input Port  |          System Bus            | Output Port  |
|  (Ethernet)  | <----------------------------> |  (Ethernet)  |
+--------------+               |                +--------------+
                               v
                         +-----------+
                         |  Memory   |
                         +-----------+
```

This represents the first generation of routers, which were essentially traditional computers.
*   The switching operation was placed under direct control of the CPU.
*   An arriving packet was copied from the input port into the system's central memory, processed by the CPU, and then copied back out to the output port.
*   Because every packet required two bus crossings (input to memory, memory to output), the maximum switching speed was strictly limited by the system's memory bandwidth.

***

### 1.8 Switching via a bus

```text
[ Input Port A ] -----\                      /---> [ Output Port X ]
                       \                    /
[ Input Port B ] -------[   SHARED BUS     ]-----> [ Output Port Y ]
                       /                    \
[ Input Port C ] -----/                      \---> [ Output Port Z ]
```

In this architecture, input ports transfer datagrams directly to the output port memory across a shared system bus, bypassing central memory entirely.
*   **Bus Contention**: Because the bus is a shared medium, only one packet can traverse it at a given time. If multiple packets arrive simultaneously on different input ports, they must queue and wait their turn.
*   The overall switching speed is thus strictly limited by the total bus bandwidth. This architecture is generally sufficient for smaller access routers but fails at enterprise scale.

***

### 1.9 Switching via interconnection network

```text
Crossbar Switch (3x3)                 Multistage Switch (Clos Network)
  In 1 --+----+----+--> Out A             [ ]--[ ]--[ ]
         |    |    |                      [ ]--[ ]--[ ]
  In 2 --+----+----+--> Out B             [ ]--[ ]--[ ]
         |    |    |
  In 3 --+----+----+--> Out C
```

To overcome bus bandwidth limitations, modern high-speed routers use interconnection networks (like crossbars or Clos networks), originally developed to connect multiple processors in supercomputers.
*   **Multistage Switch**: A large $N \times N$ switch is constructed by linking together multiple stages of smaller switches, allowing multiple packets to be forwarded simultaneously as long as they do not target the exact same output port.
*   **Exploiting Parallelism**: Arriving variable-length datagrams are often fragmented into fixed-length cells. These cells are switched in parallel across the fabric and reassembled back into the original datagram at the output port.
*   High-end routers (e.g., Cisco CRS) scale further by employing multiple switching "planes" working in parallel, achieving switching capacities in the hundreds of Terabits per second (Tbps).

***

### 1.10 Input port queuing

```text
+-------------------+      +--------+      +-------------------+
| In Queue 1        |      |        |      | Out Queue A       |
| [Red] -> [Green]  |----->| Switch |----->| [Red Packet]      |
|                   |      | Fabric |      |                   |
| In Queue 2        |      |        |      | Out Queue B       |
| [Red] -> [Blue]   |--X-->|        |----->| (Empty)           |
+-------------------+      +--------+      +-------------------+
  ^ "Blue" is trapped behind "Red"
```

Input port queuing happens when the switch fabric operates slower than the combined arrival rates of the input ports.
*   This mismatch results in queueing delays and potential packet loss if the input buffer overflows.
*   **Head-of-the-Line (HOL) blocking**: A critical issue in input queues. If the packet at the very front of an input queue (the "head") is blocked from crossing the switch fabric because its destination output port is busy, all the packets behind it in the queue are also blocked—even if their target output ports are completely free.

***

### 1.11 Output port queuing

```text
From Switch Fabric                        +------------------+
(Rate: N * R)                             | Line Termination |
======> [|||||||||||||] ===> [ Protocol ]===> [ Link Output ] ===> (Rate: R)
       Datagram Buffer        (Send)
       (Queueing Area)
```

Output port queuing occurs when the switching fabric delivers packets to an output port faster than the outgoing link can transmit them onto the physical wire.
*   **Buffering**: Mandatory for absorbing these traffic bursts.
*   **Drop Policy**: If packets arrive when the output buffer is completely full, the router must execute a drop policy (e.g., dropping the newly arriving packet, or dropping an older packet to make room). This is a primary cause of network packet loss.
*   **Scheduling Discipline**: The algorithm used to decide which packet in the buffer should be transmitted next. Simple routers use First-Come-First-Served, while advanced routers use priority-based scheduling to ensure network neutrality or Quality of Service (QoS).

***

### 1.12 How much buffering?

Determining the correct buffer size is a classic networking problem. 
*   **RFC 3439 Rule of Thumb**: Historically, buffering was calculated as the average Round Trip Time (RTT), typically estimated at 250ms, multiplied by the link capacity $C$. (e.g., a 10 Gbps link requires a 2.5 Gbit buffer).
*   **Modern Recommendation**: In high-speed core networks handling thousands of simultaneous TCP flows ($N$), the required buffering can be much smaller due to statistical multiplexing. The updated formula is: $\frac{RTT \cdot C}{\sqrt{N}}$.
*   **Bufferbloat Issue**: While buffers prevent packet loss, *too much* buffering is detrimental. Large buffers can absorb massive queues, leading to persistently long RTTs. This causes sluggish TCP response times and severe performance degradation for real-time applications (e.g., VoIP, gaming), particularly in home access routers.

***

### 1.13 Buffer Management

```text
                      [ Packet Arrivals ]
                              |
                              v
                  +-----------------------+
                  |  Queue (Waiting Area) |
                  |  [P3] [P2] [P1]       |
                  +-----------------------+
                              |
                              v
                      [ Link Server ] ---> [ Packet Departures ]
```

Active buffer management is required to maintain network health during times of congestion.
*   **Drop Management**: Determines which packet to discard when the queue is saturated.
    *   **Tail Drop**: The simplest method; arriving packets are simply dropped if there is no room.
    *   **Priority Drop**: Packets are removed or dropped based on a pre-assigned priority level.
*   **Marking**: Instead of dropping packets immediately, routers can mark specific packet headers to signal impending congestion to end-hosts (e.g., Explicit Congestion Notification - ECN, or Random Early Detection - RED).

***

### 1.14 Packet Scheduling: FCFS

```text
Arrivals: P1, P2, P3  --> [ Queue: [P3] [P2] [P1] ] --> Server sends P1, then P2, then P3.
```

**Packet Scheduling** is the mechanism of deciding the exact sequence in which queued packets are sent over the outgoing link.
*   **FCFS (First Come First Served)**: Also known as FIFO (First-In-First-Out). This is the most basic scheduling algorithm. Packets are transmitted strictly in the order they arrived at the output port. 

***

### 1.15 Scheduling policies: round robin

```text
Classified Arrivals     Queues by Class          Server (Cyclic Scanner)
---> [ Class 1 ] -----> [||||||] --\               _
---> [ Class 2 ] -----> [||||||] ----> [ Router ] (_) --> Link Departures
---> [ Class 3 ] -----> [||||||] --/
```

**Round Robin (RR) Scheduling** ensures a more equitable distribution of link bandwidth among different types of traffic.
*   Arriving traffic is inspected and classified into distinct queues based on designated criteria (using any IP header fields, such as port numbers or source IPs).
*   The router's scheduler then cyclically and repeatedly scans these class queues. It transmits exactly one complete packet from Class 1, then one from Class 2, then one from Class 3, and repeats. If a class queue is empty, the server immediately moves to the next available class without waiting (work-conserving).

***

### 1.16 Sidebar: Network Neutrality

Network Neutrality touches upon the policies governing how Internet Service Providers (ISPs) manage the data passing through their networks.
*   **Technical**: Focuses on *how* an ISP allocates its router resources, primarily governed through the mechanisms of packet scheduling and buffer management.
*   **Social & Economic Principles**: Centers on protecting free speech and encouraging fair market innovation and competition without ISP interference.
*   **Legal**: Involves government-enforced rules, which vary wildly by country.
*   For context, the 2015 US FCC Order established "clear, bright line" rules for an open internet:
    *   **No blocking**: Cannot block lawful content.
    *   **No throttling**: Cannot impair or degrade lawful internet traffic.
    *   **No paid prioritization**: Cannot favor some traffic over others in exchange for payment (no "fast lanes").

***

### 1.17 Architectural Principles of the Internet

A core exercise in network design is determining how to partition functions between the network layer (IP) and the transport layer (TCP).
*   **RFC 1958**: Outlines the philosophical foundation of the Internet. The primary goal is simple *connectivity*, the tool to achieve this is the *Internet Protocol*, and the intelligence is placed *end-to-end* rather than hidden inside the network.
*   Three cornerstone beliefs dictate this architecture:
    1.  Simple connectivity at the core.
    2.  The IP protocol acts as a narrow waist.
    3.  Intelligence and complexity reside at the network edge.

***

### 1.18 The IP hourglass: thin waist

```text
      HTTP   SMTP   RTP        <-- Many Application Protocols
        TCP     UDP            <-- Multiple Transport Protocols
     \---------------+         
      \      IP     /          <-- The "Thin Waist" (Only ONE Network Protocol)
       +-----------+           
      / Ethernet PPP\          <-- Many Link Protocols
     / Copper  Fiber \         <-- Many Physical Mediums
```

The Internet architecture is visually modeled as an hourglass.
*   **"Thin waist"**: The center of the hourglass consists of exactly *one* network-layer protocol: the **`IP`** protocol. 
*   This design ensures universal interoperability. Regardless of what application you run (top of hourglass) or what physical wire you use (bottom of hourglass), billions of devices can communicate because they all implement IP in the middle.

***

### 1.19 The end-end argument

```text
Hop-by-hop Implementation:
[Host A] <--> [Router] <--> [Router] <--> [Host B]
 (Requires heavy complexity in every middle router to track data)

End-End Implementation:
[Host A] <------------------------------> [Host B]
 (Routers are dumb; Hosts ensure reliability themselves)
```

The **end-end argument** dictates that specific application-level functions (like reliable data transfer and congestion control) should be implemented at the end hosts rather than inside the network routers.
*   Doing so satisfies "both correctness and minimal design." 
*   It argues strongly *against* implementing low-level complex functions in the network core, as intermediate routers do not have the context of the applications, and burdened routers would become bottlenecks.

***

### 1.20 Where’s the intelligence?

```text
[ 20th Century Phone Net ] -> [ Pre-2005 Internet ] -> [ Post-2005 Internet ]
Intelligence at the Core      Intelligence at the Edge   Massive Edge Infrastructure
(Smart Switches, Dumb       (Dumb Routers, Smart PCs)  (Programmable Networks, Cloud)
 Handsets)
```

Network evolution shows a distinct migration of computing power:
*   **20th Century Phone Net**: The intelligence (routing, billing, switching) was localized entirely inside network switches. Telephones were "dumb" devices.
*   **Internet (pre-2005)**: The paradigm shifted. The core network became "dumb" (simple IP forwarding), while computing power moved to the edges (PCs and servers).
*   **Internet (post-2005)**: The edge has exploded with massive application-level infrastructure (datacenters, CDNs), while the network itself is becoming more programmable (SDN) to support this edge dominance.

---

## 2.0 Chapter 5: Network Layer: Control Plane

### 2.1 Roadmap & Functions Overview
The control plane is responsible for the overall navigation of the network. This chapter outlines the algorithms and protocols that make routing possible.
*   **Forwarding** (Data Plane): The localized action of moving a packet from a router’s input link to the appropriate output link.
*   **Routing** (Control Plane): The network-wide process that determines the end-to-end route a packet takes from source to destination.
*   **Internet Approach**: The Internet fundamentally relies on distributed, per-router control rather than a single master server dictating paths.

***

### 2.2 Per-router control plane

```text
[ Router A Control ] <--------> [ Router B Control ]
        |                               |
        v                               v
[ Forwarding Table ]            [ Forwarding Table ]
(Data Plane)                    (Data Plane)
```

In a distributed per-router control plane, individual routing algorithm components reside in *each and every router*. 
*   These algorithms communicate with each other across the control plane to share network state information. 
*   Using this shared information, each local algorithm independently computes its own localized forwarding table, dictating how its corresponding data plane should forward packets.

***

### 2.3 Internet Routing Breakdown
Internet routing requires three distinct components working in harmony:
1.  **Routing Algorithms**: The mathematical logic (Link State vs Distance Vector) used to compute the paths.
2.  **Routing Protocols**: The hierarchical rule sets dictating how routers share this algorithmic information (e.g., intra-domain **`OSPF`**, inter-domain **`BGP`**).
3.  **Signaling Protocols**: Supporting management and error reporting protocols (e.g., **`ICMP`**).

***

### 2.4 #1 Routing algorithms

```text
[ Source Host ] ---> [ ISP Router ] ---> [ Backbone Router ] ---> [ Dest Host ]
 \_________________________________ PATH ____________________________________/
```

The fundamental goal of a routing algorithm is to determine "good" paths from a sending host to a receiving host.
*   **Path**: The contiguous sequence of routers that packets traverse from the initial source to the final destination.
*   **"Good"**: This is subjective based on the network operator's metrics. It usually means the path with the least "cost", which translates to the fastest, shortest, or least congested route.

***

### 2.5 Graph abstraction for routing

```text
      (5)
  [u] ---- [v]
   | \      | \ (3)
(2)|  \(1)  |  [w]
   |   \    | / (5)
  [x] ---- [y]
      (1)
```

Network topologies are universally represented using graph theory abstraction: $G = (N,E)$
*   $N$: The set of nodes (representing the network routers).
*   $E$: The set of edges (representing the physical links between routers).
*   $c_{a,b}$: The **cost** of the direct link connecting node $a$ and node $b$. If no direct link exists, the cost is typically considered infinity ($\infty$). Costs are assigned by network operators and can represent physical distance, bandwidth, or current congestion levels.

***

### 2.6 Routing algorithm classification

```text
                      Static (Slow changes)
                                |
                                |
 Decentralized <----------------+----------------> Global 
 (Distance Vector)              |                  (Link State)
                                |
                      Dynamic (Fast changes)
```

Routing algorithms are classified along two primary spectrums:
1.  **Global vs. Decentralized**:
    *   **Global**: Every single router is provided with the complete topology and all link cost information of the entire network. These are known as **"link state"** algorithms.
    *   **Decentralized**: Routers only know the link costs to their immediate, physically attached neighbors. They compute paths via an iterative process of exchanging information with these neighbors. These are known as **"distance vector"** algorithms.
2.  **Static vs. Dynamic**:
    *   **Static**: Routes are provisioned manually and change very slowly over time.
    *   **Dynamic**: Routes adapt quickly to network changes, recalculating periodically or instantly in response to link cost changes or node failures.

***

### 2.7 Dijkstra’s algorithm for link-state routing

```text
Initialization:
  Set N' = {u} (Source Node)
  For all nodes v:
    If v is adjacent to u: D(v) = c(u,v)
    Else: D(v) = infinity

Loop:
  Find node w not in N' with minimum D(w)
  Add w to N'
  Update D(v) for all neighbors of w:
    D(v) = min( D(v), D(w) + c(w,v) )
Until all nodes are in N'
```

Dijkstra’s algorithm is the classic **Link-State (LS)** routing algorithm.
*   **Centralized Knowledge**: The network topology and all link costs must be known to *all* nodes. This is achieved by having every node **broadcast** its local link states to the entire network.
*   **Process**: The source node runs the algorithm to compute the absolute least-cost paths to all other nodes, generating its complete **forwarding table**.
*   **Iterative Execution**: It is iterative; after $k$ iterations, the shortest paths to exactly $k$ destinations are definitively known.
*   **Complexity**: 
    *   *Algorithm Complexity*: $O(n^2)$ for $n$ nodes, though optimized implementations utilizing min-heaps can achieve $O(n \log n)$.
    *   *Message Complexity*: Requires $O(n^2)$ messages because every router must broadcast state information to every other router.
*   **Route Oscillations**: A notable drawback. If link costs are dependent on traffic volume, the algorithm can cause routing loops where traffic rapidly flips back and forth between two paths as the costs dynamically adjust to the load.

***

### 2.8 Bellman-Ford algorithm for distance vector routing

```text
Node x computing path to y via neighbor v:
D_x(y) = min [ c(x,v) + D_v(y) ]
```

The **Distance Vector (DV)** algorithm is grounded in dynamic programming and specifically the **Bellman-Ford equation**.
*   **Bellman-Ford Equation**: $D_x(y) = \min_v \{ c_{x,v} + D_v(y) \}$
*   This means the shortest distance from node $x$ to node $y$ is the cost of the direct link to a neighbor $v$, plus the neighbor $v$'s shortest known distance to $y$. The algorithm calculates this for all neighbors and selects the path yielding the absolute minimum cost.

***

### 2.9 Distance vector algorithm

```text
       Wait for Change
     (Local link cost or
      update from neighbor)
              |
              v
     Recompute DV Estimates
    (Using Bellman-Ford Eq)
              |
              v
     Did my DV change?
      /             \
    YES             NO
     |               |
  Notify           Do Nothing
 Neighbors
```

Unlike Link-State, Distance Vector does not require full network topology.
*   Each node periodically sends its own distance vector estimate to its direct neighbors.
*   **Iterative, Asynchronous**: The algorithm wakes up and iterates locally only when triggered by an event: either a local link cost changes, or it receives a DV update message from a neighbor.
*   **Distributed, Self-Stopping**: It is entirely distributed. A node only sends an update to its neighbors if its own internal distance vector calculations resulted in a change. If the network is stable, message passing ceases completely.

***

### 2.10 Distance vector: link cost changes

```text
Cost Drops (Good News):
[Node Y] detects cost to [Node X] dropped from 50 to 1.
Y immediately updates and tells Z. Network converges rapidly.

Cost Spikes (Bad News):
[Node Y] detects cost to [Node X] spiked from 4 to 60.
Y thinks, "Z previously told me it can reach X in 5. I'll route through Z!"
(Y doesn't realize Z's path actually goes backwards through Y)
```

Distance Vector handles network changes asynchronously, leading to specific behavioral traits:
*   **"Good news travels fast"**: When a link cost decreases, the lower cost rapidly diffuses through the network as nodes update and forward their new, cheaper vectors.
*   **"Bad news travels slow"**: When a link cost severely spikes or fails, the algorithm can fall into a trap known as the **count-to-infinity problem**. Two neighboring routers may end up locked in a routing loop, incrementally bouncing the path cost back and forth to infinity because they are relying on outdated paths through each other.

***

### 2.11 Comparison of LS and DV algorithms

Both algorithms are foundational but excel in different environments:
*   **Message Complexity**:
    *   **LS**: High. Requires $O(n^2)$ messages because link states must be broadcast network-wide.
    *   **DV**: Low. Messages are only exchanged between direct neighbors.
*   **Speed of Convergence**:
    *   **LS**: Generally fast, though processing $O(n^2)$ algorithms takes CPU cycles. May suffer from oscillations.
    *   **DV**: Slower convergence time. Susceptible to routing loops and the count-to-infinity problem.
*   **Robustness** (What happens if a router is hacked or fails?):
    *   **LS**: Robust. A compromised router can only advertise incorrect costs for its *own direct links*. Every other router independently computes its own table.
    *   **DV**: Fragile. A compromised router can advertise an incorrect cost for an entire *path* (e.g., falsely claiming a 0-cost path to all destinations, known as black-holing). Because DV routers trust their neighbors' calculations, this error cascades entirely through the network.

***

### 2.12 Making Internet routing scalable
The idealized "flat" routing model—where every router computes paths to every other router—fails in the real-world Internet due to two major factors:
*   **Scale**: There are billions of destinations. Storing all of them would require impossibly large routing tables, and the constant exchange of routing updates would instantly swamp all link bandwidth.
*   **Administrative Autonomy**: The Internet is quite literally a "network of networks." Network administrators (ISPs, universities, corporations) demand sovereign control over how routing occurs within their own infrastructure.

***

### 2.13 Autonomous Systems (AS)

```text
        +---------------+             +---------------+
        |     AS 1      |             |     AS 2      |
        | [R]--[R]--[GW]<------------>[GW]--[R]--[R]  |
        |  |         |  |             |  |            |
        +---------------+             +---------------+
```

To solve the scalability and autonomy problems, routers are aggregated into large regional blocks known as **Autonomous Systems (AS)** or domains.
*   **Intra-AS (Intra-domain) routing**: Routing that occurs strictly within the boundaries of a single AS. All routers within this AS run the exact same routing protocol.
*   **Inter-AS (Inter-domain) routing**: The macro-level routing that occurs *between* different ASes.
*   **Gateway Router**: A specialized router sitting at the literal "edge" of its own AS. It possesses physical links connecting it to gateway routers residing in a different AS.
*   **AS Number (ASN)**: A globally unique 4-byte identifier assigned to each Autonomous System to allow global inter-AS routing policies to function.

***

### 2.14 Intra-AS routing protocols
Within an AS, administrators can choose from several well-established protocols:
*   **`RIP` (Routing Information Protocol)**: A classic Distance Vector protocol. It is legacy and no longer widely used due to slow convergence.
*   **`EIGRP` (Enhanced Interior Gateway Routing Protocol)**: An advanced Distance Vector protocol. It was historically a Cisco-proprietary standard but was made open in 2013.
*   **`OSPF` (Open Shortest Path First)**: Currently the dominant standard. It is a Link-State routing protocol that relies heavily on Dijkstra's algorithm (the `IS-IS` protocol functions almost identically).

***

### Key Terms to Define:

*   **Control plane / Data plane**: The data plane is the hardware-level switching of packets from input to output ports within a router. The control plane is the software-level logic that calculates the network paths and populates the forwarding tables used by the data plane.
*   **Switching fabric**: The internal, high-speed physical network inside a router that moves datagrams from input ports to output ports.
*   **Match plus action**: The paradigm used by input ports where a packet's header fields are evaluated (match) to determine the outgoing port (action) locally.
*   **Longest prefix match**: The rule routers use to resolve overlapping IP range entries; the forwarding table entry with the most specific (longest) matching bit prefix dictates the output port.
*   **Head-of-the-Line (HOL) blocking**: A queuing bottleneck where a packet at the front of an input queue is blocked from moving, thus trapping all subsequent packets behind it, even if their destination ports are available.
*   **Link-state (LS) algorithm / Dijkstra's**: A routing methodology where every router possesses a complete map of the network topology and link costs, independently computing the shortest paths.
*   **Distance vector (DV) algorithm / Bellman-Ford**: A decentralized routing methodology where routers only know paths based on cost estimates provided by their immediate physical neighbors.
*   **Count-to-infinity problem**: A failure condition in Distance Vector algorithms where a broken link causes nodes to endlessly increment their cost estimates in a localized routing loop.
*   **Autonomous System (AS) / AS Number (ASN)**: A self-contained network or group of networks under a single administrative domain, identified globally by a unique ASN.
*   **Gateway router**: A boundary router sitting at the edge of an Autonomous System that connects to and shares routing information with other Autonomous Systems.
