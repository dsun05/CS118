## 1.0 IP Protocol

### 1.1 IP addressing: introduction
To understand how devices communicate across the Internet, we first need to understand how they are identified. An **IP address** is a 32-bit identifier strictly associated with a **network interface**, not the device itself. 

A **network interface** is the physical and logical boundary between a host or router and the physical link. 
* **Routers:** By definition, routers connect multiple networks together, meaning they typically have multiple interfaces (one for each network they connect to).
* **Hosts:** End-user devices typically have one or two interfaces (e.g., one for wired Ethernet and one for wireless 802.11 WiFi).

Because 32 bits are difficult for humans to read, IPv4 addresses are written in **dotted-decimal notation**. The 32 bits are divided into four 8-bit bytes, and each byte is translated into a decimal number between 0 and 255.
* *Example:* `223.1.1.1` translates to `11011111 00000001 00000001 00000001` in binary.

*Visual Aid: Network topology diagram showing hosts, routers, interfaces, and their dotted-decimal IP addresses.*
```text
      [Host: 223.1.1.1]
             |
      [Host: 223.1.1.2] --- (Subnet 1: 223.1.1.x)
             |
      [Host: 223.1.1.3]
             |
      [Router Interface: 223.1.1.4]
             |
          (Router) --- [Interface: 223.1.2.9] --- (Subnet 2) --- [Host: 223.1.2.1, Host: 223.1.2.2]
             |
      [Router Interface: 223.1.3.27]
             |
         (Subnet 3: 223.1.3.x)
             |
      [Host: 223.1.3.1, Host: 223.1.3.2]
```

*Visual Aid: Diagram demonstrating how wired Ethernet interfaces connect via switches and wireless interfaces connect via base stations.*
```text
 Wired Ethernet Connection:                  Wireless Connection:
 
 [Host A] ----+                              [Mobile Host]
              |                                     |
 [Host B] -- [Ethernet Switch]               [WiFi Base Station]
              |                                     |
 [Router] ----+                              [Router]
```

***

### 1.2 Subnets
What exactly is a **subnet** (subnetwork)? From an architectural standpoint, a subnet consists of device interfaces that can physically reach each other without passing through an intervening router. 

IP addresses are hierarchical and are broken into two distinct parts:
1. **Subnet part**: The common high-order (leftmost) bits shared by all devices on the same subnet.
2. **Host part**: The remaining low-order (rightmost) bits that uniquely identify specific devices within that subnet.

**Recipe for defining subnets:** If you were to visually detach every interface from its host or router, the remaining isolated "islands" of networks represent your subnets. 

To determine how many bits belong to the subnet part, we use a **subnet mask**. For example, a mask of `/24` means the first (high-order) 24 bits of the IP address define the subnet.

*Visual Aid: Multiple network diagrams highlighting isolated subnet islands and their corresponding subnet masks (e.g., 223.1.1.0/24).*
```text
 Island 1: subnet 223.1.1.0/24
 Island 2: subnet 223.1.2.0/24
 Island 3: subnet 223.1.3.0/24
 
       (Subnet 223.1.1.0/24)
          /             \
 [Hosts 223.1.1.x]    [Router Int. 223.1.1.4]
                             |
                      [Router Core]
                     /             \
   [Router Int. 223.1.2.9]       [Router Int. 223.1.3.27]
           |                               |
 (Subnet 223.1.2.0/24)             (Subnet 223.1.3.0/24)
           |                               |
 [Hosts 223.1.2.x]                 [Hosts 223.1.3.x]
```

***

### 1.3 IP addressing: CIDR
Historically, subnets were divided into rigid, fixed-size classes (Class A, B, C). This was inefficient, leading to the creation of **CIDR** (Classless InterDomain Routing). 
* CIDR allows the subnet portion of an IP address to be of an arbitrary length.
* It uses the format `a.b.c.d/x`, where `x` represents the exact number of bits in the subnet portion.

*Visual Aid: Diagram illustrating the division between the subnet part and host part of a 32-bit binary IP address.*
```text
  <------------- 32 bits ------------->
  <--- subnet part ---> <--- host part --->
  11001000 00010111 00010000  00000000
  
  CIDR Notation: 200.23.16.0/23
  (The first 23 bits define the network, the remaining 9 bits define the host)
```

***

### 1.4 IP addresses: how to get one?
There are two core questions regarding IP allocation:
1. How does a single host get an IP address within its network?
2. How does a network get its overall subnet address block?

For a host to get an IP address, it can either be hard-coded by a system administrator in configuration files, or dynamically assigned using **DHCP** (Dynamic Host Configuration Protocol), which enables "plug-and-play" networking.

***

### 1.5 DHCP: Dynamic Host Configuration Protocol
**DHCP** operates on a client-server model. Its goal is to allow a host to dynamically obtain, renew, or reuse an IP address when it joins a network, which is crucial for supporting mobile users.

The process of obtaining an IP address is known as the **DORA** message exchange:
1. **D**iscover: Client broadcasts a `DHCP Discover` message to find a server.
2. **O**ffer: Server responds with a `DHCP Offer` message containing an available IP.
3. **R**equest: Client requests the offered IP via a `DHCP Request` message.
4. **A**ck: Server confirms the allocation via a `DHCP Ack` message.

***

### 1.6 Important Details on DHCP
To make the DORA process work before a client even has an IP address, special IP values are utilized:
* **IP broadcast address:** `255.255.255.255` is used to send messages to every device on the local network.
* **Special-purpose address:** `0.0.0.0` is used by the client to denote "this host" before an IP is assigned.
* **Transaction ID (XID):** Because multiple clients might be broadcasting for IPs simultaneously, and multiple servers might offer them, the client generates a unique XID to ensure it pairs with the correct DORA exchange.

***

### 1.7 DHCP client-server scenario
In most modern local networks, the DHCP server is co-located (built-in) directly within the first-hop router.

*Visual Aid: Diagram showing a newly arriving client and a router with a built-in DHCP server.*
```text
 [Arriving Client] ------> (Network) <------ [Router w/ built-in DHCP Server]
   Needs an IP!                              IP: 223.1.2.5
```

*Visual Aid: Sequence diagram detailing the DORA message exchange.*
```text
 Client (Src IP: 0.0.0.0)                             DHCP Server (IP: 223.1.2.5)
      |                                                        |
      | ---- DHCP Discover (Broadcast Dest: 255.255.255.255) ->|
      |      XID: 654                                          |
      |                                                        |
      | <--- DHCP Offer (Broadcast Dest: 255.255.255.255) ---- |
      |      XID: 654, Offered IP: 223.1.2.4                   |
      |      Lifetime: 3600s                                   |
      |                                                        |
      | ---- DHCP Request (Broadcast Dest: 255.255.255.255) -> |
      |      XID: 654, Requesting IP: 223.1.2.4                |
      |                                                        |
      | <--- DHCP ACK (Broadcast Dest: 255.255.255.255) ------ |
      |      XID: 654, Confirmed IP: 223.1.2.4                 |
      V                                                        V
```

***

### 1.8 DHCP Address Renewal (optional for your knowledge)
DHCP assigns IP addresses on a "lease" basis. 
* At **50% lease time expiration**, the client attempts to renew by sending a direct unicast `DHCP Request` to the server.
* If the primary server is down, at **87.5% lease time expiration**, the client will broadcast a `DHCP Request` to try and find a backup server to renew the lease.

***

### 1.9 DHCP: more than IP addresses
DHCP is incredibly powerful because it provides much more than just the client's IP address. It also configures:
1. The allocated IP address.
2. The address of the **first-hop router** (the default gateway).
3. The name and IP address of the **DNS server**.
4. The **network mask** (indicating network vs. host portions).

***

### 1.10 DHCP: example
*Visual Aid: Diagram showing the encapsulation of a DHCP Request message.*
```text
 +---------------------------------------------------+
 | Ethernet Header (Dest MAC: FFFFFFFFFFFF Broadcast)|
 |   +---------------------------------------------+ |
 |   | IP Header (Src: 0.0.0.0, Dest: 255.255... ) | |
 |   |   +---------------------------------------+ | |
 |   |   | UDP Header (Ports 67 and 68)          | | |
 |   |   |   +---------------------------------+ | | |
 |   |   |   | DHCP REQUEST Message            | | | |
 |   |   |   +---------------------------------+ | | |
 |   |   +---------------------------------------+ | |
 |   +---------------------------------------------+ |
 +---------------------------------------------------+
```

*Visual Aid: Diagram showing the server formulating the DHCP ACK and forwarding it back to the client.*
```text
  [DHCP Server/Router]
          |
  1. Formulates DHCP ACK (Assigns IP: 223.1.2.4, DNS, Gateway)
  2. Encapsulates in UDP -> IP -> Ethernet
  3. Broadcasts frame onto LAN
          |
          V
  [Client Laptop] receives Ethernet frame, demultiplexes up through IP and UDP, 
  and successfully configures its network interface.
```

***

### 1.11 IP addresses: how to get a subnet address block?
To answer the second core question—how networks get address blocks—organizations request a portion of their Internet Service Provider's (ISP) allocated address space. The ISP uses CIDR to carve out smaller subsets of their large block to hand out.

*Visual Aid: Text block showing an ISP dividing a /20 block into eight /23 blocks for different organizations.*
```text
 ISP's Master Block: 200.23.16.0/20 
 
 Allocated out as 8 sub-blocks:
 Org 0: 200.23.16.0/23 (11001000 00010111 00010000 00000000)
 Org 1: 200.23.18.0/23 (11001000 00010111 00010010 00000000)
 Org 2: 200.23.20.0/23 (11001000 00010111 00010100 00000000)
 ...
 Org 7: 200.23.30.0/23 (11001000 00010111 00011110 00000000)
```

***

### 1.12 Example: UCLA IP Address Blocks
Real-world institutions own large swaths of IP blocks. 

*Visual Aid: List of IP prefixes assigned to UCLA.*
```text
 * 128.97.0.0/16
 * 131.179.0.0/16
 * 149.142.0.0/16
 * 164.67.0.0/16
 * 172.16.0.0/12
 * 192.35.210.0/24
 * 2607:F010::/32 (An IPv6 block)
```

***

### 1.13 Hierarchical addressing
CIDR allows for **route aggregation** (or summarization). Instead of an ISP telling the rest of the Internet about thousands of individual customer networks, the ISP advertises a single, large aggregated prefix (e.g., "Send me anything starting with `200.23.16.0/20`").

However, if an organization switches to a different ISP, the new ISP must advertise a **more specific route** (a longer prefix, like `/23`). Internet routers always forward traffic based on the longest matching prefix, meaning the specific `/23` advertisement will override the old `/20` advertisement.

*Visual Aid: Network architecture diagrams showing ISPs advertising aggregated routes vs. specific longest-prefix routes.*
```text
                 [Fly-By-Night ISP]
                /   Advertises: "Send me anything starting with 200.23.16.0/20"
 [Org 0: /23] -+             \
 [Org 2: /23] -+              \
 [Org 7: /23] -+               \
                                +---> [Internet]
 [Org 1: /23] -+               /      (Internet routes Org 1 traffic to ISPs-R-Us 
  (Moved ISPs!) \             /        because /23 is more specific than /20)
                 [ISPs-R-Us]
                    Advertises: "Send me anything starting with 199.31.0.0/16"
                    AND Advertises: "Also send me 200.23.18.0/23" (Specific route)
```

***

### 1.14 IP addressing: last words ...
Who manages all of these IP addresses? **ICANN** (Internet Corporation for Assigned Names and Numbers). ICANN allocates IP spaces to 5 regional registries (RRs) and manages the DNS root zones.

The major problem: In 2011, ICANN allocated the last remaining chunks of the IPv4 address space. The internet essentially ran out of 32-bit addresses. To combat this IPv4 exhaustion, two main solutions were deployed: **NAT** (Network Address Translation) as a stopgap, and **IPv6** as the permanent solution.

***

## 2.0 NAT (Network Address Translation)

### 2.1 NAT: network address translation
**NAT** allows all devices within a local network to share a *single* public IPv4 address as far as the outside world is concerned. Devices inside the network are assigned **Private IP Addresses** that are strictly locally significant and unroutable on the global Internet.

* Advantages: Drastically reduces the need for public IPs, allows you to change local IPs without notifying the outside world, allows you to change ISPs easily, and provides implicit security because internal devices cannot be directly addressed from the outside.

*Visual Aid: Diagram showing datagrams leaving a local network translating to a single public IP address.*
```text
 [Local Network 10.0.0.0/24]
 
 [Host 10.0.0.1] \
 [Host 10.0.0.2] -- [NAT Router] -----------> (Rest of Internet)
 [Host 10.0.0.3] /  |          |
                    |          |
    Inside IPs:     |          |  Outside IP replacing all source IPs:
    10.0.0.x        |          |  138.76.29.7 (with different port numbers)
```

***

### 2.2 Dedicated Space for Carrier-Grade NAT (RFC6598)
Because ISPs also ran out of addresses, they implemented Carrier-Grade NAT using the dedicated space `100.64.0.0/10`. This is used strictly for internal carrier operations and should not be routed on the public internet.

***

### 2.3 Private IP Address Spaces
If you are setting up a private network behind a NAT, you must use designated private IP ranges. 
* **IPv4 (RFC1918):** `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
* **IPv6 (RFC4193):** `fc00::/7`

***

### 2.4 NAT implementation & example
To make this sharing work, the NAT router maintains a **NAT translation table**. It maps combinations of `(Source IP, Source Port)` from the local network to `(NAT Public IP, New Port)`.
* **Outgoing datagrams:** Router replaces the local source IP and port with the NAT public IP and new port.
* **Incoming datagrams:** Router looks up the destination IP/port in the table and replaces it with the internal local IP and port.

*Visual Aid: Diagram mapping a packet from a private host through the router, detailing the exact NAT translation table.*
```text
   NAT Translation Table
   WAN Side Addr (Public) | LAN Side Addr (Private)
   -----------------------+-----------------------
   138.76.29.7, 5001      | 10.0.0.1, 3345
 
 1. Host 10.0.0.1 sends packet to web server (128.119.40.186, Port 80).
    Src: 10.0.0.1:3345 -> Dest: 128.119.40.186:80
 
 2. NAT Router transparently changes Src to 138.76.29.7:5001 and logs to table.
 
 3. Reply arrives from web server.
    Src: 128.119.40.186:80 -> Dest: 138.76.29.7:5001
 
 4. NAT Router looks up table, changes Dest to 10.0.0.1:3345 and forwards to host.
```

***

### 2.5 NAT controversy
Despite its widespread use, NAT is highly controversial among network engineers:
* **Layer Violation:** Routers are Layer 3 (Network Layer) devices. They "should" only process up to Layer 3, but NAT forces routers to read and modify Layer 4 port numbers.
* **Violates end-to-end argument:** Hosts should be able to communicate transparently. NAT prevents direct connections, creating massive headaches for peer-to-peer applications (NAT Traversal problems).
* IPv6 was supposed to solve the address shortage natively.
* Regardless of controversy, NAT is extensively used in home networks, institutional networks, and 4G/5G mobile networks today.

***

## 3.0 IPv6

### 3.1 IPv6: motivation
The primary motivation for IPv6 was the complete exhaustion of the 32-bit IPv4 address space. However, engineers took the opportunity to improve the protocol overall:
* **Processing Speed:** The IPv4 header varies in size. IPv6 uses a streamlined, fixed 40-byte header to speed up routing.
* **Flows:** Added native ability to identify and treat streams of packets (flows) uniformly.

***

### 3.2 IPv6 datagram format: 40-byte basic header
The new 40-byte header includes several updated fields:
* `priority`: Identifies the priority among datagrams in a flow.
* **`flow Label`**: Identifies datagrams in the same "flow" (e.g., a specific TCP connection or media stream).
* `next header`: Identifies the upper-layer protocol (replacing IPv4's Protocol field).

*Visual Aid: Diagram of the 40-byte IPv6 header structure.*
```text
 +---------------+-------------------+-----------------------+
 | Version (4b)  | Traffic Class(8b) |    Flow Label (20b)   |
 +---------------+-------------------+-----------------------+
 |     Payload Length (16 bits)      | Next Hdr | Hop Limit  |
 +-----------------------------------+----------+------------+
 |                                                           |
 |                  Source Address (128 bits)                |
 |                                                           |
 +-----------------------------------------------------------+
 |                                                           |
 |                Destination Address (128 bits)             |
 |                                                           |
 +-----------------------------------------------------------+
```

***

### 3.3 Other changes from IPv4
To streamline the header, several IPv4 features were removed entirely:
* **Checksum:** Removed to eliminate the need for routers to recalculate the checksum at every hop (which saves massive processing time).
* **Options:** Removed from the standard header. Options are now allowed only outside the fixed header.
* **ICMPv6:** A new version of ICMP was created to handle additional message types (like "Packet Too Big" because IPv6 routers no longer fragment packets) and manage multicast groups.

***

### 3.4 IPv6: Next header & Extension header(s)
Because options were removed from the base header, IPv6 uses **Extension headers**. 
* The `next header` field points either to the transport layer (like TCP/UDP) or to an optional extension header. 
* These headers can form a "daisy-chain" where each header points to the next.

*Visual Aid: Diagram showing the daisy-chaining of the fixed IPv6 header to subsequent extension headers.*
```text
 +---------------------+       +----------------------+      +----------------------+
 | IPv6 Fixed Header   |       | Extension Header 1   |      | Upper-Layer Header   |
 | Next Header: Ext 1  | ----> | Next Header: TCP     | ---> | (e.g., TCP Payload)  |
 +---------------------+       +----------------------+      +----------------------+
```

***

### 3.5 Summary on IPv6 Header and datagram format
In summary, IPv6 provides massive 128-bit addresses. To achieve efficiency, it removed the checksum, moved options to extension headers, and completely removed fragmentation/reassembly responsibilities from intermediate routers (fragmentation is now only handled by the end hosts).

*Visual Aid: Simplified block diagram highlighting IPv6 header fields.*
```text
 +--------------------------------------------------+
 | 32 bits wide                                     |
 | Version | Priority |       Flow Label            |
 | Payload Length     | Next Header  | Hop Limit    |
 |                Source Address                    |
 |             Destination Address                  |
 |                  Payload                         |
 +--------------------------------------------------+
```

***

### 3.6 IPv4 & IPv6 Header Comparison
*Visual Aid: Side-by-side comparison comparing kept, removed, renamed, and new fields between IPv4 and IPv6 headers.*
```text
 IPv4 Header Fields:                   IPv6 Header Fields:
 - Version (Kept)                      - Version (Kept)
 - Header Length (Removed)
 - Type of Service (Renamed) --------> - Traffic Class
 - Total Length (Modified) ----------> - Payload Length
 - Identification (Removed)            (Fragmentation removed from router)
 - Flags (Removed)                     (Fragmentation removed from router)
 - Fragment Offset (Removed)           (Fragmentation removed from router)
 - Time to Live (Renamed) -----------> - Hop Limit
 - Protocol (Renamed) ---------------> - Next Header
 - Header Checksum (Removed)           (Saves processing time)
 - Source Address (32b -> 128b) -----> - Source Address
 - Destination Address (32b -> 128b)-> - Destination Address
 - Options (Moved) ------------------> - Extension Headers
                                       - Flow Label (NEW)
```

***

### 3.7 Transition from IPv4 to IPv6
Because the Internet is vast, it is impossible to upgrade all routers to IPv6 simultaneously (there are no "flag days" where the internet turns off to upgrade). 
To allow networks to operate with mixed IPv4 and IPv6 routers, we use **Tunneling**. 

**Tunneling** is the process where an entire IPv6 datagram is encapsulated and carried as the *payload* inside a standard IPv4 datagram to traverse legacy networks.

*Visual Aid: Block diagram showing an IPv6 datagram encapsulated inside an IPv4 datagram.*
```text
 +---------------------------------------------------+
 | IPv4 Header (Src: IPv4, Dest: IPv4)               |
 |   +---------------------------------------------+ |
 |   | IPv6 Header (Src: IPv6, Dest: IPv6)         | |
 |   |   +---------------------------------------+ | |
 |   |   | TCP/UDP Payload Data                  | | |
 |   |   +---------------------------------------+ | |
 |   +---------------------------------------------+ |
 +---------------------------------------------------+
```

***

### 3.8 Tunneling and encapsulation
*Visual Aid: Logical vs. physical view network diagrams illustrating an IPv6 packet being routed through an intermediate IPv4 network via Tunneling.*
```text
 Logical View (What the IPv6 hosts see):
 [IPv6 Router A] ---------------------------------- [IPv6 Router F]
 
 Physical View (Tunneling through IPv4 network):
 [IPv6 Router A]
        | (Takes IPv6 packet, wraps it in IPv4 header)
 [IPv4 Router B]
        |
 [IPv4 Router C]  <-- This router only sees a standard IPv4 packet!
        |
 [IPv4 Router E]
        | (Receives IPv4 packet, strips IPv4 header, revealing the IPv6 packet)
 [IPv6 Router F]
```

***

### 3.9 IPv6: adoption
Adoption of IPv6 has been remarkably slow. According to data (up to 2020), roughly 30% of Google's clients access services via IPv6, and about 1/3 of US government domains are IPv6 capable. The transition has taken over 25 years because replacing core infrastructure is vastly more complex than application-level changes (like the rise of streaming or social media). 

*Visual Aid: Line graph summary tracking Google's IPv6 adoption percentage.*
```text
  IPv6 Adoption Percentage (Google)
  30% |                                      /
      |                                    /
  20% |                                  /
      |                                /
  10% |                             /
      |                         / 
   0% +------------------------------------------
       2009   2011   2013   2015   2017   2020
```

***

## 4.0 Non-highlight topic

### 4.1 What’s inside a router
*(Brief aside for context)*
A router consists of input ports, a switching fabric, and output ports. The router must manage buffers and scheduling queues to push packets from input to output efficiently. 

*Visual Aid: Conceptual representation of router switching.*
```text
 [Input Port] ---> ( Switching Fabric / "Bridge" ) ---> [Output Port]
```

***

### **Glossary of Key Terms**

* **IP address**: A 32-bit (IPv4) or 128-bit (IPv6) numerical identifier associated with a specific network interface.
* **Network interface**: The boundary/connection point between a host or router and the physical communication link.
* **Subnet**: A logical subdivision of an IP network; devices on a subnet can physically reach each other without passing through a router.
* **Subnet part / Host part**: The hierarchical structure of an IP address. The high-order bits define the network (subnet part), and the remaining low-order bits identify the specific machine (host part).
* **Subnet mask**: A bitmask (often represented by a `/x` prefix) used to determine how many bits of an IP address constitute the subnet part.
* **CIDR (Classless InterDomain Routing)**: A method of allocating IP addresses and routing IP packets that allows for arbitrary-length subnet masks, eliminating rigid class sizes.
* **DHCP (Dynamic Host Configuration Protocol)**: A client-server protocol that automatically assigns IP addresses and other network configurations (like DNS and default gateway) to devices joining a network.
* **ICANN (Internet Corporation for Assigned Names and Numbers)**: The global non-profit organization responsible for coordinating the maintenance and allocation of IP address spaces and the DNS root zone.
* **NAT (Network Address Translation)**: A method by which a router translates private, local IP addresses into a single public IP address for traffic bound for the external Internet.
* **NAT translation table**: A table maintained by a NAT router that maps local internal combinations of `(Source IP, Source Port)` to outward-facing public `(NAT IP, New Port)` combinations.
* **Private IP Address Space**: Specific blocks of IP addresses (like `10.x.x.x` or `192.168.x.x`) reserved for internal local networks. They are not routable on the public Internet.
* **IPv6 Flow Label**: A 20-bit field in the IPv6 header used to identify a sequence of packets ("flow") generated by a single application that requires special handling by intermediate routers.
* **Extension headers (IPv6)**: Optional internet-layer information placed between the fixed IPv6 header and the upper-layer protocol payload. They replace the "Options" field from IPv4 to speed up basic routing.
* **Tunneling**: A transition mechanism where a protocol (like IPv6) is encapsulated entirely as the payload within another protocol (like IPv4) so it can traverse incompatible network infrastructure.
