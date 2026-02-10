# Lecture Study Notes: CS118 Discussion (Application & Transport Layers)

## 1.0 Discussion 1A: Application Layer Review

### 1.1 Agenda
*   **Project 1 Push:** Focus on current project deadlines.
*   **Chapter 2 Review:** Finalizing Application Layer topics (Cookies, DNS, P2P, CDNs).
*   **Chapter 3 Introduction:** Beginning the transition to the Transport Layer.

***

### 1.2 Cookie
The HTTP protocol is fundamentally **stateless**, meaning the server does not remember past requests from the same client. To support complex web applications (like shopping carts or user logins), we need a mechanism to maintain state.

*   **Purpose:** To introduce **statefulness** into HTTP transactions.
*   **Components:** The cookie framework relies on four specific components working in unison:
    1.  **Response Header:** The server includes a `Set-Cookie:` header line in its HTTP response message.
    2.  **Request Header:** The client includes a `Cookie:` header line in subsequent HTTP request messages.
    3.  **Cookie File:** The user's browser manages a file stored on the local host to keep these cookies.
    4.  **Backend Database:** The website maintains a database to map cookie IDs to specific user data (e.g., cart contents, login status).

```text
* Visual Aid: Cookie Concept *

[ Client Browser ]                      [ Web Server ]
       |                                      |
       |  (3) Stores Cookie File              | (4) Database Entry
       |   [ ID: 12345 ]                      |    [ ID 12345 = User A ]
       |                                      |
```

***

### 1.3 Cookie: make HTTP stateful
The interaction process ensures that a user is recognized across multiple requests.

*   **Process Flow:**
    1.  **Login:** User sends a standard HTTP Request (usually `POST`) with credentials.
    2.  **Creation:** The server verifies credentials and creates a unique **Session ID**.
    3.  **Response:** The server responds with `Set-Cookie: SESSIONID=66C5...`.
    4.  **Storage:** The client browser saves this ID in its local cookie storage.
    5.  **Subsequent Access:** For every future request to this domain, the browser automatically appends the header `Cookie: SESSIONID=66C5...`.
    6.  **Lookup:** The server reads the ID from the header and queries its database to restore the user's state.

```text
* Visual Aid: HTTP Statefulness with Cookies *

HTTP Client                                   HTTP Server
    |                                              |
    | [1] POST (Login data)                        |
    |--------------------------------------------->|
    |                                              | [2] Create Session ID
    |                                              | [3] Store in Database
    | [4] HTTP Response                            |
    |     Set-Cookie: SESSIONID=1678               |
    |<---------------------------------------------|
    |                                              |
    | [5] Client stores ID: 1678                   |
    |                                              |
    | [6] Later Request                            |
    |     Cookie: SESSIONID=1678                   |
    |--------------------------------------------->|
    |                                              | [7] Server looks up ID 1678
    |                                              |     to identify user.
```

***

### 1.4 Cookie: operations
Cookies allow a specific client to maintain simultaneous, distinct states with different servers.

*   **Example Scenario:** A client browser visits eBay and Amazon.
    *   **eBay Server:** Assigns ID `8734`. The browser stores this specific to `ebay.com`.
    *   **Amazon Server:** Assigns ID `1678`. The browser stores this specific to `amazon.com`.
*   **Persistence:** One week later, if the user returns to Amazon, the browser checks its file, finds the ID `1678` associated with Amazon, and sends it. Amazon retrieves the user's preferences from the backend database.

```text
* Visual Aid: Cookie File Interaction *

[ Client Browser ]
+------------------+
| Cookie File      |
| ebay.com: 8734   | ---------> [ eBay Server ] ---> [ eBay DB ]
| amazon.com: 1678 | ---------> [ Amazon Server ] -> [ Amazon DB ]
+------------------+
       ^
       | (Manages specific IDs for specific domains)
```

***

### 1.5 HTTP conditional GET
Caching improves performance, but clients need a way to ensure their cached copy is up-to-date without re-downloading the whole file if it hasn't changed.

*   **Initial Request Headers:**
    *   `GET /sample.html HTTP/1.1`
*   **Initial Response Headers (Key for Caching):**
    *   **`Cache-Control`**: Specifies the maximum age (in seconds) the object is considered fresh (e.g., `max-age=21600`).
    *   **`Last-Modified`**: The exact date/time the object changed on the server.
    *   **`Etag`**: A unique hash string representing the file content version (e.g., `"4135cda4"`).

***

### 1.6 HTTP conditional GET (Mechanism)
When the cache expires or the client wants to verify freshness, it issues a **Conditional GET**.

*   **The Request:** The client sends the same GET request but adds conditions based on the previous response:
    *   `If-Modified-Since: [Date from Last-Modified]`
    *   `If-None-Match: [Hash from Etag]`
*   **The Response (If Valid):** If the file on the server has *not* changed:
    *   The server responds with **`HTTP/1.x 304 Not Modified`**.
    *   **Crucially:** The response body is **empty**. This saves bandwidth.
    *   The server may send updated headers (e.g., a new expiration time).

```text
* Visual Aid: Conditional GET Comparison *

      [Initial Request]                    [Conditional Request]

Client: GET /file.html               Client: GET /file.html
                                             If-None-Match: "4135cda4"

Server: HTTP 200 OK                  Server: HTTP 304 Not Modified
        Etag: "4135cda4"                     (No Data Body)
        [Data Body Included]
```

***

### 1.7 Server Support for Conditional GET
*   **Question:** What happens if the server software does not support `If-Modified-Since` or `If-None-Match`?
*   **Answer:** The server ignores the conditional headers and treats it as a standard request, replying with **`200 OK`** and sending the full file again.

***

### 1.8 DNS (Introduction)
The Domain Name System (DNS) translates human-readable hostnames (like `www.ucla.edu`) into IP addresses.
*   **Key Design Questions:**
    *   What transport protocol does it use?
    *   How does it scale to the entire internet?
    *   How do queries traverse the hierarchy?

***

### 1.9 DNS (Answers)
*   **Transport Protocol:** DNS primarily uses **UDP** (Port 53) for standard queries due to its speed and low overhead.
*   **Scalability:** Achieved through a **distributed, hierarchical database**. No single server holds all records.
*   **Query Types:**
    *   **Iterative:** The local server asks the Root, which refers it to the TLD, which refers it to the Authoritative server. The local server does the work.
    *   **Recursive:** The burden of resolution is passed completely to the next server in the chain.
*   **DNS Resolver:** Usually the Local DNS Server provided by the ISP or organization. Its primary purpose is to cache records to **reduce latency** and network traffic.

```text
* Visual Aid: DNS Iterative Query Structure *

[Local DNS Server] --(1) Query--> [Root DNS Server]
       ^           <-(2) Refer---
       |
       +-------------(3) Query--> [.com DNS Server]
       ^           <-(4) Refer---
       |
       +-------------(5) Query--> [Authoritative DNS Server]
                   <-(6) Answer--
```

***

### 1.10 DNS protocol: exercise
**Scenario A:** Host A queries `www.ucla.edu`.
*   **Assumptions:** Cache is empty. No CDN involved.
*   **Query Path:**
    1.  Host A -> Local DNS
    2.  Local DNS -> Root Server (Returns `.edu` NS)
    3.  Local DNS -> `.edu` TLD Server (Returns `ucla.edu` NS)
    4.  Local DNS -> `ucla.edu` Authoritative Server (Returns IP)
*   **Result:** The resolver issues **3 queries** (Root, TLD, Authoritative).

**Scenario B:** After Host A, Host B queries `www.mit.edu`.
*   **Optimization:** The Local DNS cache now contains the **NS record for `.edu`** from the previous query.
*   **Query Path:**
    1.  Host B -> Local DNS
    2.  Local DNS -> `.edu` TLD Server (Skips Root)
    3.  Local DNS -> `mit.edu` Authoritative Server
*   **Result:** The resolver issues **2 queries**.

***

### 1.11 DNS protocol: exercise (Cache)
After querying `www.ucla.edu`, the Local DNS cache is populated.
*   **Cache Content:**
    1.  NS record for `.edu` TLD servers.
    2.  NS record for `ucla.edu` authoritative servers.
    3.  A record (IP address) for `www.ucla.edu`.
    4.  A records for the nameservers themselves (Glue records).

```text
* Visual Aid: Cache State Table *

| Name         | Type | Value             |
|--------------|------|-------------------|
| edu          | NS   | a.edu-servers.net |
| ucla.edu     | NS   | ns1.dns.ucla.edu  |
| www.ucla.edu | A    | 128.97.x.x        |
```

***

### 1.12 Experiment: DNS query
Using the command `dig google.com` provides insight into DNS packet structure.
*   **Header:** Shows status (e.g., `NOERROR`) and flags.
*   **Question:** `google.com. IN A` (Asking for the IPv4 address).
*   **Answer:** Returns the `A` record (e.g., `172.217.2.14`).
*   **Authority:** Lists the Name Servers (`NS` records) responsible for the domain (e.g., `ns1.google.com`).
*   **Additional:** Provides IP addresses for those Name Servers to prevent extra lookups.

```text
* Visual Aid: dig Output Structure *

;; ->>HEADER<<- opcode: QUERY, status: NOERROR
;; QUESTION SECTION:
;google.com.            IN      A

;; ANSWER SECTION:
google.com.     76      IN      A       172.217.2.14

;; AUTHORITY SECTION:
google.com.     85950   IN      NS      ns1.google.com.
```

***

### 1.13 Application Layer: CDN
*   **Definition:** **Content Distribution Network**.
*   **Architecture:** A network of globally distributed web servers.
*   **Function:** Copies of heavy content (images, videos) are stored on these edge servers close to users to minimize travel time and server load.

***

### 1.14 CDN: Example (Part 1)
When `dig` is used to query a site using a CDN (e.g., querying the authoritative server for `www.ucla.edu`):
*   **Observation:** The Answer Section does **not** return an IP address immediately.
*   **Result:** It returns a **CNAME** (Canonical Name) record.
    *   Example: `www.ucla.edu` maps to `d1zev4mn1zpfbc.cloudfront.net`.
*   **Implication:** This redirects the client to ask the CDN provider for the IP.

***

### 1.15 CDN: Example (Part 2)
The client (or resolver) then queries the CNAME provided by the CDN.
*   **Command:** `dig d1zev4mn1zpfbc.cloudfront.net`
*   **Result:** The CDN's DNS server returns multiple **A records** (IP addresses).
*   **Why Multiple IPs?** These represent different physical CDN servers. The CDN's DNS logic selects the optimal IPs based on the user's location and server load.

***

### 1.16 Application Layer: P2P
**Peer-to-Peer** architecture differs from Client-Server.
*   **Characteristics:**
    *   No always-on central server.
    *   Arbitrary end systems (peers) communicate directly.
    *   Peers are intermittently connected and change IP addresses.
*   **Example:** BitTorrent (file sharing).

***

### 1.17 BitTorrent Operations
*   **Torrent:** A collection of all peers participating in the distribution of a specific file.
*   **Tracker:** A centralized node that tracks which peers are currently in the torrent.
*   **Lifecycle:**
    1.  **Join:** A new peer (Alice) joins, having no chunks.
    2.  **Register:** Alice registers with the **Tracker** and receives a list of active peers.
    3.  **Connect:** Alice attempts to establish TCP connections with a subset of peers ("Neighbors").
    4.  **Exchange:** While downloading chunks she lacks, she uploads chunks she has.
    5.  **Churn:** Peers may leave or join at any time.
    6.  **Completion:** Upon finishing, Alice may leave (**selfish**) or stay to seed (**altruistic**).

```text
* Visual Aid: BitTorrent Mesh *

      [ Tracker ]
           |
       (Peers List)
           |
      [ Peer A ] <-----> [ Peer B ]
           ^ \             / ^
           |   \         /   |
           |     \     /     |
      [ Peer C ] <-----> [ Peer D ]
```

***

### 1.18 BitTorrent: requesting, sending file chunks
BitTorrent uses specific algorithms to maximize efficiency and incentive participation.

*   **Requesting Strategy (Rarest First):**
    *   Alice periodically asks neighbors for their list of chunks.
    *   She requests chunks that are **rarest** among her neighbors first. This ensures rare chunks propagate quickly to prevent them from disappearing if a seed leaves.
*   **Sending Strategy (Tit-for-Tat):**
    *   Alice sends chunks to the top 4 peers who are sending data to *her* at the highest rate.
    *   **Choked:** All other peers are "choked" (receive no data).
    *   **Optimistic Unchoking:** Every 30 seconds, Alice randomly selects one choked peer to send data to. This helps discover better connections.

---

## 2.0 Transport Layer Introduction

### 2.1 Transport Layer V.S. Network Layer
*   **Network Layer:** Provides logical communication between **hosts**.
    *   Addressing: Uses **IP Address**.
*   **Transport Layer:** Provides logical communication between **processes** (apps) running on hosts.
    *   Addressing: Uses **IP Address + Port Number**.

***

### 2.2 Multiplexing and De-multiplexing
*   **Multiplexing (Sender):** The process of gathering data chunks from different sockets, encapsulating them with transport headers, and passing them to the network layer.
*   **De-multiplexing (Receiver):** Using header information to deliver the received segment to the correct socket.
*   **Identification (5-Tuple):** A TCP connection is uniquely identified by:
    1.  Source IP
    2.  Source Port
    3.  Destination IP
    4.  Destination Port
    5.  Protocol (TCP/UDP)
*   **Tool:** `lsof -i` can be used to list open sockets and their tuples.
*   **Port Sharing:** TCP and UDP are distinct protocols; therefore, they can use the same port number simultaneously (e.g., DNS uses Port 53 for both TCP and UDP).

***

### 2.3 UDP
**User Datagram Protocol** is a lightweight transport protocol.
*   **Characteristics:**
    *   **Connectionless:** No handshaking delay.
    *   **Stateless:** The server maintains no connection state.
    *   **Low Overhead:** The packet header is only 8 bytes.
*   **Checksum:** Used for error detection. It is calculated over the **Pseudo Header** (IP details) + **UDP Header** + **Data**.

```text
* Visual Aid: UDP Header Format *

 0      7 8     15 16    23 24    31
+--------+--------+--------+--------+
|     Source      |   Destination   |
|      Port       |      Port       |
+--------+--------+--------+--------+
|                 |                 |
|     Length      |    Checksum     |
+--------+--------+--------+--------+
|                                   |
|          Data Octets ...          |
+-----------------------------------+
```

***

### 2.4 Principles of Reliable Data Transfer
Since the underlying IP network is unreliable, the Transport layer (specifically TCP) must handle potential issues:

1.  **Bit Errors:** Corrupted data.
    *   *Solution:* **Checksums** for detection.
2.  **Packet Loss:** Data never arrives.
    *   *Solution:* **Retransmission** (Automatic Repeat Request).
3.  **Unknown Status:** Sender doesn't know if receiver got data.
    *   *Solution:* **Feedback** (ACKs - Acknowledgments).
4.  **Duplicate Packets:** Retransmissions causing doubles.
    *   *Solution:* **Sequence Numbers**.
5.  **Lost Feedback:** ACK is lost.
    *   *Solution:* **Timers** (Timeout mechanisms).

---

## 3.0 Review and Calculation Exercises

### 3.1 Review: SMTP
*   **Problem:** SMTP is a legacy 7-bit ASCII protocol. It cannot natively transmit binary files (images, executables).
*   **Specific Constraint:** The sequence `CRLF.CRLF` is used to mark the end of an email body. If a binary file accidentally contains this bit pattern, the transmission would terminate prematurely.
*   **Solution:** Convert binary data into text using **Base64** encoding.
*   **Overhead:** Base64 expands the file size by a factor of approximately **4/3** (33% increase).

***

### 3.2 Question for review (Link Properties)
**Scenario:**
*   Link Length: 10 meters
*   Transmission Rate ($R$): 150 bits/sec
*   Data Packet Size ($L_{data}$): 100,000 bits
*   Control Packet Size ($L_{ctrl}$): 200 bits
*   Propagation Speed ($s$): $3 \times 10^8$ m/s

**Calculations:**
1.  **Propagation Delay ($T_p$):** Time to travel the wire.
    $$T_p = \frac{\text{Distance}}{s} = \frac{10}{3 \times 10^8} \approx 3.33 \times 10^{-8} \text{ sec}$$ (negligible)
2.  **Transmission Delay (Data) ($T_x$):** Time to push bits onto the link.
    $$T_x = \frac{L_{data}}{R} = \frac{100,000}{150} = 666.6 \text{ seconds}$$
3.  **Transmission Delay (Control):**
    $$T_x = \frac{200}{150} = 1.33 \text{ seconds}$$

***

### 3.3 Question for review (RTT Calculation)
**Scenario:** HTTP interaction to fetch a base HTML page containing **10 references objects**.
*   Assumption: Transmission delay is ignored. Focus on **RTT** (Round Trip Time). Handshake takes 1 RTT. Request/Response takes 1 RTT.

**Cases:**
1.  **Non-persistent HTTP:** Each object requires a new connection (Handshake + Request).
    *   Base HTML: 2 RTTs.
    *   10 Objects: $10 \times 2$ RTTs.
    *   **Total:** $2 + 20 = 22$ RTTs.
2.  **Non-persistent HTTP + 10 Parallel Connections:**
    *   Base HTML: 2 RTTs.
    *   10 Objects: Fetched simultaneously in parallel connections ($1 \times 2$ RTTs).
    *   **Total:** $2 + 2 = 4$ RTTs.
3.  **Persistent HTTP:** Reuses connection. Handshake only once.
    *   Connection Setup + Base HTML: 2 RTTs.
    *   10 Objects: Serial requests ($10 \times 1$ RTT).
    *   **Total:** $2 + 10 = 12$ RTTs.
4.  **Persistent HTTP + Pipelining:**
    *   Connection Setup + Base HTML: 2 RTTs.
    *   10 Objects: Requests sent back-to-back ($1$ effective RTT for the batch).
    *   **Total:** $3$ RTTs.

***

### 3.4 DNS Dig Exercise (Part 1)
Analyze `dig google.com A` output:
*   (a) **IP Address:** Found in the ANSWER section (e.g., `172.217.4.142`).
*   (b) **1 Minute Later:** The **TTL (Time To Live)** value in the ANSWER section will have decreased by 60 seconds. The IP remains the same.

***

### 3.5 DNS Dig Exercise (Part 2)
*   (c) **Contacting Google NS again:** The Local DNS will cache the **A record** for the duration of its TTL (e.g., 239 seconds). It will not contact Google's nameservers again until `Current Time + 239 seconds`.
*   (d) **Contacting .com NS again:** The Local DNS caches the **NS record** for `google.com` (referral from `.com` TLD) for a much longer TTL (e.g., 12412 seconds). It won't contact `.com` servers until this expires.

***

### 3.6 Caching example: Performance Analysis
**Scenario:**
*   Link Speed: 1.54 Mbps (Access Link).
*   Request Rate: 15 requests/sec.
*   Object Size: 100k bits.
*   Traffic Intensity = $\frac{15 \times 100k}{1.54 \times 10^6} = \frac{1.5}{1.54} \approx 0.97 \ (97\%-99\%)$.

**Consequences (No Cache):**
*   Because utilization is approaching 1.0 (100%), queueing delay explodes asymptotically.
*   **Result:** The total delay becomes excessively high (minutes), rendering the network unusable.

```text
* Visual Aid: Network Topology *

[Inst. LAN] --(1.54 Mbps Access Link)--> [Public Internet] -> [Origin Servers]
    |
(Utilization ~99% -> Bottleneck)
```

***

### 3.7 Caching example: Solution 1 (Fatter Link)
*   **Action:** Upgrade Access Link from 1.54 Mbps to 154 Mbps.
*   **Result:** Utilization drops to $\approx 0.01$ (1%). Delay becomes negligible (milliseconds).
*   **Drawback:** Extremely expensive hardware and ISP costs.

***

### 3.8 Caching example: Solution 2 (Local Cache)
*   **Action:** Install a Web Cache (Proxy) in the institutional LAN.
*   **Assumption:** Cache Hit Rate is 0.4 (40%).
*   **Calculations:**
    *   40% of requests stay in LAN (fast).
    *   60% of requests go to Access Link.
    *   New Data Rate on Link: $0.6 \times 1.5 \text{ Mbps} = 0.9 \text{ Mbps}$.
    *   New Utilization: $\frac{0.9}{1.54} = 0.58$ (58%).
*   **Result:** 58% utilization is healthy; delays are low.
*   **Total Delay:** $0.6 \times (\text{Internet Delay}) + 0.4 \times (\text{LAN Delay}) \approx 1.2 \text{ seconds}$.
*   **Verdict:** Cheap and effective.

---

## 4.0 Discussion B: Transport Layer Deep Dive

### 4.1 The Plan
*   Logistics review.
*   Deep dive into Transport Layer mechanisms (headers, reliability, congestion).
*   Addressing Homework and Project questions.

### 4.2 Logistics
*   Attendance counts toward participation.
*   Requests for North Campus location noted.

### 4.3 DNS (Practical)
*   **Process:** DNS maps names to records.
*   **Security Chain:** Modern web browsing relies on a chain of trust:
    1.  **Certificates:** Pre-installed public keys.
    2.  **DNS:** Resolve IP.
    3.  **Connection:** Connect to IP.
    4.  **Verification:** Verify the server's certificate matches the DNS name.
    5.  **Encryption:** Establish secure key.
*   **DNSSEC:** Security extensions to prevent DNS spoofing (ensuring the IP returned is authentic).

### 4.4 DNS Issues
*   DNS is a single point of failure. If DNS fails, users cannot find websites even if the web servers are running perfectly.
*   Recent historical outages (e.g., Facebook) were often linked to DNS or BGP configuration errors making domains "disappear."

### 4.5 Multiplexing Review
*   **Connection-Oriented (TCP):** Demultiplexing uses the full **5-tuple**. A server can distinguish between two different clients sending to port 80 based on their Source IPs/Ports.
*   **Connection-Less (UDP):** Demultiplexing uses only **Destination IP and Destination Port**. Packets from different users to the same UDP port end up in the same socket buffer.

### 4.6 Headers
*   **UDP Header:** Simple. Source Port, Dest Port, Length, Checksum.
*   **TCP Header:** Complex. Includes reliability fields.
    *   **Sequence Number / Ack Number:** For ordering and reliability.
    *   **Flags:** `SYN` (Start), `FIN` (End), `ACK`, `RST` (Reset), `PSH` (Push).
    *   **Window:** Flow control.
    *   **Urgent Pointer:** Rarely used.

```text
* Visual Aid: TCP Header Format (Simplified) *

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Offset | Rsrvd |  Flags  |             Window                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 4.7 Reliable Data Transfer Mechanisms
*   **Stop-and-Wait:** Send 1 packet, wait for ACK. Inefficient.
*   **Pipelining (Windowing):** Send `W` packets without waiting.
    *   **Go-Back-N:** If packet $N$ is lost, retransmit $N$ and all subsequent packets. Receiver discards out-of-order packets.
    *   **Selective Repeat:** Retransmit only the lost packet. Receiver buffers out-of-order packets.
*   **ACK Semantics:** In TCP, an ACK indicates the **next expected sequence number**.
    *   *Example:* If receiver gets byte 0-99, it sends `ACK 100`.
    *   **Fast Retransmit:** If sender receives 3 duplicate ACKs for the same packet (e.g., ACK 100, ACK 100, ACK 100), it infers packet 100 is lost and retransmits immediately without waiting for a timeout.
*   **Byte-Stream:** TCP numbers **bytes**, not packets.

### 4.8 TCP Handshake & Congestion Control
*   **3-Way Handshake:**
    1.  **SYN:** Client sends random sequence number $x$.
    2.  **SYN-ACK:** Server ACKs $x+1$, sends random sequence number $y$.
    3.  **ACK:** Client ACKs $y+1$. Connection established.
*   **Congestion Control:** Flow control protects the receiver; Congestion control protects the **network**.
*   **The Variable:** `cwnd` (Congestion Window). Sender limits unacknowledged data to $\min(\text{rwnd}, \text{cwnd})$.
*   **ECN (Explicit Congestion Notification):** A modern extension where routers mark IP packets (CE flag) when queues are filling, rather than dropping them. The receiver sees this and echoes it (ECE flag) to the sender to slow down.

### 4.9 Congestion Control Algorithms
*   **Slow Start:** Start `cwnd` at 1 MSS. Double `cwnd` every RTT (exponential growth) until a loss occurs or `ssthresh` is reached.
*   **AIMD (Congestion Avoidance):** Additive Increase (add 1 MSS per RTT), Multiplicative Decrease (cut `cwnd` in half on loss).
*   **TCP Tahoe:** On any loss (Timeout or Triple Duplicate ACK), set `cwnd = 1` and enter Slow Start.
*   **TCP Reno:**
    *   **Timeout:** `cwnd = 1`.
    *   **Triple Duplicate ACK:** Fast Recovery. Set `cwnd = cwnd / 2`, skip Slow Start, and continue with AIMD.

```text
* Visual Aid: Congestion Window Behavior *

Window Size
   ^
   |           /  (AIMD)
   |          / \
   |         /   \ (Loss -> Cut in half)
   |   (SS) /     \  /
   |       /       \/
   |      /
   +--------------------------> Time
```

### 4.10 QUIC Protocol
QUIC is a modern transport protocol designed to replace TCP+TLS for HTTP traffic.
*   **Stack:** Moves reliability and security into user-space on top of UDP.
    *   *Old:* HTTP/2 -> TLS -> TCP -> IP.
    *   *New:* HTTP/3 -> QUIC -> UDP -> IP.
*   **Latency:**
    *   **1-RTT:** Standard secure handshake (faster than TCP+TLS which took 2-3 RTTs).
    *   **0-RTT:** If a client has spoken to a server recently, it can send encrypted data *in the very first packet*.
*   **HOL (Head-of-Line) Blocking:** In TCP, one lost packet blocks all streams. QUIC supports independent streams; loss in one stream does not block others.

```text
* Visual Aid: TCP vs QUIC Handshake *

      [ TCP + TLS 1.2 ]               [ QUIC ]

Client               Server     Client         Server
  | ----- SYN ------>  |           | --- CHLO ---> | (Ctl)
  | <---- SYNACK ----  |           |               | (Key)
  | ----- ACK ------>  |           | <--- REJ ---- |
  |                    |           |               |
  | -- ClientHello ->  |           | --- CHLO ---> |
  | <- ServerHello --  |           | -- Enc Req -> |
          ...                      | <--- SHLO --- |
                                   | <- Enc Resp - |
     (Multiple RTTs)                 (1 RTT or 0 RTT)
```

---

## 5.0 Conclusion & Preview

### 5.1 Review Questions
To test understanding:
*   What requirements distinguish Multiplexing from Congestion Control?
*   How does the receiver handle out-of-order packets in Go-Back-N vs. Selective Repeat?
*   Why is the handshake necessary (synchronizing sequence numbers)?
*   How does QUIC achieve 0-RTT safely?

### 5.2 Background Papers
*   **End-to-End Principle (Saltzer):** Intelligence should be at the endpoints (hosts), not in the core network (routers).
*   **TCP/IP Philosophy:** Design for survivability and flexibility over strict accounting.

### 5.3 What's to come (Network Layer)
*   **Control Plane:** The logic/software (Routing Algorithms) that decides paths (OSPF, BGP).
*   **Data Plane:** The hardware mechanism (Forwarding) that moves packets from input ports to output ports in a router.
*   **Router Architecture:** Input buffers, Switch fabric, Output buffers.
*   **IPv6:** addressing the exhaustion of IPv4 addresses.
