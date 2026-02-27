## 1.0 CSMA and Random Access MAC Protocols

### 1.1 CSMA (carrier sense multiple access)
**CSMA (Carrier Sense Multiple Access)** is a foundational random-access MAC protocol based on a simple, polite rule: "listen before transmit." 

*   **Simple CSMA**:
    *   **Channel idle**: If a node senses that no other node is currently transmitting, it will transmit its entire frame.
    *   **Channel busy**: If the channel is sensed as busy (another transmission is detected), the node will defer its transmission until the channel becomes idle.
    *   *Human analogy*: Do not interrupt others when they are speaking.
*   **CSMA/CD (CSMA with Collision Detection)**:
    *   Even with carrier sensing, collisions can happen. CSMA/CD adds a mechanism to **detect collisions** within a very short time.
    *   Once a collision is detected, the colliding transmissions are immediately aborted. This prevents the nodes from transmitting the rest of their frames, significantly reducing wasted channel time and bandwidth.
    *   This is easy to implement in wired environments (like copper Ethernet) where devices can simultaneously listen and transmit, but very difficult in wireless networks where the transmission signal overwhelms the receiving antenna.
    *   *Human analogy*: The polite conversationalist who stops talking immediately if they accidentally start speaking at the same time as someone else.

***

### 1.2 CSMA: collisions
Despite "listening before transmitting," collisions can still occur. The primary culprit is **propagation delay**—the physical time it takes for an electrical or electromagnetic signal to travel from one node to another. 

*   Because of this delay, a node might sense an idle channel and begin transmitting, completely unaware that a distant node has *just* started its own transmission. The first signal simply hasn't reached the second node yet.
*   **Collision effect**: Without collision detection, the entire packet transmission time for both nodes is completely wasted, resulting in garbled data.
*   The probability of a collision is directly influenced by the physical distance between nodes and the resulting propagation delay.

```text
* Visual Aid: CSMA Collision Due to Propagation Delay *

Node A                        Node B                        Node C
  |                             |                             |
  | t0: Node A senses idle      |                             |
  |     and starts sending.     |                             |
  |========================\    |                             |
  |                         \   |                             |
  |                          \  | t1: Node B senses idle      |
  |                           \ |     (A's signal hasn't      |
  |                            \|     reached B yet) and      |
  |                             X     starts sending.         |
  |                            /|\                            |
  |                           / | \                           |
  |                          /  |  \                          |
  | A's transmission        /   |   \  B's transmission       |
  | continues (wasted)     /    |    \ continues (wasted)     |
  v                             v                             v
time                          time                          time

[!] The entire transmission time is wasted due to overlapping signals.
```

***

### 1.3 CSMA/CD:
CSMA/CD improves upon basic CSMA by actively monitoring the line while transmitting.

*   By detecting the collision early, CSMA/CD **aborts** the transmission, drastically reducing the amount of time the channel is tied up by garbled, useless data. 

```text
* Visual Aid: CSMA/CD Shortened Wasted Time *

Node A                        Node B                        Node C
  |                             |                             |
  | t0: A starts sending        |                             |
  |========================\    |                             |
  |                         \   | t1: B starts sending        |
  |                          \  |                             |
  |                           \ |                             |
  |                            \|                             |
  |                             X  <-- COLLISION occurs here  |
  |                            /|\                            |
  |                           / | \                           |
  |                          /  |  \                          |
  | xxxxxxxxxxxxxxxxxxxxxxxx/   |   \xxxxxxxxxxxxxx           |
  | ^ A detects collision       |   ^ B detects collision     |
  |   and ABORTS.               |     and ABORTS.             |
  v                             v                             v
time                          time                          time

[!] Wasted time is severely limited to the detect/abort duration.
```

***

### 1.4 Ethernet CSMA/CD algorithm
In older, slower 10/100Mbps Ethernet networks running on a half-duplex channel (where devices share the same copper path to send and receive), the CSMA/CD algorithm operates as follows:

1.  The NIC (Network Interface Controller) receives a datagram from the network layer and creates an Ethernet frame.
2.  **Carrier Sense**: The NIC senses the channel.
    *   If **idle**: It starts frame transmission.
    *   If **busy**: It waits until the channel is idle, then transmits.
3.  If the NIC transmits the entire frame without detecting a collision, it is done.
4.  If the NIC detects another transmission while sending, it **aborts** and immediately sends a **jam signal**.
5.  After aborting, the NIC enters the **binary (exponential) backoff algorithm** to calculate a random wait time before attempting step 2 again.

***

### 1.5 On Jam Signal
The **Jam Signal** acts as an "emergency broadcast" to ensure network-wide **collision detection**. 

*   **Purpose**: It guarantees that every device on the wire knows a collision has occurred. 
    *   It solves the **"Partial Collision"** issue common in long bus networks. Without a jam signal, a "far" station might only receive a slightly distorted signal that its hardware doesn't recognize as a collision, assuming its data was delivered successfully. 
    *   It extends the duration of the collision "noise" long enough for even the most distant stations to detect it.
*   **Design**:
    *   **Length**: Typically 32 to 48 bits long.
    *   **Pattern**: Usually a repeating pattern (e.g., `101010...` or `010101...`).
    *   **Intent**: It is not meant to convey data. Its sole purpose is to intentionally violate the expected electrical/logical timing of a normal frame, making it completely uninterpretable as a valid packet by any receiver.

***

### 1.6 Binary Exponential Backoff Algorithm
The **Binary Exponential Backoff (BEB) algorithm** is used to introduce **randomness** and **adaptability** into retry attempts. If colliding stations tried to retransmit immediately, they would infinitely collide.

*   **Algorithm parameters**: Time is divided into discrete slots. In 10 Mbps Ethernet, a **`slot time`** is 51.2 microseconds (the time it takes to transmit 512 bits / 64 bytes).
*   **Steps on collision**:
    1.  **Collision Counter $n$**: The station tracks the number of collisions for the current frame.
    2.  **Calculate the Range**: The station randomly chooses an integer $k$ from the range $[0, 2^n - 1]$.
    3.  **Wait**: The station waits $(k \times \text{slot time})$ before sensing the carrier again.
    4.  **Maximum Limit**: The exponent $n$ caps at 10. Even if collisions continue, the maximum range stays at $[0, 2^{10} - 1]$ (which is 0 to 1023 slots).
    5.  **Give Up**: If a frame fails to transmit after 16 unsuccessful attempts, the NIC gives up and reports an error to the higher IP layer.

***

### 1.7 Example on BEB Algorithm
Imagine two stations, A and B, attempt to transmit and collide for the first time. Here is how their wait ranges expand exponentially:

*   **1st Collision ($n=1$)**: 
    *   Range: $[0, 2^1 - 1] = \{0, 1\}$. 
    *   Both pick randomly. There is a **50%** chance they pick the same number and collide again.
*   **2nd Collision ($n=2$)**: 
    *   Range: $[0, 2^2 - 1] = \{0, 1, 2, 3\}$. 
    *   The "menu" of wait times doubles. The chance of a repeat collision drops to **25%**.
*   **3rd Collision ($n=3$)**: 
    *   Range: $[0, 2^3 - 1] = \{0 \text{ to } 7\}$. 
    *   Collision chance drops to **12.5%**.
*   **4th Collision ($n=4$)**: 
    *   Range: $[0, 2^4 - 1] = \{0 \text{ to } 15\}$. 
    *   Collision chance drops to **6.25%**.

***

### 1.8 CSMA/CD efficiency
The efficiency of CSMA/CD depends heavily on the maximum propagation delay between any two nodes ($t_{prop}$) and the time it takes to transmit a maximum-size frame ($t_{trans}$).

```text
* Visual Aid: CSMA/CD Efficiency Formula *

                  1
efficiency = ------------------
              1 + 5(t_prop / t_trans)
```

*   **Efficiency characteristics**: The efficiency approaches $1$ (100%) as $t_{prop} \rightarrow 0$ (nodes are very close together) or as $t_{trans} \rightarrow \infty$ (frames are very large, making propagation delay negligible in comparison).
*   **Conclusion**: CSMA/CD offers significantly better performance than ALOHA protocols, while remaining simple, cheap, and decentralized (no central controller required).

***

## 2.0 “Taking Turns” MAC Protocols & Cable Access

### 2.1 “Taking turns” MAC protocols
Different MAC protocols shine under different network loads:
*   **Channel partitioning MAC protocols** (like TDMA/FDMA) are efficient and fair at high loads, but highly inefficient at low loads (a single active node is limited to its $1/N$ bandwidth slice).
*   **Random access MAC protocols** (like CSMA) are extremely efficient at low loads (a single node can utilize the full channel bandwidth) but suffer from heavy collision overhead at high loads.

**"Taking turns" protocols** were designed to capture the best of both worlds, though they introduce their own complexities.

*   **Polling**:
    *   A primary central node "invites" other secondary nodes to transmit in turn. Often used with "dumb" devices.
    *   **Concerns**: High polling overhead (the invitation messages consume bandwidth), latency (waiting for your turn), and a single point of failure (if the primary node goes down, the network dies).
```text
* Visual Aid: Polling *

         [Primary Node]
          /    |    \
     poll/     |     \
        /      |      \
       v       v       v
   [Node 1] [Node 2] [Node 3]
    (Data)   (Wait)   (Wait)
```

*   **Token passing**:
    *   A control **token** (a special, small message) is passed sequentially from one node to the next. A node can only transmit data when it holds the token.
    *   **Concerns**: Token overhead, latency, and single point of failure (if a node crashes while holding the token, the network halts until recovery).
```text
* Visual Aid: Token Passing *

          [Node 1] (Holding Token 'T' -> Can send data)
           /    \
          /      \
[Node 4] <--    --> [Node 2] (Nothing to send, will pass token)
          \      /
           \    /
          [Node 3] 
```

***

### 2.2 Cable access network
Cable networks provide internet access over the existing coaxial television infrastructure, utilizing a combination of frequency, time, and random access protocols.

```text
* Visual Aid: Cable Access Network Topology *

[Cable Headend] 
      |
   [CMTS] (Cable Modem Termination System)
      |
      |============================ (Coaxial / Fiber Trunk)
      |         |         |
   [Splitter][Splitter][Splitter]
      |         |         |
   [Modem]   [Modem]   [Modem]
   (Residences with Cable Modems)
```

*   The network uses **FDM (Frequency Division Multiplexing)**, **TDM (Time Division Multiplexing)**, and **random access**.
*   **Downstream (Broadcast)**: Multiple FDM channels are used (up to 1.6 Gbps/channel). The CMTS broadcasts data downstream to all modems.
*   **Upstream (Multiple Access)**: Multiple upstream channels are available. Users contend for certain upstream time slots using random access, while other slots are explicitly assigned via TDM.

***

### 2.3 Cable access network: DOCSIS
**DOCSIS (Data Over Cable Service Interface Specification)** is the international standard governing cable modem networks.

*   **Topology**: It uses a **Point-to-Multipoint (P2MP)** tree-and-branch structure. All modems listen to the CMTS, but *modems cannot hear each other*.
*   **No Carrier Sense**: Because modems cannot hear each other's transmissions, CSMA/CD is physically impossible. 
*   **Solution**: The CMTS acts as a central scheduler. Modems use **Slotted ALOHA** to send request frames during a dedicated **contention window**.

```text
* Visual Aid: DOCSIS Downstream/Upstream Framing *

Downstream Channel (CMTS to Modems):
[====== MAP Frame (Interval t1 to t2) ======][ Data for Modem A ][ Data for B ]...

Upstream Channel (Modems to CMTS):
|<----------- MAP Interval [t1, t2] ----------->|
|                                               |
|-- Contention Minislots --|-- Assigned Minislots (Data) --|
| [Req][Req][Collision]    | [Modem A Data] [Modem B Data] |
```

*   **DOCSIS Operations**:
    *   **FDMA**: Divides upstream and downstream into frequency channels.
    *   **Downstream**: A continuous broadcast stream from the CMTS.
    *   **TDMA Upstream**: The CMTS broadcasts a `MAP` frame downstream. This map dictates which upcoming upstream slots are reserved for specific modems, and which slots are "contention slots" open for bandwidth requests.
    *   **Collision Handling**: Handled by the CMTS. Since modems lack collision detection, they realize a collision occurred if they do not receive an acknowledgment or a data grant in the next `MAP` frame.
    *   **Backoff**: Upon collision, modems use a **truncated binary exponential backoff** algorithm before re-requesting.

***

## 3.0 Summary of MAC protocols

### 3.1 Summary of MAC protocols
To recap, Multiple Access Control (MAC) protocols fall into three broad categories:
*   **Channel partitioning**: Divides the channel into smaller, dedicated "pieces" (Time Division, Frequency Division, Code Division).
*   **Random access (dynamic)**: Nodes transmit dynamically and recover from collisions. 
    *   Includes ALOHA, Slotted-ALOHA, CSMA, CSMA/CD (Ethernet), CSMA/CA (802.11 Wi-Fi).
    *   Carrier sensing is easy in wired technologies but difficult in wireless.
*   **Taking turns**: Nodes cooperate to share the channel.
    *   Includes polling from a central site and token passing (used in legacy systems like Token Ring, FDDI, and modern protocols like Bluetooth).

***

## 4.0 Local Area Network Basics & Addressing

### 4.1 Local Area Network Basics
To understand how data moves across networks, we must understand addressing at two layers:
*   **A.** MAC/link layer addresses (Layer 2).
*   **B.** Address Resolution Protocol (ARP) - the bridge between Layer 3 and Layer 2.
*   **C.** The routing and forwarding mechanics between different subnets.

***

### 4.2 A. MAC addresses
Nodes have two types of addresses for communication:

*   **32-bit IP address**: The network-layer (Layer 3) address. It is hierarchical and used for routing datagrams across the global internet to a specific subnet (e.g., `128.119.40.136`). Think of this as a *Postal Address*.
*   **MAC (LAN/Physical/Ethernet) address**: The link-layer (Layer 2) address. It is used "locally" to move frames from one physically-connected interface to another within the *same* subnet.
    *   It is a 48-bit address, typically burned into the Network Interface Card (NIC) ROM.
    *   Written in hexadecimal notation (e.g., `1A-2F-BB-76-09-AD`).

```text
* Visual Aid: LAN Nodes with MAC and IP Addresses *

               [ LAN (Wired/Wireless) IP Subnet: 137.196.7/24 ]
                /                      |                      \
 [Node A]                      [Node B]                      [Node C]
 IP: 137.196.7.23              IP: 137.196.7.78              IP: 137.196.7.14
 MAC: 71-65-F7-2B-08-53        MAC: 1A-2F-BB-76-09-AD        MAC: 58-23-D7-FA-20-B0
```

*   **Administration**: MAC addresses are centrally allocated and administered by the IEEE to ensure global uniqueness.
*   **MAC flat addressing**: MAC addresses have no geographical or topological hierarchy. This ensures **portability**—you can move a laptop from a LAN in New York to a LAN in Tokyo, and its MAC address remains exactly the same. (IP addresses, however, must change based on the subnet).

***

### 4.3 B. ARP: address resolution protocol
When a node knows the target IP address but needs to build a local frame, it must figure out the target's MAC address. This is the goal of **ARP (Address Resolution Protocol)**.

*   Every IP node (hosts and routers) on a LAN maintains an **ARP table**.
*   The table maps IP addresses to MAC addresses: `<IP address; MAC address; TTL>`.
*   **TTL (Time To Live)**: The time after which a mapping is forgotten (typically 20 minutes) to ensure the table stays updated as devices join and leave the network.

```text
* Visual Aid: ARP Tables *

Node A's Internal ARP Table:
+----------------+-------------------+------+
| IP Address     | MAC Address       | TTL  |
+----------------+-------------------+------+
| 137.196.7.78   | 1A-2F-BB-76-09-AD | 13:40|
+----------------+-------------------+------+
```

***

### 4.4 B. ARP protocol in action
If Node A wants to send a datagram to Node B, but Node B is missing from A's ARP table, A will use the ARP protocol:

*   **Step 1: ARP Query**
    A broadcasts an ARP query frame to the entire LAN. The frame contains B's IP address. Because it is a broadcast, the destination MAC is `FF-FF-FF-FF-FF-FF`.
```text
* Visual Aid: Node A Broadcasting ARP Query *
[Node A] -----> (Broadcast to FF-FF-FF-FF-FF-FF) "Who has IP 137.196.7.14?"
  |----> [Node C] (Ignores, IP doesn't match)
  |----> [Node D] (Ignores, IP doesn't match)
  \----> [Node B] (Matches! IP is 137.196.7.14)
```

*   **Step 2: ARP Response**
    Node B receives the broadcast, recognizes its own IP, and replies directly to Node A (unicast) with an ARP response frame containing its MAC address.
```text
* Visual Aid: Node B Sending ARP Response *
[Node B] -----> (Unicast to 71-65-F7-2B-08-53) "I am 137.196.7.14, my MAC is 58-23-D7-FA-20-B0" -----> [Node A]
```

*   **Step 3: Table Update**
    Node A receives the reply and caches the IP-to-MAC mapping in its local ARP table.
```text
* Visual Aid: Node A's Updated ARP Table *
Node A's Internal ARP Table:
+----------------+-------------------+------+
| IP Address     | MAC Address       | TTL  |
+----------------+-------------------+------+
| ...            | ...               | ...  |
| 137.196.7.14   | 58-23-D7-FA-20-B0 | 20:00| <-- New Entry!
+----------------+-------------------+------+
```

***

### 4.5 C. Routing to another subnet: addressing
How do MAC and IP addresses work together when sending data *outside* the local subnet? Consider Node A sending a datagram to Node B through Router R.

*   **Prerequisites**: Node A knows B's IP address (via DNS), the IP address of its first-hop Router R (via DHCP default gateway), and Router R's MAC address (via ARP).

```text
* Visual Aid: Packet traversing A -> R -> B *

[Node A] ------------------------> [Router R] -----------------------> [Node B]
IP: 111.111.111.111                IP(Left): 111.111.111.110           IP: 222.222.222.222
MAC: 74-29...                      MAC(Left): E6-E9...                 MAC: 49-BD...
                                   IP(Right): 222.222.222.220
                                   MAC(Right): 1A-23...

--- Hop 1: A to R ---
A creates the IP datagram. 
  IP Src:  A (111.111.111.111)  --> Note: IP addresses trace the end-to-end journey.
  IP Dest: B (222.222.222.222)

A encapsulates it into a Link-Layer Frame.
  MAC Src: A (74-29...)
  MAC Dest: R's Left MAC (E6-E9...) --> Note: MAC addresses only trace the single physical hop.

[Frame travels A -> R]

--- At Router R ---
Router R receives the frame, strips the Ethernet header, and passes the IP datagram up to its network layer.
It inspects the IP Dest (B) and determines the outgoing interface.

--- Hop 2: R to B ---
R encapsulates the datagram into a *new* Link-Layer Frame.
  IP Src:  A (111.111.111.111)  --> IP header remains completely unchanged!
  IP Dest: B (222.222.222.222)
  MAC Src: R's Right MAC (1A-23...)
  MAC Dest: B's MAC (49-BD...)

[Frame travels R -> B]
Node B extracts the datagram and passes it up the protocol stack.
```
*Crucial Takeaway*: IP addresses remain constant from source to final destination. MAC addresses change at every single router hop along the path.

***

## 5.0 Ethernet Design

### 5.1 Ethernet
Ethernet is the foundational technology for modern local area networking. Key topics include its basics, frame structure, switches, self-learning mechanisms, and VLANs.

***

### 5.2 a. Ethernet Basics & Topology
Ethernet is the "dominant" wired LAN technology because it was the first widely used, it is simple and cheap to implement, and it has scaled remarkably well (from 10 Mbps in the 1980s to 400 Gbps today).

```text
* Visual Aid: Metcalfe's Original Ethernet Sketch (Conceptual representation) *
     [Station]    [Station]    [Station]
         |            |            |
(Tap) ---+------------+------------+--- (Terminator)
             THE ETHER (Coaxial Cable)
```

*   **Physical Topology**:
    *   **Bus**: Popular through the mid-90s. All nodes connected to the same coaxial cable, meaning they were all in the same **collision domain** (transmissions could collide with anyone else on the wire).
    *   **Switched**: The topology that prevails today. Nodes connect to an active link-layer switch in the center (a star topology). Each "spoke" runs a separate Ethernet protocol instance, meaning nodes *do not collide* with each other. Switches isolate collision domains.

```text
* Visual Aid: Bus vs Switched Topology *

    BUS TOPOLOGY                 SWITCHED TOPOLOGY
       | |                              [PC]
  [PC]-+-+-[PC]                           |
       | |                         [PC]-[SWITCH]-[PC]
  [PC]-+-+-[PC]                           |
       | |                              [PC]
(Single Shared Wire)             (Active Central Hub)
```

***

### 5.3 b. Ethernet frame structure
The sending NIC encapsulates an IP datagram into an Ethernet frame.

```text
* Visual Aid: Ethernet Frame Structure *

+----------+--------------+--------------+------+----------------+-------+
| Preamble | Dest Address | Src Address  | Type | Data (Payload) |  CRC  |
| (8 bytes)|  (6 bytes)   |  (6 bytes)   |(2B)  |   (46-1500B)   | (4B)  |
+----------+--------------+--------------+------+----------------+-------+
```

*   **Preamble**: Used to synchronize the receiver and sender clock rates. Consists of 7 bytes of `10101010` followed by 1 byte of `10101011` (Start Frame Delimiter).
*   **Addresses**: 6-byte source and destination MAC addresses. If an adapter receives a frame with a matching or broadcast MAC, it accepts it; otherwise, it drops it.
*   **Type**: Indicates the higher-layer protocol (usually IP, but allows for others like AppleTalk). Used for demultiplexing.
*   **CRC (Cyclic Redundancy Check)**: Checked at the receiver. If an error is detected, the frame is immediately dropped.

***

### 5.4 Ethernet: unreliable, connectionless
Unlike TCP, Ethernet operates strictly at the link layer with a "best effort" delivery approach.
*   **Connectionless**: No handshaking process takes place between sending and receiving NICs before transmission.
*   **Unreliable**: The receiving NIC does *not* send ACKs (acknowledgments) or NAKs (negative acknowledgments). If a frame is dropped due to CRC errors or collisions, the Ethernet layer will not recover it. Recovery must be handled by higher layers (like TCP).
*   **MAC Protocol Used**: Older, shared-medium Ethernet uses **unslotted CSMA/CD with binary backoff**.

***

### 5.5 802.3 Ethernet standards: link & physical layers
"Ethernet" actually refers to a family of standards under the IEEE 802.3 umbrella.
*   While they all share a common MAC protocol and frame format, they support vastly different speeds and physical media.
*   **Media**: Copper (twisted pair) or Fiber optics.

```text
* Visual Aid: 802.3 Protocol Stack *

[ Application ]
[ Transport   ]
[ Network     ]
[ Link        ] <-- MAC protocol and frame format (Common to all)
+-------------+
|  Physical   | 
+-------------+ 
  /         \
 Copper      Fiber
 (e.g.,      (e.g.,
 100BASE-TX) 100BASE-FX,
             100BASE-SX)
```

***

## Key Terms to Define
*   **CSMA (Carrier Sense Multiple Access)**: A MAC protocol where nodes listen to the channel before transmitting to avoid collisions.
*   **CSMA/CD (Collision Detection)**: An extension of CSMA that actively listens during transmission and aborts if a collision is detected, saving channel time.
*   **Jam Signal**: A 32-48 bit signal broadcasted upon detecting a collision to ensure all nodes on the network recognize the collision occurred.
*   **Partial Collision**: An event in long cables where a collision's signal degrades, causing distant nodes to miss the collision; mitigated by the jam signal.
*   **Binary (Exponential) Backoff Algorithm**: A collision resolution algorithm that exponentially increases the range of possible random wait times after successive collisions.
*   **Slot time**: A discrete unit of time used in backoff calculations (e.g., 51.2 $\mu s$ in 10Mbps Ethernet).
*   **Channel partitioning MAC protocols**: Protocols (TDMA/FDMA) that divide channel resources statically; great for high loads, wasteful at low loads.
*   **Random access MAC protocols**: Protocols where nodes compete for the channel on-demand; great for low loads, prone to overhead at high loads.
*   **Taking turns protocols (Polling / Token passing)**: Protocols where nodes pass control (via primary node or a token) to avoid collisions, balancing efficiency but introducing latency overhead.
*   **DOCSIS (Data Over Cable Service Interface Specification)**: The international standard for providing internet access via existing coaxial cable TV infrastructure.
*   **Point-to-Multipoint (P2MP)**: A network topology with one central hub communicating with multiple end-nodes (e.g., CMTS to Cable Modems).
*   **Slotted ALOHA**: A random-access protocol where time is divided into discrete slots; used by DOCSIS modems to request bandwidth.
*   **32-bit IP address**: The hierarchical, logical layer-3 address used for routing data across networks.
*   **MAC / LAN / Ethernet address**: The flat, 48-bit physical layer-2 address burned into a NIC, used for local frame delivery.
*   **MAC flat addressing / Portability**: Because MACs are flat (not tied to a physical location), a device can move to any network in the world without changing its MAC address.
*   **Address Resolution Protocol (ARP) / ARP Table**: The protocol used to map a known IP address to an unknown MAC address on a local subnet, storing the results in an ARP Table.
*   **TTL (Time To Live)**: The lifespan of a cached entry (like in an ARP table) before it must be discarded or refreshed.
*   **Ethernet Frame (Preamble, Addresses, Type, CRC)**: The layer-2 data packet consisting of sync bits (preamble), routing info (MACs), payload type, and error checking (CRC).
*   **Connectionless / Unreliable Transmission**: A communication method (like Ethernet) that does not establish a prior connection and does not acknowledge receipt of data.
