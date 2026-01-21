# Lecture Study Notes: Application Layer Wrap-up & Transport Layer Introduction

## 1.0 Summary on Internet DNS (Domain Name System)

### 1.1 Why and what is DNS?
The Domain Name System (DNS) is a critical component of the Internet's infrastructure, functioning as a **distributed database** responsible for translating human-readable hostnames (like `www.google.com`) into machine-readable IP addresses (like `142.250.190.46`).

*   **The Scalability Problem:** A centralized database (one single server containing all mappings) does **NOT** scale.
    *   It would represent a single point of failure.
    *   It would create massive traffic volume concentration.
    *   It would result in significant delays for distant clients.
    *   Maintenance would be impossible.
*   **The Solution:** The Internet uses a hierarchical design to scale. By adding **more hierarchies** (Root, TLD, Authoritative), the load is distributed across the globe.

### 1.2 How DNS works?
*   **Basic Concepts:** The system relies on hierarchical names (e.g., `.com`, `.edu`) and hierarchical servers.
*   **DNS Structure:**
    1.  **Root Servers:** The first step in resolving a name; they direct queries to the appropriate TLD servers.
    2.  **Top-Level Domain (TLD) Servers:** Handle domains such as `.com`, `.org`, `.net`, and country codes like `.uk` or `.cn`.
    3.  **Authoritative/Local Servers:** The final destination for the query, containing the actual IP mapping for a specific organization or ISP.
*   **Query Modes:**
    *   **Recursive queries:** The burden of resolution is placed on the contacted name server (it fetches the answer for you).
    *   **Iterative queries:** The contacted name server replies with the address of the *next* server to query ("I don't know, but ask this server").
*   **DNS Record Format:** DNS is essentially a database of Resource Records (RR). A record follows a simple format: `(name, value, type, ttl)`.
    *   This structure allows for caching (storing records locally to reduce traffic) and updating.
*   **DNS Protocol:**
    *   Utilizes a query and response format.
    *   Operates primarily over **UDP** (for speed and low overhead) but can use **TCP** (for larger transfers like zone transfers).

***

## 2.0 Internet Application: P2P File Sharing

### 2.1 Peer-to-peer (P2P) Architecture
Unlike the traditional Client-Server model where a dedicated server serves all content, P2P architecture fundamentally changes the roles of the nodes.

*   **Dual Roles:** Each peer application running on a host performs both **client and server functions**.
*   **Reciprocity:** Peers request services (downloads) from other peers and provide services (uploads) in return.
*   **Examples:** BitTorrent (file sharing), KanKan (streaming), Skype (VoIP).
*   **Key Benefit: Self Scalability:**
    *   In client-server models, adding users increases the load on the server.
    *   In P2P, new peers bring **new service capacity** (upload bandwidth and storage) alongside their new service demands.

```markdown
* Visual Aid: Architecture Comparison *

   Client-Server            Peer-to-Peer (P2P)
   _____________           ____________________
   |   Server  |           | Peer | <--> | Peer |
   |___________|              ^            ^
       ^   ^                  |            |
      /     \                 v            v
  |Client| |Client|        | Peer | <--> | Peer |

(Server is the bottleneck)   (Interconnected, distributed capacity)
```

### 2.2 P2P Scalability Analysis
Mathematically, P2P demonstrates superior scalability as the number of peers (**N**) grows large.

*   **Client-Server:** As $N$ increases, the time to distribute a file increases linearly because the server's upload bandwidth is a fixed bottleneck.
*   **P2P:** As $N$ increases, the total system upload capacity increases. The minimum distribution time remains relatively flat/low regardless of how many peers join.

```markdown
* Visual Aid: Distribution Time vs. Number of Peers (N) *

Time
 ^
 |                  /  (Client-Server: Linear Increase)
 |                /
 |              /
 |            /
 |----------/----------- (P2P: Stays Low/Flat)
 |____________________> Number of Peers (N)
```

### 2.3 BitTorrent: P2P Application for File Sharing
BitTorrent is the standard protocol for P2P file distribution.

*   **File Structure:** A file is divided into chunks, typically **256Kb** in size.
*   **Definitions:**
    *   **Tracker:** A centralized server that tracks which peers are currently participating in a specific torrent.
    *   **Torrent:** The group of peers exchanging chunks of a specific file.
*   **Operations:**
    *   **Joining:** A peer (e.g., Alice) joins with no chunks. She registers with the tracker to get a list of peers and connects to a subset of them ("neighbors").
    *   **Downloading:** Alice downloads chunks she is missing from her neighbors.
    *   **Uploading:** simultaneously, Alice uploads chunks she already has to others.
    *   **Churn:** The network is dynamic; peers may come and go at any time.
    *   **Completion:** Once Alice has the full file, she may leave (selfish) or remain to seed the file (altruistic).

```markdown
* Visual Aid: BitTorrent Topology *

      [Tracker Server]
             |
        (Connects)
             v
          [Alice] <---------> [Peer B]
             ^                   ^
             |                   |
             v                   v
          [Peer C] <---------> [Peer D]

(Alice obtains peer list from Tracker, then exchanges chunks directly with B, C, D)
```

***

## 3.0 Internet Application: Video Streaming

### 3.1 Video Streaming Context
*   **Bandwidth Dominance:** Streaming video is the major consumer of Internet bandwidth, with services like Netflix and YouTube accounting for over 80% of residential ISP traffic.
*   **The Challenge:** Scaling to reach approximately 1 billion users.
    *   A single "mega-video server" is impossible due to bandwidth constraints and network distance latency.
*   **The Solution:** A **distributed, application-level infrastructure** (CDNs).

### 3.2 What is Video?
*   **Definition:** A sequence of images displayed at a constant rate (e.g., 24 or 30 frames per second).
*   **Digital Image:** An array of pixels, where each pixel is represented by bits.
*   **Coding (Compression):** Raw video is too large. Coding uses redundancy to compress data:
    *   **Spatial Redundancy:** Redundancy *within* a single image (e.g., a large patch of blue sky can be described mathematically rather than listing every blue pixel).
    *   **Temporal Redundancy:** Redundancy *between* images (e.g., if the background doesn't move between Frame 1 and Frame 2, only send the moving object).

```markdown
* Visual Aid: Video Compression *

   Spatial Coding:                  Temporal Coding:
   [Purple Area]                    [Frame i]      [Frame i+1]
   Instead of: P,P,P,P,P...         Only send the *changes* (delta)
   Send: (Purple, Repeat 5x)        from i to i+1.
```

### 3.3 Types of Video Encoding
*   **CBR (Constant Bit Rate):** The video is encoded at a fixed rate (e.g., 1.5 Mbps). Quality may fluctuate if the scene becomes complex.
*   **VBR (Variable Bit Rate):** The encoding rate changes based on spatial/temporal complexity (higher bitrate for action scenes, lower for static scenes).
*   **Examples:** MPEG 1 (CD-ROM), MPEG 2 (DVD), MPEG 4 (Internet).

### 3.4 Streaming Stored Video
*   **The Problem:** The Internet acts as a "best-effort" network.
    *   **Bandwidth Variation:** Throughput from server to client varies over time due to congestion (in the house, access network, or core).
    *   **Jitter:** Packet loss and delay variations impact video playback smoothness.

```markdown
* Visual Aid: Streaming Scenario *

   [Video Server] ---(Internet: Variable Delay/Loss)---> [Client/Buffer]
```

### 3.5 DASH: Streaming Video over the Internet
**DASH** stands for **D**ynamic, **A**daptive **S**treaming over **H**TTP. It shifts the intelligence from the server to the client.

*   **Server Role (Simple):**
    *   Divides the video into small chunks.
    *   Stores each chunk encoded at **multiple different bit rates** (low, medium, high quality).
    *   Provides a **Manifest file**: A structured file containing URLs for the different chunks and qualities.
*   **Client Role (Intelligent):**
    *   Periodically measures the available server-to-client bandwidth.
    *   Consults the manifest.
    *   **Adaptive Requesting:** Requests one chunk at a time, selecting the highest quality possible that current bandwidth can sustain without buffering.
*   **Client Intelligence Decisions:**
    *   **When:** When to request chunks (to prevent buffer starvation or overflow).
    *   **What:** What encoding rate to request (based on bandwidth).
    *   **Where:** Which server to request from (if multiple are available).

***

## 4.0 Content Distribution Networks (CDNs)

### 4.1 Challenge and Solutions
*   **Challenge:** How to stream popular content (like a new movie) to millions of simultaneous users efficiently.
*   **Naïve Solution:** A single massive data center.
    *   **Failure:** It creates a single point of failure, massive network congestion at the source, and long/slow paths for distant clients. It simply **doesn't scale**.
*   **Real-World Solution (CDN):** Store and serve multiple copies of the content at geographically distributed sites.
    *   **Enter Deep:** Push CDN servers deep into access networks (ISPs) to get as close to the user as possible (Strategy used by Akamai).
    *   **Bring Home:** Deploy large clusters in fewer, key locations (Points of Presence - POPs) near access networks (Strategy used by Limelight).

### 4.2 How CDNs Work
*   The CDN copies content (e.g., "MadMen") to its distributed nodes.
*   When a subscriber requests content, the system determines the "best" node (usually geographically closest or least congested) and directs the user there.

```markdown
* Visual Aid: CDN Architecture *

                     [Origin Server (Netflix)]
                            | (Pushes copies)
       _____________________|_____________________
      |                     |                     |
 [CDN Node A (USA)]   [CDN Node B (EU)]   [CDN Node C (Asia)]
      |
      | (Manifest directs user here)
   [User]
```

### 4.3 Finding the Best CDN Server via DNS
The "magic" of connecting a user to the right CDN server happens via DNS redirection using `CNAME` records.

**The Process:**
1.  **User Request:** Client visits `netcinema.com` and requests a video URL.
2.  **Local DNS:** Client's OS asks Local DNS to resolve the URL.
3.  **Redirection:** `netcinema.com`'s DNS does *not* return an IP. Instead, it returns a **CNAME** (alias) pointing to the CDN's domain (e.g., `kingcdn.com`).
4.  **CDN Resolution:** The Local DNS queries the CDN's authoritative DNS.
5.  **Selection:** The CDN DNS analyzes the request (source IP) to determine the best server and returns that specific IP.
6.  **Streaming:** Client streams directly from that CDN IP via HTTP.

```markdown
* Visual Aid: DNS Redirection Flow *

User -> Local DNS -> NetCinema DNS
                          |
                          v (Returns CNAME: KingCDN.com)
                     Local DNS -> KingCDN DNS
                                      |
                                      v (Returns IP of Best Server)
User <-------------------------> KingCDN Server (Stream Video)
```

### 4.4 Case Study: Netflix
*   **Hybrid Cloud/CDN:** Netflix uses **Amazon Cloud** for website logic, registration, and content uploading/transcoding.
*   **Distribution:** Processed videos are pushed from Amazon Cloud to private CDN servers.
*   **Manifest:** When a user clicks "Play," the Amazon cloud returns a manifest file containing URLs specific to that user and video.
*   **DASH:** The client selects the best CDN server and begins DASH streaming.

***

## 5.0 Chapter 3: Transport Layer

### 5.1 Introduction and Roadmap
*   **Scope:** The Transport Layer serves as the bridge between the Application layer and the Network layer.
*   **Protocol Stack Hierarchy:**
    *   **Application:** (HTTP, SMTP) - Reliable/unreliable transport built on...
    *   **Transport:** (TCP, UDP) - Provides process-to-process communication. Built on...
    *   **Network:** (IP) - Best-effort global packet delivery. Built on...
    *   **Link:** Data transfer between neighboring network elements.
    *   **Physical:** Bits on the wire.
*   **Key Topics:** Multiplexing/Demultiplexing, UDP (connectionless), Reliable Data Transfer, TCP (connection-oriented), Congestion Control.

### 5.2 Transport-Layer Services
*   **Logical Communication:** Transport protocols provide **logical communication** between application processes running on different hosts.
    *   From the application's perspective, it looks like the hosts are directly connected.
*   **End-System Only:** Transport protocols run on the end hosts (computers/phones), **NOT** on network routers. Routers only act on Network layer (IP) fields.
*   **Sender Role:** Breaks application messages into **segments** and passes them to the network layer.
*   **Receiver Role:** Reassembles segments into messages and passes them to the application layer.

```markdown
* Visual Aid: Logical Communication *

   [Host A]                                      [Host B]
   | App  |                                      | App  |
   | Trans| <--- (Logical Connection) ---------> | Trans|
   | Net  |       (Jumps over routers)           | Net  |
      |                                             |
      v                                             v
   [Router] ----------------------------------- [Router]
```

### 5.3 Transport Layer in Action
*   **Sender Side:**
    1.  Receives an application-layer message.
    2.  Encapsulates it with transport header fields (creating a **segment**).
    3.  Passes the segment to the Network Layer (IP).
*   **Receiver Side:**
    1.  Receives the segment from the IP layer.
    2.  Checks header values (checksum, ports).
    3.  Extracts the application message.
    4.  **Demultiplexes** the message to the correct application socket.

### 5.4 Example: Web Application (HTTP)
*   Encapsulation involves wrapping data: `[ Ethernet Header [ IP Header [ TCP Header [ HTTP Data ] ] ] ]`.
*   A single server (e.g., Apache) can handle multiple clients simultaneously. Each client connection is handled by a distinct process or thread, managed via the transport layer.

***

## 6.0 Multiplexing and Demultiplexing

### 6.1 Definitions
*   **Multiplexing (Sender Side):** The job of gathering data chunks from different application sockets, encapsulating each with header information (source/dest ports), and feeding them into the network layer.
*   **Demultiplexing (Receiver Side):** The job of delivering the data in a received transport-layer segment to the **correct application socket**.

```markdown
* Visual Aid: Mux/Demux *

Sender:
[Process 1]  [Process 2]
      \          /
       \        / (Multiplexing)
     [Transport Layer]
            |
            v
      (Network Link)
            |
            v
     [Transport Layer]
       /        \ (Demultiplexing)
      /          \
[Process 1]  [Process 2]
```

### 6.2 How Demultiplexing Works
*   The receiver uses **IP addresses** and **Port numbers** to direct segments.
*   The receiving host receives an IP datagram.
    *   The **IP Header** contains Source/Dest IP addresses.
    *   The **Transport Segment** inside contains **Source Port** and **Destination Port**.

### 6.3 Two Types of Demultiplexing
1.  **Connectionless demultiplexing** (Used by UDP).
2.  **Connection-oriented demultiplexing** (Used by TCP).

### 6.4 Connectionless Demultiplexing (UDP)
*   **Socket Creation:** A UDP socket is bound to a specific local port (e.g., `DatagramSocket(12534)`).
*   **Behavior:**
    *   When a UDP segment arrives, the host checks the **destination port number**.
    *   It directs the segment to the socket with that port number.
    *   **Crucial Detail:** IP datagrams with different source IPs and/or source ports, but the **same destination IP and destination port**, are directed to the **same socket**.
    *   The application process creates a response by extracting the source address from the received packet.

```markdown
* Visual Aid: UDP Demultiplexing *

   [Client A]                    [Server]
   Src: A, 9999                  Socket Bound to Port 6428
   Dst: S, 6428  ------------->  [ Application ]
                                      ^
   [Client B]                         |
   Src: B, 8888                       |
   Dst: S, 6428  ---------------------|

   (Both go to the SAME socket because Dest Port matches)
```

### 6.5 Connection-oriented Demultiplexing (TCP)
*   **4-Tuple Identification:** A TCP socket is identified by four values:
    1.  Source IP address
    2.  Source Port number
    3.  Destination IP address
    4.  Destination Port number
*   **Behavior:** The receiver uses **all four values** to demultiplex the segment.
*   **Implication:** Two segments with the same destination IP and port (e.g., Port 80 for Web) but different *source* IPs will be directed to **different sockets**. This allows a web server to maintain distinct conversations with Client A and Client B simultaneously.

```markdown
* Visual Aid: TCP Demultiplexing *

   [Client A]                    [Server]
   (Src A, Dst S:80) --------->  [Socket 1 (Process A)]

   [Client B]                    [Server]
   (Src B, Dst S:80) --------->  [Socket 2 (Process B)]

   (Different sockets because the Source IP differs)
```

***

## 7.0 UDP: User Datagram Protocol

### 7.1 Overview and Characteristics
*   UDP is the "no frills," bare-bones transport protocol.
*   **Best Effort Service:** Segments may be lost or delivered out of order. There are no guarantees.
*   **Connectionless:** No handshaking (like TCP's SYN/ACK) takes place. This means there is no connection establishment delay (RTT).
*   **State:** The protocol maintains no connection state at the sender or receiver.
*   **Speed:** Without congestion control, UDP can blast data as fast as the application and network allow.

### 7.2 Usage
*   **Streaming Multimedia:** Can tolerate loss but is sensitive to rate/delay (e.g., VoIP, Gaming).
*   **DNS:** Needs speed; a lost query can simply be re-sent.
*   **SNMP:** Network management.
*   **HTTP/3:** Uses UDP to implement reliability and congestion control at the *application* layer (QUIC) to avoid TCP's head-of-line blocking.

### 7.3 UDP in Action (SNMP Example)
*   **Sender:** Creates a UDP segment containing the SNMP message.
*   **Receiver:** Checks the UDP checksum. If valid, extracts the message and demultiplexes it to the application via the socket.
*   **Header Size:** Very lightweight—only **8 bytes**.
    *   Source Port (2 bytes)
    *   Destination Port (2 bytes)
    *   Length (2 bytes)
    *   Checksum (2 bytes)

### 7.4 UDP Checksum
*   **Goal:** To detect errors (e.g., flipped bits) in the transmitted segment.
*   **Algorithm:**
    *   **Sender:** Treats segment contents as a sequence of 16-bit integers. Computes the **One's Complement Sum** of these integers. Puts the result in the Checksum field.
    *   **Receiver:** Adds the received data (as 16-bit integers) and the checksum.
        *   If result is all `1`s: No error detected.
        *   If result contains `0`: Error detected.
*   **Weakness:** It is possible for two different data sets to yield the same checksum (collisions), so it is a weak form of protection.

```markdown
* Visual Aid: Internet Checksum Math *

  1. Add two 16-bit integers:
     1 1 1 0 0 1 1 0 0 1 1 0 0 1 1 0
   + 1 1 0 1 0 1 0 1 0 1 0 1 0 1 0 1
   ---------------------------------
   1 1 0 1 1 1 0 1 1 1 0 1 1 1 0 1 1  (Wraparound carried bit)

  2. One's Complement (Flip bits of sum) = Checksum
```

***

## 8.0 TCP Protocol Overview

### 8.1 Key Characteristics
*   **Point-to-point:** One sender, one receiver (unicast only).
*   **Reliable, in-order byte stream:** TCP guarantees that data arrives and is assembled in the correct order. It does not preserve message boundaries (it is a stream of bytes).
*   **Connection-oriented:** Requires a handshaking phase (exchange of control messages) to initialize state before data transfer begins.
*   **Flow Controlled:** Sender will not overwhelm the receiver's buffer.
*   **Congestion Controlled:** Sender will throttle back to avoid overwhelming the network core (the Internet).
*   **Full Duplex:** Data can flow bi-directionally in the same connection.
*   **MSS:** Maximum Segment Size (limits the size of data in one segment).
