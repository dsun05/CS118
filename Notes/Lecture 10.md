## 1.0 Evolving Transport-Layer Functionality

### 1.1 Evolving transport-layer functionality

For nearly 40 years, the internet has primarily relied on two transport protocols: **TCP** (Transmission Control Protocol) and **UDP** (User Datagram Protocol). TCP provides reliable, connection-oriented data delivery, while UDP offers a lightweight, connectionless, and unreliable service. 

As network environments became more diverse, standard TCP struggled to perform optimally across all conditions. To address this, engineers developed different "flavors" or variants of TCP optimized for specific operational scenarios. 

```text
+------------------------------------+---------------------------------------------------+
| Scenario                           | Specific Network Challenges                       |
+------------------------------------+---------------------------------------------------+
| Long, fat pipes                    | Many packets are "in flight" simultaneously. A    |
| (Large data transfers)             | single packet loss can shut down the pipeline,    |
|                                    | drastically reducing throughput.                  |
+------------------------------------+---------------------------------------------------+
| Wireless networks                  | Packet loss is frequently caused by noisy links   |
|                                    | or mobility, but TCP mistakenly treats this as    |
|                                    | congestion loss, unnecessarily slowing down.      |
+------------------------------------+---------------------------------------------------+
| Long-delay links                   | Extremely long Round-Trip Times (RTTs) make       |
|                                    | standard TCP recovery and scaling very slow.      |
+------------------------------------+---------------------------------------------------+
| Data center networks               | Environments are highly latency-sensitive;        |
|                                    | microsecond delays impact application performance.|
+------------------------------------+---------------------------------------------------+
| Background traffic flows           | Needs to be treated as low priority so it doesn't |
|                                    | compete with primary user-facing TCP flows.       |
+------------------------------------+---------------------------------------------------+
```

Despite these TCP variants, the modern trend is shifting toward moving complex transport-layer functions *out* of the OS kernel and up into the application layer, running them on top of the simpler UDP protocol. The most prominent example of this evolution is **HTTP/3**, which is powered by a new protocol called **QUIC**.

***

### 1.2 QUIC: Quick UDP Internet Connections

**QUIC (Quick UDP Internet Connections)** is an application-layer protocol built on top of UDP. Its primary goal is to increase the performance of HTTP by eliminating the inefficiencies of traditional TCP-based web traffic. It has already seen massive deployment, notably on Google servers and applications (like Chrome and the mobile YouTube app).

```text
      TRADITIONAL STACK                      NEW HTTP/3 STACK
+---------------------------+          +---------------------------+
|                           |          |                           |
|          HTTP/2           |          |      HTTP/2 (Slimmed)     |
|                           |          |                           |
+---------------------------+          +---------------------------+
|                           |          |                           |
|           TLS             |          |           QUIC            |
|       (Encryption)        |          | (App-Layer Reliability &  |
+---------------------------+          |   Congestion Control)     |
|                           |          +---------------------------+
|           TCP             |          |                           |
|                           |          |           UDP             |
+---------------------------+          +---------------------------+
|                           |          |                           |
|            IP             |          |            IP             |
|                           |          |                           |
+---------------------------+          +---------------------------+
       "HTTP/2 over TCP"                     "HTTP/3 over UDP"
```

Rather than reinventing the wheel, QUIC intelligently adopts established approaches from TCP:
*   **Error and congestion control**: QUIC utilizes loss detection and congestion control algorithms that heavily parallel well-known, highly tested TCP algorithms. 
*   **Connection establishment**: QUIC combines reliability, congestion control, authentication, encryption, and state establishment into a single streamlined handshake (1 RTT).

Furthermore, QUIC allows for the multiplexing of multiple application-level "streams" over a single QUIC connection. While all streams share a common congestion control state to be fair to the network, they maintain separate reliable data transfer and security states.

***

### 1.3 QUIC: Connection establishment

One of the largest performance bottlenecks in traditional web traffic is connection setup. 

*   **Traditional TCP + TLS**: Requires two serial handshakes. First, the TCP transport-layer handshake establishes connection state and reliability. Second, the TLS security handshake establishes authentication and cryptographic keys. This results in at least a 2-RTT (Round Trip Time) delay before any actual data can be sent.
*   **QUIC**: Combines both processes. Reliability, congestion control, authentication, and cryptographic state are negotiated simultaneously, resulting in a **1-RTT delay**.

```text
       TRADITIONAL TCP/TLS HANDSHAKE (2 RTT)                  QUIC HANDSHAKE (1 RTT)
       
   Client                         Server               Client                         Server
     |                              |                    |                              |
     | ------- TCP SYN -----------> |                    | -------- QUIC Setup -------> |
     | <------ TCP SYN/ACK -------- | (1 RTT)            | <------- QUIC Setup -------- | (1 RTT)
     | ------- TCP ACK / TLS -----> |                    | -------- Data -------------> |
     | <------ TLS ACK ------------ | (2 RTT)            |                              |
     | -------- Data -------------> |                    |                              |
```

***

### 1.4 QUIC: streams: parallelism, no HOL blocking

When multiple pieces of data (e.g., different images for a webpage) are sent over a single traditional TCP connection, they are placed in a single strict sequence. If one packet is lost, the receiver must wait for it to be retransmitted before it can process *any* of the subsequent packets, even if they arrived perfectly fine. This phenomenon is known as **HOL blocking (Head of Line blocking)**.

QUIC solves this by supporting multiple independent streams within a single connection. If an error occurs in one stream, only that specific stream pauses for retransmission. The other streams continue processing in parallel, eliminating HOL blocking.

```text
      HTTP 1.1 (TCP/TLS) - HOL Blocking           HTTP/3 (QUIC) - Parallel Streams, No HOL Blocking
      
      [ Stream 1: GET Image A ]                   [ QUIC Stream 1: Image A ]
      [ Stream 2: GET Image B ]                   [ QUIC Stream 2: Image B ]
      [ Stream 3: GET Image C ]                   [ QUIC Stream 3: Image C ]
               |                                          |           |           |
        +------+------+                             +-----+-----+ +---+---+ +-----+-----+
        |     TLS     |                             |  Encrypt  | | Encrypt| |  Encrypt  |
        +------+------+                             +-----+-----+ +---+---+ +-----+-----+
               |                                          |           |           |
        +------+------+                               +---+---+   +---+---+   +---+---+
        |   TCP RDT   | <-- ERROR HERE blocks ALL     | RDT   |   | RDT   |   | RDT   | <-- Error here
        +------+------+     subsequent data.          +---+---+   +---+---+   +---+---+     only blocks
               |                                          |           |           |         Stream 3!
        +------+------+                               +-------------------------------+
        | Cong. Cont. |                               |    QUIC Congestion Control    |
        +-------------+                               +---------------+---------------+
                                                                      |
                                                                   [ UDP ]
```

***

### 1.5 Chapter 3: summary

Before moving deeper into the network stack, let's summarize the Transport Layer:
*   **Transport layer services**: Primarily responsible for multiplexing and demultiplexing data to the correct application processes, utilizing UDP for lightweight, unreliable transport.
*   **TCP Focus**: TCP provides **reliable data transfer**, complex connection management, flow control (preventing the sender from overwhelming the receiver), and **congestion control** (preventing the sender from overwhelming the network).

**Up next:** We transition from the network "edge" (Applications and Transport) into the network "core"—exploring the Network-layer data plane and control plane.

***

## 2.0 Chapter 4: Network Layer: The Data Plane

### 2.1 Layering in Internet protocol stack

As we move down the protocol stack, we arrive at the Network Layer. The core responsibility of this layer is **best-effort global packet delivery**. It achieves this by relying on the layers below it (Link and Physical layers) which provide best-effort local packet delivery. 

```text
+-----------------------+ 
|     Application       | <-- Messages
+-----------------------+
|      Transport        | <-- Segments (Reliable/Unreliable)
+-----------------------+
| >>>   NETWORK   <<<   | <-- Datagrams (Global best-effort packet delivery)
+-----------------------+
|        Link           | <-- Frames (Local best-effort packet delivery)
+-----------------------+
|      Physical         | <-- Bits
+-----------------------+
```

***

### 2.2 “Data plane” roadmap

To understand the Network layer, we separate it into two major conceptual areas: the **data plane** and the **control plane**. 
This chapter provides a roadmap focusing on:
1.  **Network layer: overview** (defining the data plane vs. the control plane).
2.  Inside a router (hardware components like input/output ports, switching fabrics, and packet scheduling).
3.  **IP: the Internet Protocol** (the format of datagrams, addressing, NAT, and IPv6).
4.  Generalized Forwarding and SDN (Match+action paradigms).
5.  Middleboxes (firewalls, load balancers, etc.).

```text
    ========================================================
   ||   _ |                                            | _   ||
   ||  ( )|     "Bridging the Gap: Data & Control"     |( )  ||
   ||___|_|____________________________________________|_|___||
      | |                                              | |
     /   \                                            /   \
   [ROUTER]------------------------------------------[ROUTER]
```

***

### 2.3 Highlights of Chapter 4

For the scope of our immediate study, we will strictly focus on the following core highlights:
1.  **Network layer overview**
2.  **IP protocol**

*Note: Topics such as deep router internals, Software-Defined Networking (SDN) specifics, generalized forwarding, and middleboxes are excluded from this specific note set.*

***

## 3.0 Network layer overview

### 3.1 Key functions & Service model

The network layer has two distinct, vital functions: **Routing and forwarding**. Together, they form the operations of the control plane and the data plane. Additionally, the network layer operates on a specific **service model**, which defines the guarantees (or lack thereof) provided to data traveling across the network.

***

### 3.2 Network-layer services in the layered protocol stack

Unlike the transport layer, which logically connects end-to-end (host to host), the network layer provides **hop-by-hop delivery**. Network layer protocols exist in *every single Internet device*—both hosts and routers.

*   **Sender**: Takes transport-layer segments, encapsulates them into network-layer *datagrams*, and passes them down to the link layer.
*   **Receiver**: Extracts segments from incoming datagrams and delivers them up to the transport layer.
*   **Routers**: 
    *   Examine header fields in passing IP datagrams.
    *   Move datagrams from input ports to the correct output ports to advance them along their path.

```text
 [ Sending Host ]                                                      [ Receiving Host ]
   App->Trans->Net                                                       Net->Trans->App
       |                                                                        ^
       v                                                                        |
   [ Router A ] -----( ISP Network )-----> [ Router B ] -----( Datacenter )-----+
  (Net->Link->Phy)                        (Net->Link->Phy)
```

***

### 3.3 Two key network-layer functions

It is crucial to differentiate between the network layer's two primary tasks:

1.  **Forwarding**: The hardware-level process of moving arriving packets from a router's input link to the appropriate output link.
    *   *Analogy*: Navigating through a single complex highway interchange. You look at a sign and take the immediate correct exit.
2.  **Routing**: The software-level process of determining the overall end-to-end path that packets will take from their source to their destination.
    *   *Components*: Powered by routing algorithms and routing protocols (like OSPF or BGP).
    *   *Analogy*: Planning the entire road trip from New York to California on a map before leaving.

```text
     FORWARDING (Local Action)                ROUTING (Global Planning)
     
        [ Input Port 1 ]                        (Source) 
               |                                   |
         +-----------+                            [R1]----[R2]
         |  ROUTER   |                             | \    /|
         +-----------+                             |  [R3] |
            /     \                                | /    \|
 [Output Port A] [Output Port B]                  [R4]----[R5]
                                                   |
                                              (Destination)
```

***

### 3.4 Network layer: data plane vs. control plane

These two functions map directly to the two planes of the network layer:

*   **Data plane**: Represents **data forwarding**.
    *   This is a *local*, per-router function.
    *   It determines how a datagram arriving on an input port is forwarded to an output port based on values in the arriving packet's header. It operates at nanosecond timescales in hardware.
*   **Control plane**: Represents **routing**.
    *   This constitutes *network-wide logic*.
    *   It determines the end-to-end path.
    *   There are two distinct approaches to building the control plane:
        1.  *Traditional routing algorithms*: Computed collaboratively by software inside the routers themselves.
        2.  **SDN (Software-Defined Networking)**: Computed centrally by remote servers and pushed down to the routers.

```text
[ Arriving Packet: Dest IP 0111 ] ---> [ ROUTER DATA PLANE ] ---> Forwards out Port 2
                                               ^
                                               | (Consults Forwarding Table)
                                               |
                                     [ ROUTER CONTROL PLANE ]
```

***

### 3.5 Per-router control plane

In traditional networking, individual routing algorithm components reside *in each and every router*. These routers communicate with one another using routing protocols to share network state information. Based on this shared intelligence, every router computes its own local forwarding table.

```text
        +------------+               +------------+
        | Routing    | <--(OSPF)-->  | Routing    |
        | Algorithm  |               | Algorithm  |
        +------------+               +------------+
              |                            |
       [ Forwarding Table ]         [ Forwarding Table ]
              |                            |
          (Router A)                   (Router B)
```

***

### 3.6 Software-Defined Networking (SDN) control plane

In contrast, **SDN (Software-Defined Networking)** separates the brain from the brawn. A centralized, remote controller server gathers network information, computes the best routes, and then directly installs the resulting forwarding tables into the individual routers via Control Agents (CA). The routers themselves merely forward packets without calculating the routes.

```text
                       [ CENTRALIZED REMOTE CONTROLLER ]
                               /       |       \
                              /        |        \
                             v         v         v
                         +----+     +----+     +----+
                         | CA |     | CA |     | CA |
                         +----+     +----+     +----+
                         | FW |     | FW |     | FW |
                         | Tbl|     | Tbl|     | Tbl|
                         +----+     +----+     +----+
                        Router 1   Router 2   Router 3
```

***

### 3.7 Network service model

When sending data through the network "channel", we must define the service model. A network could theoretically offer guarantees for:
*   *Individual datagrams*: e.g., Guaranteed delivery, or delivery within a strict timing boundary (less than 40ms delay).
*   *Flow of datagrams*: e.g., In-order delivery, guaranteed minimum bandwidth, or strict limits on inter-packet spacing (jitter).

***

### 3.8 Internet service model today

The architecture of the Internet utilizes a **best effort service model**. It is fundamentally a "best effort global packet delivery" system. 

Under the Internet's best-effort model, there are explicitly **no guarantees** regarding:
*   Successful datagram delivery (packets can be dropped).
*   Timing or exact order of delivery (packets can be delayed or arrive out of sequence).
*   Available bandwidth (congestion can throttle throughput).

```text
+-------------------+---------------+--------------------------------------------------+
| Network Arch.     | Service Model | Guarantees Provided?                             |
|                   |               | Bandwidth | Loss | Order | Timing                |
+-------------------+---------------+-----------+------+-------+-----------------------+
| Internet          | Best Effort   | NONE      | NO   | NO    | NO                    |
+-------------------+---------------+-----------+------+-------+-----------------------+
```

***

### 3.9 Other network-layer service models

Historically, other network architectures attempted to provide strict Quality of Service (QoS) guarantees:
*   **ATM Constant Bit Rate / Available Bit Rate**: ATM (Asynchronous Transfer Mode) networks provided strict guarantees on timing, order, and loss.
*   **Intserv (Guaranteed) / Diffserv**: Attempts to add guaranteed bandwidth or differentiated priority classes over the internet.

```text
+-------------------+-----------------------+-----------+--------+-------+--------+
| Network Arch.     | Service Model         | Bandwidth | Loss   | Order | Timing |
+-------------------+-----------------------+-----------+--------+-------+--------+
| Internet          | best effort           | none      | no     | no    | no     |
| ATM               | Constant Bit Rate     | constant  | yes    | yes   | yes    |
| ATM               | Available Bit Rate    | guar. min | no     | yes   | no     |
| Internet          | Intserv Guaranteed    | yes       | yes    | yes   | yes    |
| Internet          | Diffserv              | possible  | possibly| possibly| no  |
+-------------------+-----------------------+-----------+--------+-------+--------+
```

***

### 3.10 Reflections on best-effort service

If the best effort service model provides zero guarantees, why is the Internet so massively successful?
1.  **Simplicity of mechanism**: Stripping out complex guarantee mechanisms at the core network layer made routers cheaper, faster, and infinitely easier to widely deploy.
2.  **Sufficient provisioning of bandwidth**: Telecoms have continually upgraded link capacities (fiber optics). When bandwidth is abundant, real-time apps perform "good enough" most of the time without needing strict mathematical guarantees.
3.  **Replicated, application-layer distributed services**: Content Delivery Networks (CDNs) place data geographically close to end-users, bypassing long, delay-prone network paths.
4.  **"Elastic" services**: Transport layer congestion control (like TCP) adapts intelligently to available bandwidth, allowing elastic apps (like web browsing and file transfers) to thrive.

***

## 4.0 #2 IP Protocol

### 4.1 Intro to IP Protocol

The Internet Protocol (IP) is the heart of the network layer. Our focus will be on:
*   The structure and fields of the IPv4 header.
*   **IP addressing** (how addresses are unique, how they integrate into routing, and address space exhaustion).
*   IPv6 (its differences from IPv4 and practical deployment strategies).

***

### 4.2 Recall Network Layer functions of the Internet

Inside the network layer of an Internet host or router, three major components interact to make global delivery possible:

```text
   +-------------------------------------------------------------------------+
   |                     Network Layer Functions                             |
   |                                                                         |
   |   1. Path-Selection Algorithms             2. IP Protocol               |
   |      (OSPF, BGP, SDN Controllers)             - Datagram format         |
   |            |                                  - Addressing              |
   |            |                                  - Handling conventions    |
   |            v                                                            |
   |      [ Forwarding Table ]                                               |
   |                                                                         |
   |                                            3. ICMP Protocol             |
   |                                               - Error reporting         |
   |                                               - Router signaling        |
   +-------------------------------------------------------------------------+
```

***

### 4.3 IP Header Fields: IP Datagram format

An IPv4 datagram consists of a header and a payload. The header is typically 20 bytes long and is processed in 32-bit (4-byte) wide rows. 

When combining the standard 20 bytes of IP header with 20 bytes of TCP header, the total application layer **overhead** is 40 bytes per packet before any actual data payload is sent.

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |Type of Service|          Total Length         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |         Header Checksum       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source IP Address                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination IP Address                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (if any) ...                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|          Payload Data (Variable Length, e.g. TCP/UDP)         |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

***

### 4.4 Notes on IP header fields

Each field in the IP header serves a critical purpose:
*   **TTL (Time to Live)**: This is *not* measured in wall-clock time (seconds), but in a "hop count." It is decremented by 1 at every router. If it reaches 0, the packet is discarded. This prevents orphaned IP packets from traveling eternally in routing loops.
*   **Lengths**: Because both the header (due to optional fields) and the payload can be of variable lengths, the header includes specific fields to define the boundary of both.
*   **Checksum**: Verifies the integrity of the header itself (not the payload) to ensure routing decisions aren't made on corrupted IP addresses.
*   **Fragmentation fields** (16-bit identifier, 3-bit flags, fragment offset): If an IP packet is too large to fit across a specific physical link (like an Ethernet cable's MTU), the router fragments the packet. These fields allow the receiving host to correctly re-assemble the fragments.

***

### 4.5 IP addressing: introduction

An **IP address** is a 32-bit identifier associated with each host or router interface.
*   A **network interface** (or just **interface**) is the boundary connection between a host/router and the physical link.
*   Because routers connect multiple links together, they typically have multiple interfaces (and thus, multiple IP addresses). 
*   Hosts typically have one or two interfaces (e.g., wired Ethernet, wireless WiFi).

IP addresses are written in "dotted-decimal" notation, breaking the 32 bits into four 8-bit blocks (bytes) represented as decimal numbers.

*Example:* `223.1.1.1` in binary is `11011111 00000001 00000001 00000001`

```text
       [Host 223.1.1.2]                    [Host 223.1.2.1]
             \                                    /
              \                                  /
            [ Switch ] ---- (223.1.1.4)[Router](223.1.2.9) ---- [ Switch ]
              /                                  \                  |
             /                                    \            [Host 223.1.2.2]
       [Host 223.1.1.1]                    [Host 223.1.3.27]
```

***

### 4.6 Subnets

A **subnet** (subnetwork) is a grouping of device interfaces that can physically reach each other *without passing through an intervening router*. In our example above, the switch connects hosts `223.1.1.1`, `223.1.1.2`, and the router's interface `223.1.1.4` into a single subnet.

To facilitate routing, an IP address is logically divided into two parts:
1.  **Subnet part**: The high-order bits of the address. Devices existing in the same physical subnet will all share the exact same subnet part. 
2.  **Host part**: The remaining low-order bits, which uniquely identify the specific device within that subnet.

```text
            Subnet 1 (223.1.1.x)                   Subnet 2 (223.1.2.x)
          ........................               ........................
          :  [223.1.1.1]         :               :  [223.1.2.1]         :
          :         \            :               :         /            :
          : [223.1.1.2] -[Router Interface]--[Router Interface]- [223.1.2.2]
          :              223.1.1.4               :  223.1.2.9           :
          ........................               ........................
                              |
                       [Router Interface 223.1.3.27]
                              |
                        .............
                        : Subnet 3  :
                        : 223.1.3.x :
                        .............
```
