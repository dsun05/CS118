## 1.0 Introduction

### 1.1 Outline

*   **Project 2**: A brief reminder of the ongoing practical assignment for the course.
*   **Lecture review: Network Layer**: We are focusing primarily on the Network Layer in this section.
    *   **Overview: Data vs. Control Plane**: The network layer operates in two planes. The *data plane* is responsible for moving packets from a router's input link to the appropriate output link (forwarding). The *control plane* determines the route or path that packets take from the source to the destination (routing algorithms).
    *   **IPv4/IPv6, DHCP**: These are the core protocols that govern addressing, formatting, and host configuration in the network layer.
*   **Midterm review**: The topics discussed represent key foundational knowledge necessary for the upcoming midterm examination.

***

## 2.0 Network Layer Overview

### 2.1 Network layer: overview

*   **Basic functions for the network layer**: The fundamental responsibility of the network layer is routing and forwarding. 
    *   **Forwarding/Routing**: **`Forwarding`** involves the localized action of transferring a packet from an incoming interface to the correct outgoing interface within a single router. **`Routing`** involves the network-wide process of determining the end-to-end path a packet must take from source to destination.
*   **Network service model**: Different architectures offer varying models of service. Traditional models could hypothetically guarantee the following:
    *   Guaranteed delivery
    *   Guaranteed delivery with bounded delay
    *   In-order packet delivery
    *   Guaranteed minimal bandwidth
    *   *Note on IP:* The Internet Protocol (IP) provides a **"best-effort" service**, meaning it does *none* of the above. It tries its best to deliver packets but offers no guarantees against loss, delay, or out-of-order delivery.
*   **Connection vs. connection-less delivery**:
    *   **Circuit Switching** requires a dedicated connection setup before data can flow (like traditional telephone networks).
    *   **Packet Switching** (used by the Internet) is connection-less. Packets are forwarded independently based on their destination addresses without prior circuit establishment.
*   **Network layer protocols**:
    *   **Addressing and fragmentation**: **`IPv4`** and **`IPv6`** are the foundational protocols establishing host identity and payload management.
    *   **Routing**: Protocols that construct routing tables include **`RIP`**, **`OSPF`**, **`BGP`**, **`DVMRP`**, and **`PIM`**.
    *   **Others**: Essential utility protocols include **`DHCP`** (address allocation), **`ICMP`** (error reporting/diagnostics), and **`NAT`** (address translation).

***

## 3.0 IPv4

### 3.1 A look at IP (IPv4) header

*   **OS prepends IP headers**: When an application sends data, the Operating System adds an IP header to the **`TCP`** or **`UDP`** transport segments to ensure they can navigate the network.
*   **First octet (byte) is often `0x45`**: 
    *   **`4`**: The **version field**. It is always 4 for an IPv4 packet.
    *   **`5`**: The **header length**. This is measured in 32-bit (4-byte) words. Thus, $5 \times 4 = 20$ bytes. This represents the standard length of an IPv4 header without any options.
*   **IPv4 headers**: Must always be a multiple of 32 bits. If optional fields are included that do not align, padding bits must be added.
*   **Differentiating transport payload**: The IP header contains a **`protocol`** field to tell the receiving OS which transport layer protocol to hand the payload off to. For example, `6` represents TCP, and `17` represents UDP.
*   **TTL field usage**: The **`Time To Live`** (TTL) field prevents packets from circulating endlessly in routing loops. It is decremented by 1 at every router hop. If it reaches 0, the packet is dropped.
*   **Source and Destination IP addresses**: Identifies the sender and the receiver. For example, in a DNS query, the Source IP is Jane's laptop, and the Destination IP is the DNS resolver.
*   **Address acquisition method**: **`DHCP`** (Dynamic Host Configuration Protocol) is typically used for a device (like Jane's laptop) to acquire its source IP address upon joining the network.

* Visual Aid: IPv4 Header diagram detailing fields *
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
|                       Source Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|             Payload Data (Typically a TCP or UDP segment)     |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 3.2 IPv4 Header (Specifics)

*   **Header length**: Measured in 4-byte units (e.g., value of 5 means 20 bytes).
*   **Length**: The total length of the packet (header + payload), measured in 1-byte units. Maximum size is 65,535 bytes.
*   **Fragmentation**: Uses three fields: 
    *   `id` (Identifier): Groups pieces of a fragmented packet together.
    *   `MF/DF`: Flags indicating "More Fragments" or "Do Not Fragment".
    *   `offset`: Indicates where in the original payload this fragment belongs, measured in 8-byte units.
*   **TTL**: Time To Live (hop limit).
*   **Checksum**: A redundancy query used for error-checking. 
    *   *Why just the header?* Calculating a checksum over the entire payload would be computationally expensive at every router. Since the TTL changes at every hop, the header checksum must be recalculated at every router. Therefore, IP restricts the checksum to just the header to speed up processing, leaving payload integrity to the Transport Layer (TCP/UDP).
*   **Protocol**: Identifies the upper-layer protocol (TCP/UDP).
*   **Source and destination IP addresses**: The 32-bit routing identifiers.

***

## 4.0 IP Addressing and Subnetting

### 4.1 IP address

*   **Globally recognizable identifier**: An IP address uniquely identifies a host interface on the internet.
*   **IPv4 range**: 32-bit addresses formatted as four decimal numbers ranging from `0.0.0.0` to `255.255.255.255`.
*   **Globally unique**: By design, public IP addresses must be globally unique so traffic can reach specific destinations. 
    *   *Exceptions*: Private network addresses behind NAT (Network Address Translation) routers can be reused across different internal networks because they are not directly routable on the public Internet.
*   **Network id, host id**: An IP address is logically divided into two parts. The Network ID identifies the specific physical/logical network, and the Host ID identifies the specific machine on that network.
*   **CIDR address**: Classless Inter-Domain Routing allows for flexible allocation of the Network ID length, replacing rigid legacy address classes.

### 4.2 IP address classes

* Visual Aid: Table of IP address classes *
```text
| Class | 1st Octet Decimal Range | High Order Bits | Network/Host ID Structure | Default Subnet Mask | Usable Hosts per Network |
|-------|-------------------------|-----------------|---------------------------|---------------------|--------------------------|
|   A   | 1 - 126                 | 0               | N.H.H.H                   | 255.0.0.0           | 16,777,214 (2^24 - 2)    |
|   B   | 128 - 191               | 10              | N.N.H.H                   | 255.255.0.0         | 65,534 (2^16 - 2)        |
|   C   | 192 - 223               | 110             | N.N.N.H                   | 255.255.255.0       | 254 (2^8 - 2)            |
|   D   | 224 - 239               | 1110            | Reserved for Multicasting | N/A                 | N/A                      |
|   E   | 240 - 254               | 1111            | Experimental / Research   | N/A                 | N/A                      |
```

* Visual Aid: Table of Private Networks *
```text
| Class | Private Network Subnet Mask | Address Range                     |
|-------|-----------------------------|-----------------------------------|
|   A   | 255.0.0.0                   | 10.0.0.0 - 10.255.255.255         |
|   B   | 255.240.0.0                 | 172.16.0.0 - 172.31.255.255       |
|   C   | 255.255.0.0                 | 192.168.0.0 - 192.168.255.255     |
```

### 4.3 Hierarchical addressing

*   **Subnet**: A localized portion of the address space. Subnetting allows administrators to break a larger network down into smaller, manageable logical networks.
    *   This is achieved by "borrowing" or extending bits from the Host ID portion to become part of the Network ID.
    *   Format: `<network address>/<subnet mask>`
*   **Route aggregation**: By arranging IP addresses hierarchically, ISPs can advertise a single large prefix to the global routing table rather than thousands of individual routes, making internet routing highly efficient.

### 4.4 CIDR (Classless Inter-Domain Routing) address

*   **Format**: `a.b.c.d/x`
    *   `x`: Denotes the number of bits defining the Network ID portion of the address.
    *   **Address**: `a.b.c.d` is the 32-bit IP address.
    *   **Network mask**: Derived mathematically as $2^{32} - 2^{(32-x)}$. The mask features `x` consecutive 1s followed by $(32-x)$ 0s.

* Visual Aid: Binary representations of a CIDR IP prefix *
```text
CIDR:        200.23.16.0/23

IP prefix:   11001000 00010111 00010000 00000000 (200.23.16.0)
Netmask:     11111111 11111111 11111110 00000000 (255.255.254.0)

Notice that the first 23 bits of the netmask are '1', defining the network boundary.
```

### 4.5 Quick question (Subnets)

*   **Identifying subnets within a network topology**: A subnet consists of interfaces that can physically reach one another without passing through an intervening router. Router interfaces act as the boundaries for subnets. In the diagram provided in the lecture, there are a total of **6 subnets**. 

* Visual Aid: Network diagram displaying 6 distinct subnets *
```text
Subnet 1: 223.1.1.0/24 (Top router interface connecting 3 host PCs)
Subnet 2: 223.1.2.0/24 (Bottom-left router interface connecting 2 host PCs)
Subnet 3: 223.1.3.0/24 (Bottom-right router interface connecting 2 host PCs)
Subnet 4: 223.1.9.0/24 (Point-to-point link between Top and Bottom-left routers)
Subnet 5: 223.1.8.0/24 (Point-to-point link between Bottom-left and Bottom-right routers)
Subnet 6: 223.1.7.0/24 (Point-to-point link between Bottom-right and Top routers)

Total Subnets = 3 (Host networks) + 3 (Router-to-router networks) = 6
```

***

## 5.0 IPv4 vs. IPv6

### 5.1 IPv4 & IPv6 Header Comparison

* Visual Aid: Comparative diagram of IPv4 and IPv6 headers *
```text
+-----------------------------------+   +-----------------------------------+
|            IPv4 Header            |   |            IPv6 Header            |
+-----------------------------------+   +-----------------------------------+
| Version | IHL |  Type of Service  |   | Version | Traffic Class| Flow Lbl |
+-----------------------------------+   +-----------------------------------+
|       Total Length                |   |       Payload Length              |
+-----------------------------------+   +-----------------------------------+
| Identification  |Flags| Frag Off  |   | Next Header |      Hop Limit      |
+-----------------------------------+   +-----------------------------------+
|   TTL   | Protocol| Header Check  |   |                                   |
+-----------------------------------+   |                                   |
|          Source Address           |   |          Source Address           |
|            (32-bit)               |   |            (128-bit)              |
+-----------------------------------+   |                                   |
|       Destination Address         |   |                                   |
|            (32-bit)               |   +-----------------------------------+
+-----------------------------------+   |                                   |
|      Options      |    Padding    |   |       Destination Address         |
+-----------------------------------+   |            (128-bit)              |
                                        |                                   |
                                        |                                   |
                                        +-----------------------------------+
```
*   **IPv6 header simplicity**: Despite supporting vastly larger addresses, the IPv6 header is structurally simpler than IPv4.
    *   **Fewer fields**: It strips out fragmentation fields, header lengths, and checksums.
    *   **Fixed length**: The standard IPv6 header is always exactly **40 bytes**, making hardware processing significantly faster.
*   **Fate of unkept fields**:
    *   **Disappeared**: `header length` is gone because the length is fixed. `header checksum` is removed entirely to reduce processing time at every router (relying on L2 and L4 checksums).
    *   **Moved**: Optional fields and fragmentation controls are moved to separate "Extension Headers" that sit between the standard IPv6 header and the transport payload, processing only at destinations when required.

### 5.2 IPv6/IPv4 differences

*   **Fixed-length 40 byte header**: Ensures fast, predictable processing. 
    *   **Length field**: In IPv6, the "Payload Length" field *excludes* the 40-byte header, unlike IPv4's Total Length which includes the header.
*   **Header Length field eliminated**: Because the header is always 40 bytes.
*   **Address length**: Increased massively from 32 bits (IPv4) to **128 bits** (IPv6).
*   **Priority/Traffic Class**: Specifies priority levels for different traffic types (usage rules still evolving).
*   **Flow Label**: A new field used to identify consecutive packets belonging to the same flow (e.g., streaming video), allowing routers to treat them identically without inspecting inner fields.
*   **Next header**: Replaces IPv4’s `Protocol` field. It identifies the type of data following the IPv6 header (e.g., a TCP segment or an Extension Header).
*   **Options**: Moved entirely outside the basic header, chained together via the `Next Header` pointers.
*   **Header Checksum**: Removed to speed up routing.

### 5.3 IPv6 address format & Special addresses (optional)

*   **Colon-Hex format**: Addresses are written as eight groups of four hexadecimal digits, separated by colons. 
    *   *Example*: `2607:F010:03f9:0000:0000:0000:0004:0001`
*   **Compression rules**: To make writing these long addresses easier:
    *   Leading zeros in a group can be skipped: `03f9` becomes `3f9`.
    *   One sequence of consecutive zero groups can be compressed to `::`. 
    *   *Compressed Example*: `2607:F010:3f9::4:1`
*   **Dot-decimal**: The last 32 bits can sometimes be represented in standard IPv4 dot-decimal notation for transition compatibility.
*   **Prefix specification**: Similar to CIDR, network sizes are defined using a `/length` notation (e.g., `2607:f010:3f9::/64`).
*   **Special Addresses**:
    *   `::/128` - Unspecified (similar to 0.0.0.0)
    *   `::1/128` - Loopback (similar to 127.0.0.1)
    *   `::ffff:0:0/96` - IPv4-mapped address space
    *   `2002::/16` - 6to4 transition addressing
    *   `ff00::/8` - Multicast addresses
    *   `fe80::/10` - Link-Local Unicast (used for local segment communication without global routing)

***

## 6.0 Fragmentation and Switching

### 6.1 IP fragmentation and reassembly

*   **MTU (Maximum Transmission Unit)**: The strict limit on the largest network layer packet that a given link-layer frame can carry (e.g., Ethernet standard MTU is 1500 bytes). If an IP datagram is larger than the MTU, the router must slice it into smaller fragments.
*   **Identifier**: A field copied to all fragments from the original datagram so the receiving host knows they belong together during reassembly.
*   **Flag bit (3 bits)**:
    *   **`DF` (Do not Fragment)**: If set to 1, the router will drop the packet rather than fragment it (often returning an ICMP error).
    *   **`MF` (More Fragments)**: If set to 1, it tells the receiver that more fragments of this datagram are coming. The final fragment has `MF=0`.
*   **Offset**: Dictates the data payload's correct position in the reassembled packet. Measured in 8-byte units.

* Visual Aid: Fragmentation of a 4000-byte datagram *
```text
Original Datagram: 4000 Bytes Total (20 byte IP Header + 3980 byte Payload)
MTU = 1500 Bytes

Router fragments into smaller datagrams:

Fragment 1: length=1500 | ID=x | MF=1 | offset=0    
(Payload = 1480 bytes. Offset = 0/8 = 0)

Fragment 2: length=1500 | ID=x | MF=1 | offset=185  
(Payload = 1480 bytes. Offset = 1480/8 = 185)

Fragment 3: length=1040 | ID=x | MF=0 | offset=370  
(Payload = 1020 bytes. Offset = (1480+1480)/8 = 370)
```

### 6.2 Quick question (Fragmentation)

*   **Calculating Fragments**: Given a packet with a total payload of 2380 bytes, header of 20 bytes (Total Length 2400), ID=12345, TTL=25, crossing a link with an MTU of 1450 Bytes.
    *   *Rule*: The maximum data per fragment must be divisible by 8. (MTU 1450 - 20 byte header = 1430 bytes. The closest multiple of 8 is 1424 bytes).

* Visual Aid: Packet calculation table *
```text
| Packet      | Header | Total Length | ID    | Flags | Offset | TTL | IP Payload |
|-------------|--------|--------------|-------|-------|--------|-----|------------|
| Fragment 1  | 20 B   | 1444 Bytes   | 12345 | 1 (MF)| 0      | 25  | 1424 Bytes |
| Fragment 2  | 20 B   | 976 Bytes    | 12345 | 0     | 178    | 25  | 956 Bytes  |

* Note: Offset 178 is derived from 1424 / 8. 
* Payload 1 + Payload 2 (1424 + 956) = 2380 total data bytes.
```

### 6.3 Switching

*   **Longest prefix matching**: When routers lookup a destination IP, they often find multiple matches in their forwarding table. They apply the rule of "longest prefix match," routing the packet out the interface associated with the most specific (longest) subnet mask.
*   **Linear lookup**: Comparing the destination IP against table entries line-by-line using wildcard masking syntax (`*`).

* Visual Aid: Longest Prefix Match Table *
```text
Destination Address Range                          Link interface
11001000 00010111 00011000 *********               0
11001000 00010111 00010*** *********               1
11001000 00010111 0001**** *********               2
******** ******** ******** ********                3
```

### 6.4 Tunneling

*   **Connecting private network via IP tunneling**: Used to connect logically separated secure networks (e.g., corporate branches) over the public internet by wrapping their private IP packets inside a public IP packet.
* Visual Aid: IPv4 Tunneling Diagram *
```text
[Smart Inc. West] --> [Router R1] ========= Internet ========= [Router R2] --> [Smart Inc. East]
(10.2.1.1)            (132.17.231.5)                           (17.5.13.9)     (10.3.16.1)

Original Packet:  Src: 10.2.1.1  Dest: 10.3.16.1 
Encapsulated:     Src: 132.17.231.5  Dest: 17.5.13.9  [ Payload: Original Packet ]
```
*   **IPv6 through IPv4 network Tunneling**: Because many transit routers only support IPv4, IPv6 packets are fully encapsulated inside the payload portion of an IPv4 packet to traverse the legacy network.
* Visual Aid: IPv6 in IPv4 Tunneling *
```text
  IPv6 Packet to Transmit:  [ IPv6 Header ] [ TCP/UDP Payload ]
  
  Traversing IPv4 Internet: [ IPv4 Header ] [ IPv6 Header ] [ TCP/UDP Payload ]
  (Protocol field in IPv4 header is set to 41, indicating an IPv6 payload)
```

***

## 7.0 DHCP Configuration

### 7.1 DHCP: Dynamic Host Configuration Protocol

*   **Dynamically allocates host info**: Instead of manual configuration, DHCP automates the provision of critical network parameters to host machines joining a network. It provides:
    *   **IP address** for the host.
    *   **IP address for default router** (Default Gateway).
    *   **Subnet mask** (to identify the local network boundary).
    *   **IP address for DNS caching resolver**.
*   **Allows address reuse**: DHCP allows hosts to "lease" an IP temporarily. Once they disconnect, that IP goes back into the pool for another machine to use, preserving addressing space.

### 7.2 DHCP: operations & client-server scenario

*   DHCP uses a four-step DORA process (Discover, Offer, Request, ACK). Because the new client has no IP address, initial communication relies heavily on broadcast addresses.

* Visual Aid: Sequence diagram for DHCP messages *
```text
New Client (0.0.0.0)                                    DHCP Server (223.1.2.5)
      |                                                           |
      | 1. DHCP Discover (Broadcast)                              |
      | Src: 0.0.0.0:68 | Dest: 255.255.255.255:67                |
      |---------------------------------------------------------->|
      |                                                           |
      | 2. DHCP Offer (Broadcast)                                 |
      | Src: 223.1.2.5:67 | Dest: 255.255.255.255:68              |
      | Payload: "I'm a server! Use 223.1.2.4"                    |
      |<----------------------------------------------------------|
      |                                                           |
      | 3. DHCP Request (Broadcast)                               |
      | Src: 0.0.0.0:68 | Dest: 255.255.255.255:67                |
      | Payload: "I'll take 223.1.2.4!"                           |
      |---------------------------------------------------------->|
      |                                                           |
      | 4. DHCP ACK (Broadcast)                                   |
      | Src: 223.1.2.5:67 | Dest: 255.255.255.255:68              |
      | Payload: "Confirmed. Lease lifetime 3600s"                |
      |<----------------------------------------------------------|
```

### 7.3 Quick question (DHCP updates)

*   **Analyzing DHCP update requirements**:
    *   If a host moves between switch ports **within the same subnet**, the IP address can remain the same. The DHCP lease simply persists.
    *   If a host physically unplugs and moves to a **different subnet** (behind a different router interface), it *must* undergo the DHCP process again to acquire an IP address matching that new subnet's prefix.

* Visual Aid: DHCP Transition Scenarios *
```text
Scenario A: Host PC0 moves to the same switch PC1 is attached to.
Result: The Subnet has not changed. PC0 can likely renew/retain its same IP address.

Scenario B: Host PC0 unplugs and moves to the switch PC3 is attached to.
Result: The Subnet HAS changed. The broadcast domain is completely different. 
DHCP must be updated, and PC0 must acquire a new IP address matching the new subnet prefix.
```

***

### **Key Terms to Define**

*   **Forwarding/Routing**: Forwarding is the localized hardware action of moving a packet from an input link to an output link on a router. Routing is the network-wide software process of determining the end-to-end path.
*   **Connection / Connection-less Delivery**: Connection delivery (Circuit Switching) establishes a dedicated end-to-end path prior to data transfer. Connection-less delivery (Packet Switching) treats every packet independently, routing them on the fly.
*   **DHCP (Dynamic Host Configuration Protocol)**: A network management protocol used on UDP/IP networks whereby a server dynamically assigns an IP address and other network configuration parameters to each device.
*   **TTL (Time To Live)**: An 8-bit field in the IPv4 header that decrements by 1 at every router hop. It prevents routing loops by destroying packets that have been circulating for too long.
*   **Subnet**: A logical subdivision of an IP network. The practice of dividing a network into two or more networks is called subnetting.
*   **CIDR (Classless Inter-Domain Routing)**: A method for allocating IP addresses and IP routing that replaced the older rigid Class A/B/C architecture, allowing flexible subnet masks (e.g., `/23`).
*   **MTU (Maximum Transmission Unit)**: The size (in bytes) of the largest protocol data unit that a specific communications protocol layer can pass onwards.
*   **DF (Do not Fragment) / MF (More Fragments)**: Control flags in the IP header. DF prevents a router from splitting a packet. MF tells the receiving host that more pieces of the packet are en route.
*   **Fragment Offset**: A field indicating the exact position of a fragment's data relative to the beginning of the data in the original unfragmented packet (measured in 8-byte blocks).
*   **Longest Prefix Matching**: An algorithm used by routers to select an entry from a routing table. If multiple network prefixes match a destination IP, the router chooses the one with the longest subnet mask.
*   **IP Tunneling**: The process of encapsulating one IP protocol packet inside another, commonly used to connect disjointed networks (like private LANs or IPv6 islands) securely across the public Internet.
