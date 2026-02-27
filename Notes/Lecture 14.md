## 1.0 Example on Internet Routing

### 1.1 Example on Internet Routing

To fully understand how data navigates the complex, interconnected web of the Internet, we look at how various routing protocols work together. A complete Internet routing scenario relies on a combination of inter-domain and intra-domain protocols:
*   **eBGP** (External Border Gateway Protocol) advertises the path vector to other Autonomous Systems (ASes). This protocol essentially says, "Here is a route to a destination outside of our network."
*   **iBGP** (Internal Border Gateway Protocol) takes that path vector information learned from the edge of the network and propagates it among the internal routers *within* a specific AS. This ensures all internal routers know how to reach external destinations.
*   **OSPF** (Open Shortest Path First) is an intra-domain routing protocol used to compute the mathematically shortest path between two specific routers within the same AS.
*   **Hot Potato Routing** is a routing mechanic and policy decision where a router tries to hand off traffic to an adjacent AS as quickly and cheaply as possible, minimizing internal transit costs.

***

### 1.2 Example on Internet routing: eBGP

When routing traffic across the Internet, a gateway router at the edge of an AS may learn about *multiple* paths to the same destination subnet from different external peers. 
The gateway router cannot keep all paths; it must choose the best path based on administrative *policy* (e.g., cost, security, business agreements). Once a path is chosen via **eBGP**, the router advertises this selected path to all other routers within its own AS using **iBGP**.

* Visual Aid: Network diagram of routing between AS 1, AS 2, and AS 3 showing path vector propagation *
```text
      (AS3, X)               (AS3, X)
    <-----------        <----------------
   [1b]        [3a]-----[3b]
  /    \      /    \      |
[1a]---[1c]*<-------[2b]---[3d]----[ X ] (Destination)
  \    /  ^   (AS2,AS3,X)\ /
   [1d]   |               [2c]
          |              /
         [2a]-----------[2d]
        AS 1            AS 2            AS 3

* Gateway router 1c (in AS 1) learns path (AS2, AS3, X) from router 2a.
* Gateway router 1c also learns path (AS3, X) directly from router 3a.
* Based on policy, 1c chooses (AS3, X) and advertises it within AS 1 via iBGP.
```

***

### 1.3 Internet routing example: iBGP & OSPF

Once the gateway router has selected the best external path, **iBGP** ensures that all internal routers (like `1a`, `1b`, and `1d` in the previous example) learn that the "path to destination `X` goes through gateway `1c`." 

However, iBGP only provides the logical destination (the gateway). To actually figure out how to physically forward the packet to `1c`, the internal routers rely on **OSPF**. OSPF computes the shortest intra-domain path and maps it to specific local physical interfaces.

* Visual Aid: Diagram of AS networks illustrating local link interfaces and routing tables at specific routers (1a, 1d) *
```text
Inside AS 1:
               [1b]
              /    \
  (Link 1)  /        \
[1a]------/           \[1c] ---> To AS 3 ---> [ X ]
  \                  /
  (Link 2)         / (Link 1 on 1d)
    \            /
      \        /
        [1d]

Routing Table at Router 1d:
| Destination | Interface |
|-------------|-----------|
| ...         | ...       |
| 1c          | 1         | (Learned via OSPF)
| X           | 1         | (Learned via iBGP mapped to OSPF interface)

* Router 1d knows X is reached via 1c (iBGP).
* Router 1d knows 1c is reached via interface 1 (OSPF).
* Therefore, to reach X, 1d forwards out interface 1.
```

***

### 1.4 Example on Internet routing: Hot potato routing

**Hot Potato Routing** is a routing practice where a network tries to get rid of a packet (the "hot potato") as quickly as possible. When a router learns via iBGP that there are multiple valid border routers to exit the AS, it will simply choose the local gateway that has the least *intra-domain* cost (usually determined by OSPF link weights). 

It does this *regardless* of what the remaining external path looks like, focusing solely on minimizing internal bandwidth and resource usage. Ranked route selection is still used, but hot potato principles act as a crucial tie-breaker or priority mechanism.

* Visual Aid: Diagram showing hot potato routing path choice based on OSPF link weights between ASes *
```text
            OSPF Link Weights in AS 2
            
                  [2b]
                 /    \
            112 /      \
               /        \
   To AS 1 <--[2a]       [2c]--> To AS 4 (Path to X)
               \        /
            201 \      / 263
                 \    /
                  [2d]
                   |
             Incoming packet
             destined for X

* Router 2d needs to send a packet to destination X.
* It learns it can exit via border router 2a or 2c.
* Hot Potato Routing dictates choosing the path with the lowest OSPF cost.
* Cost to 2a = 201. Cost to 2c = 263.
* 2d chooses to forward to 2a, dumping the packet out of its AS as cheaply as possible.
```


## 2.0 Internet Control Message Protocol (ICMP)

### 2.1 ICMP: internet control message protocol

**ICMP (Internet Control Message Protocol)** is a critical signaling protocol used by hosts and routers to communicate network-level information. Because IP (Internet Protocol) does not guarantee delivery and lacks built-in error reporting mechanisms, ICMP fills this gap. 

It is primarily used for:
*   **Error reporting:** Notifying the sender if a host, network, port, or protocol is unreachable.
*   **Diagnostics:** Handling echo requests and replies (this is the mechanism behind the `ping` utility).

ICMP is considered a network-layer protocol, but it operates "above" IP; ICMP messages are encapsulated and carried directly inside IP datagrams. An ICMP message consists of a `Type` field, a `Code` field, and the first 8 bytes of the IP datagram that caused the error (so the sender can identify which packet failed).

* Visual Aid: Table listing ICMP Type, Code, and descriptions (e.g., echo reply, dest. network unreachable) *
```text
| Type | Code | Description                            |
|------|------|----------------------------------------|
| 0    | 0    | Echo reply (ping response)             |
| 3    | 0    | Destination network unreachable        |
| 3    | 1    | Destination host unreachable           |
| 3    | 2    | Destination protocol unreachable       |
| 3    | 3    | Destination port unreachable           |
| 8    | 0    | Echo request (ping request)            |
| 11   | 0    | TTL expired (used by traceroute)       |
```

***

### 2.2 Traceroute and ICMP

The `traceroute` diagnostic tool relies heavily on ICMP to map the path packets take to a destination. 

1.  The source sends sets of UDP segments to an unlikely port on the destination.
2.  It sets the IP `TTL` (Time to Live) field to 1 for the first set, 2 for the second, and so on.
3.  When a datagram reaches the *nth* router, its `TTL` expires (drops to 0). The router discards the datagram and sends an ICMP "TTL expired" message (`Type 11, Code 0`) back to the source. The source records the router's IP and the Round Trip Time (RTT).
4.  **Stopping Criteria:** The process stops when the UDP segment finally reaches the destination host. Because the destination port is invalid, the host returns an ICMP "port unreachable" message (`Type 3, Code 3`), signaling the source that the trace is complete.

* Visual Aid: Diagram of the traceroute process showing 3 probes being sent across multiple routers to a destination *
```text
[Source] -----> (R1) -----> (R2) -----> (R3) -----> [Destination]
   |             |            |           |              |
   |--TTL=1----->|            |           |              |
   |<--ICMP 11,0-|            |           |              |
   |                          |           |              |
   |--------TTL=2------------>|           |              |
   |<---------ICMP 11,0-------|           |              |
   |                                      |              |
   |-------------------TTL=n---------------------------->|
   |<-------------------ICMP 3,3 (Port Unreachable)------|
```

***

### 2.3 Network layer: Summary

To summarize our exploration of the network layer:
*   The network control plane utilizes per-router control.
*   We explored traditional routing algorithms (link-state and distance-vector).
*   We mapped these algorithms to actual Internet routing protocols: OSPF for intra-domain routing and BGP for inter-domain routing.
*   We examined how **ICMP** provides essential signaling, diagnostics, and error-reporting capabilities.


## 3.0 The Link Layer and LANs

### 3.1 Layering in Internet protocol stack

While the Network layer provides best-effort *global* packet delivery between logical IP addresses, the **Link Layer** is responsible for best-effort *local* packet delivery. It focuses on physically moving data across a single, individual link from one node to the next adjacent node.

* Visual Aid: Diagram of the Internet protocol stack highlighting the Link layer *
```text
+-------------------+
|    Application    |  ... built on ...
+-------------------+
|     Transport     |  Reliable (or unreliable) transport
+-------------------+
|      Network      |  Best-effort global packet delivery
+===================+
|     LINK LAYER    | <--- Best-effort LOCAL packet delivery
+===================+
|     Physical      |  Physical transfer of bits
+-------------------+
```

***

### 3.2 Roadmap and Highlights (Chapter 6)

Moving into the Link Layer, our roadmap covers several core topics:
*   Link-layer basics and terminology.
*   Error detection and correction techniques.
*   Multiple access protocols (how nodes share a broadcast medium).
*   Local Area Networks (LANs), including MAC addressing, ARP, Ethernet, switches, and VLANs.
*   Link virtualization techniques like MPLS.
*   Data center networking architectures.


## 4.0 Link Layer Overview

### 4.1 Terminology and Responsibilities

To discuss the link layer accurately, we must establish specific terminology:
*   **Nodes:** These are the devices on the network, such as hosts (computers, phones) and routers.
*   **Links:** These are the physical or logical communication channels connecting adjacent nodes along a path (e.g., wired Ethernet, wireless Wi-Fi, LANs).
*   **Frame (Layer-2 Packet):** This is the data unit of the link layer. It encapsulates the network-layer IP datagram.

The primary responsibility of the **Link Layer** is transferring a datagram from one node to a *physically adjacent* node over a single link.

* Visual Aid: Network diagram illustrating nodes and links across mobile, enterprise, ISP, and datacenter networks *
```text
   [Mobile Host]               [Enterprise Host]
         \ (Wireless Link)            | (Wired LAN Link)
          \                           |
        [Base Station]             [Switch]
                \                     /
                 \                   /
                [   Global ISP Router  ]
                          | (Fiber Optic Link)
                          |
                 [ Datacenter Router ]
```

***

### 4.2 Link layer: context

An IP datagram does not travel over a single uniform network. It is transferred across multiple different access networks over various types of links. Consequently, different link protocols are used on different segments of a journey (e.g., Wi-Fi on the first link from a laptop to a router, Ethernet on the next link, and fiber-optic protocols over the WAN). Each different link protocol provides different services and guarantees.

***

### 4.3 Link layer services

Depending on the specific protocol used, the link layer can provide a variety of services:
*   **Framing and link access:** Encapsulating the datagram into a **frame** by adding a header and trailer. It also utilizes **MAC (Media Access Control)** addresses to identify the local source and destination.
*   **Improved delivery quality between adjacent nodes:** Reliable data transfer might be implemented at the link layer. This is rarely used on low bit-error links (like fiber), but is common on highly error-prone links (like wireless Wi-Fi) to correct local errors without forcing a slow, end-to-end TCP retransmission.
*   **Flow control:** Pacing the transmission rate between adjacent sending and receiving nodes so the receiver's buffer isn't overwhelmed.
*   **Error detection:** Identifying errors caused by signal attenuation or physical noise.
*   **Error correction:** Not only detecting but dynamically identifying and *correcting* bit errors without needing a retransmission.
*   **Half-duplex and full-duplex:** Dictating transmission flow. In half-duplex, nodes at both ends can transmit, but not simultaneously. In full-duplex, simultaneous bidirectional transmission is allowed.

* Visual Aid: Diagram illustrating framing and link access between adjacent physical nodes *
```text
  [ Sending Node ]                            [ Receiving Node ]
  | IP Datagram  |                            | IP Datagram  |
  +--------------+       Physical Link        +--------------+
  | Frame Header |  <======================>  | Frame Header |
  |   Datagram   |      (Bits on a wire/      |   Datagram   |
  | Frame Trailer|       radio waves)         | Frame Trailer|
  +--------------+                            +--------------+
```

***

### 4.4 Where is the link layer implemented?

Unlike higher layers (like Application and Transport) which are implemented mostly in software by the OS, the link layer is physically implemented in a **Network Interface Card (NIC)** or directly on a network chip (like an Ethernet or Wi-Fi chip). 

It is an integral combination of hardware, firmware, and low-level driver software, connecting directly to the host's system buses.

* Visual Aid: Architecture diagram showing a NIC connecting to a host bus, CPU, and memory *
```text
+------------------------------------------+
|                 Host Computer            |
|  +-------+    +--------+                 |
|  |  CPU  |    | Memory |                 |
|  +---+---+    +----+---+                 |
|      |             |                     |
|  ============= System Bus ============== |
|              |                           |
|      +-------+-------+                   |
|      |  Controller   |  <--- NIC         |
|      | +-----------+ |       Hardware    |
|      | | Link Layer| |                   |
|      | +-----------+ |                   |
|      | | Physical  | |                   |
|      +-------+-------+                   |
+--------------|---------------------------+
               v
         Physical Link
```

***

### 4.5 Interfaces communicating

The communication between interfaces happens seamlessly:
*   **Sending side:** The controller takes the datagram from memory, encapsulates it into a frame, and adds error-checking bits, reliable data transfer headers, and flow control signals.
*   **Receiving side:** The receiving controller examines the frame for errors, handles local flow control, extracts the datagram, and passes it up to the Network layer via the system bus.

* Visual Aid: Diagram showing datagram flow from sending CPU/memory down to physical link and up to receiving CPU/memory *
```text
     [Sender Host]                             [Receiver Host]
       Datagram                                   Datagram
          |                                          ^
          v                                          |
    +-----------+                              +-----------+
    | NIC Card  |                              | NIC Card  |
    | (Framing &| ====[ Frame on Physical Link]=>| (Error    |
    |  EDC)     |                              | Checking) |
    +-----------+                              +-----------+
```


## 5.0 Error Detection and Correction

### 5.1 Error detection: Overall flow

As frames travel over physical media, electrical noise or interference can flip bits. To counter this, the sender calculates **EDC (Error Detection and Correction bits)** based on the data (`D`) being sent.
*   `D` represents the data protected by error checking (including network headers).
*   The receiver calculates its own EDC based on the received data `D'`. If they don't match, an error has occurred.
*   *Note: Error detection is never 100% reliable!* A sufficiently corrupted packet might accidentally generate a valid EDC. Larger EDC fields yield better detection.

* Visual Aid: Flowchart showing sender adding EDC to D, transmission over a bit-error prone link, and receiver validating D' *
```text
       SENDER SIDE                           RECEIVER SIDE
       
      [ Datagram ]                           [ Datagram ]
           |                                       ^
           v                                       | (If OK)
+--------------------+                   +--------------------+
|  D (Data)  |  EDC  |                   | D' (Data) |  EDC'  |
+--------------------+                   +--------------------+
           |                                       ^
           |                                       |
           +-----> [ Bit-error prone link ] -------+
```

***

### 5.2 3 schemes of error detection

Link layer protocols typically employ one of three primary error detection schemes, varying in complexity and overhead:
1.  **Parity check**
2.  **Internet checksum**
3.  **Cyclic Redundancy Check (CRC)**

***

### 5.3 Parity checking

**Parity Checking** is the simplest form of error detection.
*   **Single bit parity:** The sender counts the number of 1s in the data. For *even parity*, an extra bit is added to ensure the total number of 1s is an even number. This can only *detect* single-bit errors.
*   **Two-dimensional bit parity:** The data is arranged in a 2D matrix. Parity bits are calculated for every row and every column. If a single bit flips, exactly one row parity and one column parity will fail. The intersection of these failures allows the receiver to detect *and correct* the exact flipped bit.

* Visual Aid: Diagrams showing a single bit parity string and a 2D matrix used for two-dimensional bit parity *
```text
Single Bit Parity (Even):
Data: 0111000110101011  -> 9 ones. Add a '1' parity bit -> 0111000110101011 [1]

Two-Dimensional Parity:
      d1,1  d1,2  d1,3 | row_parity 1
      d2,1  d2,2  d2,3 | row_parity 2
      d3,1  d3,2  d3,3 | row_parity 3
      -----------------+
col_parity  c_p   c_p  |
    1        2     3   |

* If bit d2,2 flips, row_parity 2 and col_parity 2 will both flag an error, identifying the exact coordinate of the failure.
```

***

### 5.4 Internet checksum (review)

Often used at the Transport layer (TCP/UDP), the **Internet Checksum** treats the data segment as a sequence of 16-bit integers. 
*   **Sender:** Computes the 1's complement sum of these integers and places the result in the checksum field.
*   **Receiver:** Computes the same sum on the received data. If the result does not equal the checksum field, an error is detected.

***

### 5.5 Cyclic Redundancy Check (CRC)

The **CRC (Cyclic Redundancy Check)** is a highly powerful and robust mathematical error-detection coding used universally in Ethernet and 802.11 Wi-Fi link layers.
*   `D` = Data bits (treated as a massive binary number).
*   `G` = A pre-agreed bit pattern (generator) of $r+1$ bits.
*   The goal is for the sender to choose $r$ CRC bits (referred to as `R`), and append them to `D`.
*   The appended result `<D, R>` must be exactly divisible by the generator `G` using modulo-2 arithmetic (XOR).
*   The receiver divides the received `<D, R>` by `G`. If there is a non-zero remainder, an error has occurred.

* Visual Aid: Formula representation of CRC calculation *
```text
<D, R> = D * 2^r XOR R
```

* Visual Aid: Long division mathematical example calculating the CRC remainder R *
```text
Given D = 101110, G = 1001, r = 3 (append 000 to D)
                  
            1 0 1 0 1 1  (Quotient)
          __________________
G= 1 0 0 1 |1 0 1 1 1 0 0 0 0  (D * 2^r)
            1 0 0 1
            -------
              0 1 0 1
                0 0 0
              -------
                1 0 1 0
                1 0 0 1
                -------
                  0 1 1 0
                  0 0 0 0
                  -------
                    1 1 0 0
                    1 0 0 1
                    -------
                      0 1 1 (Remainder = R) -> CRC bits
```
*FYI: CRC is extremely effective and will detect all burst errors of length less than $r+1$ bits.*


## 6.0 Multiple Access Protocols

### 6.1 Where you need multiple access protocols

Network links are categorized into two main types:
1.  **Point-to-point links:** A dedicated wire connecting exactly two nodes (e.g., dial-up PPP, or a direct Ethernet cable between a switch and a host).
2.  **Broadcast (shared wire or medium) links:** A medium where multiple nodes can transmit and receive simultaneously. 

When a medium is shared, you need a method to coordinate traffic.

* Visual Aid: Illustrations of a shared wire, shared radio (WiFi, 4G/5G, satellite), and humans at a cocktail party representing a shared air medium *
```text
Shared Wire:      [PC 1] ---+--- [PC 2] ---+--- [PC 3] (Coaxial Cable)

Shared Wireless:  [Laptop] \
                  [Phone] ---( Wi-Fi Access Point )
                  [Tablet] /

Cocktail Party: Multiple people talking in the same room. If two people talk
at the exact same time, their voices overlap (Interference/Collision).
```

***

### 6.2 Multiple access protocols

In a single shared broadcast channel, two or more simultaneous transmissions will result in **interference**, also known as a **collision**. A collision makes the transmitted data unreadable.

A **Multiple Access Protocol** is a distributed algorithm that dictates how nodes share the channel and determines precisely *when* a node is allowed to transmit. Crucially, there is no separate, out-of-band control channel; nodes must use the shared channel itself to coordinate.

***

### 6.3 An ideal multiple access protocol

If we have a **MAC (Multiple Access Channel)** with a bandwidth capacity of `R` bps (bits per second), an *ideal* protocol would have these features:
1.  When exactly one node wants to transmit, it can use the full rate `R`.
2.  When `M` nodes want to transmit, each node gets an equal, average rate of `R/M`.
3.  It is fully decentralized (no master node controls the network, no synchronized clocks).
4.  It is computationally simple to implement.

***

### 6.4 MAC protocols: taxonomy

There are three broad families of MAC protocols:
1.  **Channel Partitioning:** Dividing the channel into smaller, dedicated "pieces" (by time, frequency, or code).
2.  **Random Access:** Not dividing the channel, allowing collisions to occur, but defining rules to "recover" from them.
3.  **"Taking Turns":** Nodes pass a token or take turns sequentially to guarantee no collisions.

***

### 6.5 Channel partitioning MAC protocols: TDMA & FDMA

Two traditional methods of partitioning a channel are TDMA and FDMA:
*   **TDMA (Time Division Multiple Access):** Time is divided into "rounds," and each node gets a fixed-length slot (equal to a packet transmission time) in every round. If a node doesn't have data to send during its slot, the channel sits idle (wasted bandwidth).
*   **FDMA (Frequency Division Multiple Access):** The physical spectrum is divided into specific frequency bands (like radio stations). Each node gets a dedicated frequency. If a node isn't transmitting, that frequency band remains idle.

* Visual Aid: Timeline diagram showing 6-slot frames for TDMA *
```text
TDMA (Nodes 1, 3, and 4 have data to send. Nodes 2, 5, 6 are idle):

<-------- 6-slot frame --------><-------- 6-slot frame -------->
 [ 1 ] [   ] [ 3 ] [ 4 ] [   ] [   ] | [ 1 ] [   ] [ 3 ] [ 4 ] [   ] [   ]
 Slot1 Slot2 Slot3 Slot4 Slot5 Slot6 | Slot1 Slot2 Slot3 Slot4 Slot5 Slot6
```

* Visual Aid: Diagram showing frequency bands divided over an FDM cable *
```text
FDMA (Frequency Bands over a Cable/Air):
 ^ 
 | [ Node 1 Frequency ] --------> (Transmitting)
F| [ Node 2 Frequency ] --------> (Idle)
r| [ Node 3 Frequency ] --------> (Transmitting)
e| [ Node 4 Frequency ] --------> (Transmitting)
q| [ Node 5 Frequency ] --------> (Idle)
 | [ Node 6 Frequency ] --------> (Idle)
 +------------------------------------------> Time
```

***

### 6.6 Random access protocols

Unlike partitioning, a **Random Access MAC Protocol** allows any node with data to transmit at the full channel data rate `R`. There is no *a priori* coordination. Because multiple nodes might transmit at once, the protocol must feature logic to *detect* collisions and algorithms to *recover* from them (e.g., delayed retransmissions).

Examples of random access protocols include ALOHA, slotted ALOHA, CSMA, CSMA/CD (Ethernet), and CSMA/CA (Wi-Fi).

***

### 6.7 Slotted ALOHA

**Slotted ALOHA** is a foundational random access protocol.
*   **Assumptions:** All frames are the exact same size. Time is divided into equal-sized slots. Nodes are globally synchronized and can only begin transmitting at the absolute start of a slot.
*   **Operation:** When a node has a frame, it transmits in the next available slot.
    *   *If no collision:* The node successfully sent its data.
    *   *If collision:* All transmitting nodes detect the collision. Every node will then independently flip a weighted coin and retransmit in each subsequent slot with probability `p` until the frame succeeds. 
*   **Pros:** A single active node can use the full rate `R`. It is highly decentralized.
*   **Cons:** Collisions waste time. Empty slots waste time. Nodes require strict clock synchronization.

* Visual Aid: Timeline diagram showing nodes transmitting in slots resulting in Collisions (C), Empty (E), or Success (S) *
```text
Node 1: [ 1 ]       [ 1 ]             [ 1 ]       [ 1 ]
Node 2:       [ 2 ]       [ 2 ] [ 2 ]
Node 3:       [ 3 ]                   [ 3 ]             [ 3 ]
--------------------------------------------------------------
Result:   S     C     S     C     S     C     E     S     S

* S = Success (1 node sent)
* C = Collision (>1 node sent)
* E = Empty (0 nodes sent)
```

***

### 6.8 Slotted ALOHA: efficiency

The theoretical *efficiency* of Slotted ALOHA is the long-run fraction of slots where a successful transmission occurs (assuming `N` nodes with constantly full queues).
*   If we define `p` as the probability a node transmits, the probability of a specific node succeeding is `p(1-p)^(N-1)`.
*   The probability *any* node succeeds is `N * p(1-p)^(N-1)`.
*   Calculus shows that finding the optimal transmission probability (`p*`) and taking the limit as the number of nodes `N` approaches infinity yields a maximum efficiency of `1/e`, which is approximately `0.37`.

**Conclusion:** At its absolute best, Slotted ALOHA is only utilizing the shared channel for useful data transmission **37%** of the time. The rest is lost to collisions and idle slots.
