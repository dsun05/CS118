# Computer Networks: Lecture Study Notes
**Topics:** Application Layer (DASH), Transport Layer (UDP/TCP, Reliable Transfer), TCP Congestion Control (RFC 5681)

## 1.0 Lecture Overview and HW2 Review

### 1.1 Outline and Review
*   **HW2 Review:** A detailed walkthrough of specific problems regarding delay calculations and HTTP connection modes.
*   **Lecture Review:**
    *   **Application Layer:** Introduction to **DASH** (Dynamic Adaptive Streaming over HTTP), a modern video streaming standard.
    *   **Transport Layer:** Transitioning from Application layer down to Transport. Focus on multiplexing/demultiplexing, UDP, and the complexities of building a reliable protocol (TCP) over an unreliable network (IP).

### 1.2 HW2 Review Problems

#### **Problem 3: Calculation of Total Delay**
This problem involves calculating the time elapsed from clicking a link to receiving an object, accounting for DNS resolution, TCP setup, and transmission.

*   **Scenario:**
    *   DNS lookup required ($N$ servers visited).
    *   One object to fetch.
    *   **RTT parameters:** $RTT_1...RTT_n$ for DNS, $RTT_0$ for the server interaction.
    *   **Transmission time:** $0.002 \times RTT_0$.
*   **Breakdown of Delays:**
    1.  **DNS Lookup:** The sum of all round-trip times to the visited DNS servers.
    2.  **TCP Connection Setup:** Requires a 3-way handshake, costing $1 \times RTT_0$ before a request can be sent.
    3.  **HTTP Request/Response:** Sending the GET request and receiving the first byte of response costs $1 \times RTT_0$.
    4.  **Object Transmission:** The time to push the bits onto the wire.

**Visual Aid: Equation for Total Delay summing RTTs and transmission time**
```text
Total Delay = DNS_Delay + TCP_Handshake + HTTP_Request + Trans_Time

  Total Delay = [ Σ(RTT_i) for i=1 to n ]  <-- DNS Lookup
              + 2 * RTT_0                  <-- TCP Setup (1 RTT) + HTTP Req (1 RTT)
              + 0.002 * RTT_0              <-- Transmission Time
```

#### **Problem 4: Comparison of HTTP Modes**
This problem compares the efficiency of fetching a base HTML file and 9 referenced objects using different HTTP connection strategies.
*   **Assumptions:** All RTTs are equal ($RTT$). Negligible transmission time for the base file.

**Comparison:**
1.  **Non-persistent HTTP, no parallel TCP connections:**
    *   **Mechanism:** Open TCP -> Fetch Object -> Close TCP. Repeat for all 10 objects (1 base + 9 refs).
    *   **Cost:** 2 RTT per object (1 for handshake + 1 for request).
    *   **Calculation:** $2 RTT \times 10 = 20 RTT$. (Plus DNS delay).

2.  **Non-persistent HTTP, 6 parallel TCP connections:**
    *   **Mechanism:** Fetch base file ($2 RTT$). Then, open parallel connections for the 9 objects.
    *   **Batching:** With 6 connections allowed, we can fetch the first 6 objects in one parallel "round". The remaining 3 objects are fetched in a second "round".
    *   **Calculation:** Base ($2 RTT$) + Round 1 ($2 RTT$) + Round 2 ($2 RTT$) = $6 RTT$.

3.  **Persistent HTTP (no pipelining):**
    *   **Mechanism:** Open TCP once ($1 RTT$). Fetch base ($1 RTT$). Reuse connection for remaining 9 objects.
    *   **Cost:** Handshake happens only once. Subsequent objects cost only $1 RTT$ each (Request/Response).
    *   **Calculation:** Handshake ($1 RTT$) + Base Req ($1 RTT$) + 9 Object Reqs ($9 RTT$) = $11 RTT$.

**Visual Aid: Calculations comparing delay for the three HTTP modes**
```text
Delay Comparison (Ignoring DNS for simplicity):

1. Non-Persistent (Serial):
   Total = (2 RTT/obj) * 10 objects 
         = 20 RTT

2. Non-Persistent (Parallel, limit 6):
   Base HTML: 2 RTT
   Objects:   9 objects / 6 conn = 2 batches needed.
              Each batch costs 2 RTT (setup + req).
   Total    = 2 RTT (base) + 2 * 2 RTT (batches) 
            = 6 RTT

3. Persistent (Reuse connection):
   Setup:     1 RTT (Initial Handshake)
   Requests:  1 RTT (Base) + 9 RTT (9 Objects)
   Total    = 11 RTT
```

***

## 2.0 Application Layer: DASH

### 2.1 DASH: Video Streaming Protocol
**DASH** stands for **Dynamic Adaptive Streaming over HTTP**. It is the dominant standard for video streaming (used by YouTube, Netflix) designed to handle varying network conditions efficiently.

*   **Procedure:**
    1.  **Segmentation:** The server divides the video file into short chunks (e.g., 4-second segments).
    2.  **Multiple Bitrates:** Each chunk is encoded independently at multiple quality levels (e.g., 480p, 720p, 1080p, 4K).
    3.  **Manifest Creation:** The server creates an **MPD (Media Presentation Description)** file. This is a "menu" that lists every chunk and the URL for each quality level available.
    4.  **Adaptive Streaming (Client-Side Logic):**
        *   **Intelligence at the Edge:** The server is "dumb" (just hosts files). The client is "smart."
        *   The client monitors available bandwidth.
        *   If bandwidth is high, the client requests high-bitrate chunks.
        *   If bandwidth drops, the client requests the next chunk at a lower bitrate to prevent buffering.

### 2.2 DASH Example
*   **Implementation:** DASH relies on standard HTTP GET requests.
*   **Reference:** A sample implementation can be found in the `dash.js` library, which parses the manifest and manages the chunk requests automatically in a web browser.

***

## 3.0 Transport Layer: Multiplexing and Reliable Transfer

### 3.1 UDP and TCP Multiplexing
The transport layer creates logical communication channels between processes.
*   **Multiplexing:** Gathering data chunks from different sockets, encapsulating them with headers (adding Port numbers), and passing them to the network layer.
*   **Demultiplexing:** Delivering received segments to the correct socket based on IP addresses and Port numbers.
*   **Socket Implementation:** Both UDP and TCP sockets in C utilize these ports to bind processes to network interfaces. (See GeeksforGeeks references in slides).

### 3.2 Reliable Transfer Principles
The core challenge of the Transport layer is building a **reliable** data transfer protocol (like TCP) on top of an **unreliable** network layer (IP).
*   **The Reality of IP:** The underlying network may drop packets, reorder them, or corrupt bits.
*   **TCP's Goal:** Provide a guarantee that the byte stream is delivered in order and without error.

**The Two General’s Problem**
This is a thought experiment illustrating why guaranteed certainty in distributed communication over a lossy link is theoretically impossible.
*   **Scenario:** Army A1 and Army A2 are separated by a valley defended by City B. They must attack simultaneously to win.
*   **Constraint:** They communicate via messengers through the valley who may be captured (packet loss).
*   **The Dilemma:**
    *   A1 sends: "Attack at 0900." -> A1 doesn't know if A2 got it.
    *   A2 sends ACK: "Agreed." -> A2 doesn't know if A1 received the ACK.
    *   A1 sends ACK to ACK: "Got your confirmation." -> A1 doesn't know if this arrived.
    *   **Infinite Regress:** No matter how many messages are exchanged, the last sender can never be 100% sure their last message arrived.
*   **Key Takeaways for Protocol Design:**
    1.  **Acknowledgements (ACKs)** are essential to handle loss.
    2.  **Timers/Local Observation:** In a lossy environment, a node must rely on its own timeouts to decide if a message failed; absolute certainty is impossible.

### 3.3 Reliable Data Transfer (rdt) Versions
We build a reliable protocol iteratively, adding complexity only as needed to handle new types of errors.

**Visual Aid: Table comparing rdt versions 1.0 through 3.0**

| Version | Channel Assumptions | Mechanisms Added |
| :--- | :--- | :--- |
| **rdt1.0** | Perfect channel (No loss, no errors) | None needed. Just send and receive. |
| **rdt2.0** | **Bit Errors** occur (packets arrive but may be corrupted). No loss. | 1. **Checksum:** To detect errors.<br>2. **Feedback:** Receiver sends **ACK** (OK) or **NAK** (Error).<br>3. **Retransmission:** Sender resends on NAK. |
| **rdt2.1** | Same as 2.0, but ACKs themselves can be corrupted. | **Sequence # (0 or 1):** Sender adds a bit to packets so receiver can distinguish a retransmission from a new packet. |
| **rdt2.2** | Same as 2.1, but removes NAKs. | **Unexpected ACK:** Instead of sending NAK for pkt 1, receiver sends ACK for pkt 0. Sender treats duplicate ACK as NAK. |
| **rdt3.0** | **Errors + Packet Loss** (Packets disappear). | **Countdown Timer:** Sender waits a specific time for ACK. If timeout occurs, retransmit. |

### 3.4 Performance Issues
**rdt3.0 (Stop and Wait)** is functional but has terrible performance.
*   **Stop and Wait:** Sender sends one packet, then stops and waits for the ACK before sending the next.
*   **Throughput Calculation:**
    *   Link: 50 Kbps. RTT: 500ms (approx). Packet: 1000 bits.
    *   Time to send = $L/R$ (very small).
    *   Time waiting = $RTT$ (very large).
    *   Utilization $U = (L/R) / (RTT + L/R)$.
    *   **Result:** The effective throughput is tiny (~1.9 Kbps) compared to the link capacity (50 Kbps). We are wasting bandwidth waiting.

***

## 4.0 Pipelined Protocols
To solve the performance issue of Stop-and-Wait, we use **Pipelining**: allowing multiple packets to be "in flight" before receiving an ACK.

### 4.1 Go-back-N vs. Selective Repeat
Two main approaches to sliding window protocols:

**1. Go-Back-N (GBN)**
*   **Sender:** Can have up to $N$ unacknowledged packets.
*   **Receiver:** simpler logic. Only accepts packets **in order**.
    *   If packet $n$ is missing, it discards $n+1, n+2...$
*   **ACK Mechanism:** **Cumulative ACK**. `ACK(n)` means "I have received everything up to and including $n$."
*   **Timeout:** Uses a single timer for the **oldest unACKed packet**. If it expires, sender retransmits **ALL** packets in the current window (Go back... N).

**2. Selective Repeat (SR)**
*   **Sender:** Can have up to $N$ unacknowledged packets.
*   **Receiver:** More complex. Buffers packets received out-of-order.
*   **ACK Mechanism:** **Individual ACK**. `ACK(n)` means "I received packet $n$ specifically."
*   **Timeout:** Timer for **each individual packet**. If packet $n$ times out, only packet $n$ is retransmitted.

**Visual Aid: Screenshot of Animation Demo for GBN/SR**
*(Imagine a web simulation)*
*   **GBN:** A lost packet causes a block of subsequent packets to turn "red" (discarded) at the receiver, and the sender resends the whole block.
*   **SR:** A lost packet leaves a single gap. The receiver holds the subsequent packets (buffered). The sender eventually resends just the missing one, filling the gap.

### 4.2 Protocol Comparison

**Visual Aid: Table comparing Buffer, ACK, and Timeout methods**

| Protocol | Buffer at Sender | Buffer at Receiver | ACK Type | Timeout Behavior |
| :--- | :---: | :---: | :--- | :--- |
| **Stop-and-Wait** | No | No | No out-of-order ACKs | Retransmit current packet |
| **Go-Back-N** | **Yes** | **No** (Discards out-of-order) | **Cumulative Seq#** (ACK $n$ confirms all $\le n$) | Retransmit **ALL** packets in window |
| **Selective Repeat**| **Yes** | **Yes** (Buffers out-of-order) | **Received Seq#** (Individual ACK) | Retransmit **ONLY** the specific timeout packet |

### 4.3 Sequence Numbers in Sliding Windows
We must choose the window size ($N$) relative to the range of available sequence numbers to avoid ambiguity.
*   **Go-Back-N:** Required Seq# Range $\ge N+1$.
*   **Selective Repeat:** Required Seq# Range $\ge 2N$.

**Visual Aid: Diagram showing sequence number ambiguity in Selective Repeat**
*Problem: If Seq# range is too small (e.g., 0, 1, 2, 3 and Window size=3), the receiver cannot tell if a packet labeled "0" is a retransmission of the "old 0" or the "new 0" from the next window.*

```text
Sender Window: [0, 1, 2] ... 3 ... 0
Receiver Window: ... [0, 1, 2] ...
    
    If sender sends 0, 1, 2 and receiver ACKs all.
    Receiver window moves to [3, 0, 1].
    
    Scenario A: ACKs arrive. Sender moves window, sends new packet 0.
    Scenario B: ACKs lost. Sender retransmits old packet 0.
    
    Receiver accepts "0" in both cases. If Range < 2N, data corruption occurs.
```

***

## 5.0 Transmission Control Protocol (TCP)

### 5.1 Overview and Functionalities
TCP (RFC 793) is a point-to-point, reliable, byte-stream protocol.
*   **Key Functionalities:**
    *   **Streams:** Data is treated as a continuous stream of bytes, not discrete messages.
    *   **Reliable Delivery:** Ensures data arrives without loss or error.
    *   **Flow Control:** Prevents a fast sender from overwhelming a slow receiver.
    *   **Multiplexing:** Uses Source and Destination ports.
    *   **Full Duplex:** Data can flow in both directions on the same connection.

### 5.2 TCP Header Format
The TCP header is usually 20 bytes (without options).

**Visual Aid: Diagram of TCP Header 32-bit structure**
```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Acknowledgment Number                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |           |U|A|P|R|S|F|                               |
| Offset| Reserved  |R|C|S|S|Y|I|            Window             |
|       |           |G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (variable length)                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
*   **Seq Number:** Byte count of the first byte in the segment.
*   **Ack Number:** The *next* byte the receiver expects to receive.
*   **Flags:**
    *   **SYN:** Synchronize seq numbers (Start connection).
    *   **FIN:** Finish connection.
    *   **RST:** Reset connection.
    *   **ACK:** Ack field is valid.
*   **Window:** Current size of the receiver's free buffer space (for Flow Control).
*   **Checksum:** Covers header, data, and a "pseudo-header" (IP addresses) to protect against misrouting.

### 5.3 Connection Setup (Three-Way Handshake)
TCP establishes a connection before exchanging data.

*   **Procedure:**
    1.  **Client -> Server:** `SYN`, Seq = `x` (Random Initial Seq Number).
    2.  **Server -> Client:** `SYN`, `ACK`, Seq = `y`, Ack = `x+1`.
    3.  **Client -> Server:** `ACK`, Seq = `x+1`, Ack = `y+1`.
*   **Why 3-Way?** The third step (ACK from Client) confirms to the Server that the Client is alive and received the Server's SYN. It prevents old, delayed connection requests from confusing the server.

**Visual Aid: Diagram of Client/Server state transitions during setup**
```text
      Client State                             Server State
        LISTEN                                   LISTEN
          |                                        |
  (Send SYN) -------------------------------> (Receive SYN)
       SYN_SENT                                 SYN_RCVD
          |                                        |
          <------------------------------- (Send SYN+ACK)
  (Receive SYN+ACK)                                |
   (Send ACK) ------------------------------> (Receive ACK)
     ESTABLISHED                              ESTABLISHED
```

### 5.4 Data Transfer and Teardown
*   **Data Transfer:** TCP is "Piggybacked." Data and ACKs travel together. Seq numbers track bytes, not packets.
*   **Connection Teardown:**
    *   TCP allows a "Half-Close": One side can say "I'm done sending" (`FIN`) but continue receiving.
    *   Full teardown usually requires 4 segments (FIN, ACK, FIN, ACK).

**Visual Aid: Diagram of FIN/ACK exchange**
```text
Client                  Server
  | ----- FIN ----->       |  (Client done sending)
  | <---- ACK ------       |  (Server acknowledges)
  |                        |  (Server sends remaining data...)
  | <---- FIN ------       |  (Server done sending)
  | ----- ACK ----->       |  (Client acknowledges)
  |                        |
(Timed Wait)            (Closed)
 (Closed)
```

### 5.5 Flow Control & Timeout
*   **Flow Control:**
    *   Goal: Prevent buffer overflow at the receiver.
    *   Mechanism: The receiver advertises a `rwnd` (Receive Window) value in every TCP header. The sender ensures `FlightSize <= rwnd`.
*   **RTT Estimation (Timeout Calculation):**
    *   TCP must estimate the Round Trip Time (RTT) to set the timeout accurately.
    *   **EstimatedRTT:** Exponential Weighted Moving Average (EWMA) of samples.
        *   `EstimatedRTT = (1-α)*EstimatedRTT + α*SampleRTT`
    *   **DevRTT:** Measures variance/jitter.
    *   **RTO (Retransmission Timeout):** `EstimatedRTT + 4 * DevRTT`.
*   **Karn’s Algorithm:**
    *   Problem: If we retransmit a packet, and get an ACK, is it for the original or the retransmission? We don't know.
    *   Rule: **Do not** update RTT stats using retransmitted segments. Double the timeout interval (exponential backoff) each time a timeout occurs.

***

## 6.0 Administrative & Midterm Review

### 6.1 Midterm Hints
*   **Scope:** Chapters 1, 2, and 3.
*   **Key Topics:**
    *   Socket Programming (concepts from Project 1).
    *   **RFC 5681 (Congestion Control):** Prepare for one specific problem on this.
    *   **Self-contained problems:** You do not need to memorize every bit-level detail of rdt2.0/3.0, but you must understand the logic (ACKs, timers, states) to solve the problems provided.
*   **Review Areas:** Multiplexing, DNS, TCP/UDP Headers, Reliable Data Transfer logic.

***

## 7.0 RFC 5681: TCP Congestion Control
TCP Congestion Control prevents the network core from collapsing under too much traffic. It uses a variable called **`cwnd` (Congestion Window)** to throttle the sender.

### 7.1 Definitions
*   **SMSS (Sender Maximum Segment Size):** Largest chunk of data sender can send (usually 1460 bytes).
*   **rwnd (Receiver Window):** Flow control limit advertised by receiver.
*   **cwnd (Congestion Window):** Congestion control limit maintained by sender.
*   **Flight Size:** Bytes sent but not yet ACKed. `Max FlightSize = min(cwnd, rwnd)`.
*   **Duplicate ACK:** An ACK that re-confirms the *same* sequence number, implying a subsequent packet was lost/out-of-order.

### 7.2 Slow Start
*   **Goal:** Quickly find the available bandwidth.
*   **Start:** `cwnd` starts small (e.g., 1-4 SMSS).
*   **Growth:** **Exponential**. For every ACK received, `cwnd` increases by 1 SMSS.
    *   Effectively, `cwnd` doubles every RTT.
*   **Transition:** Stops when `cwnd >= ssthresh` (Slow Start Threshold).

**Visual Aid: Graph showing cwnd growth over time**
```text
       ^
       |           /  (Linear Growth - Congestion Avoidance)
 Window|          /
 Size  |         /
       |        /
       |       /
       |      /
thresh |-----/
       |    /
       |   _| (Exponential Growth - Slow Start)
       |__|___________________________
                  Time ->
```

### 7.3 Congestion Avoidance
*   **Goal:** Gently probe for more bandwidth without causing immediate congestion.
*   **Trigger:** When `cwnd >= ssthresh`.
*   **Growth:** **Linear**. `cwnd` increases by 1 SMSS per RTT (approx `cwnd += SMSS*SMSS/cwnd` per ACK).
*   **Response to Timeout (Severe Loss):**
    1.  `ssthresh` drops to `FlightSize / 2`.
    2.  `cwnd` is reset to **1 SMSS**.
    3.  System restarts in **Slow Start**.

### 7.4 Fast Retransmit
*   **Trigger:** Waiting for a timeout is too slow. If the sender receives **3 Duplicate ACKs** (4 ACKs total for the same seq#), it assumes the packet is lost.
*   **Action:** Retransmit the missing segment *immediately*, without waiting for the timer to expire.

### 7.5 Fast Recovery
*   **Logic:** If we get duplicate ACKs, flow is still flowing (packets are arriving, just out of order). We shouldn't drop to `cwnd=1` (Slow Start) effectively stopping transmission.
*   **Algorithm:**
    1.  On 3rd Dup ACK: `ssthresh = FlightSize / 2`.
    2.  Set `cwnd = ssthresh + 3 SMSS` (inflate window to account for buffered packets).
    3.  For every additional Dup ACK, increment `cwnd` by 1 SMSS (keep transmitting new data if allowed).
    4.  When the **New ACK** arrives (filling the hole): Set `cwnd = ssthresh` (Deflate window back to the new target) and enter **Congestion Avoidance**.

***

### Key Terms for Definition

*   **DASH (Dynamic Adaptive Streaming over HTTP):** A video streaming technique that adjusts video quality chunks based on real-time network conditions.
*   **Manifest (MPD):** A file describing media chunks and available bitrates in DASH; essentially the "menu" for the client.
*   **Two General's Problem:** A theoretical problem illustrating that 100% reliable consensus is impossible over an unreliable link with finite messages.
*   **Go-Back-N (GBN):** A sliding window protocol using cumulative ACKs; a single packet loss causes retransmission of the entire window.
*   **Selective Repeat (SR):** A sliding window protocol that ACKs individual packets; only lost packets are retransmitted, but requires buffering at the receiver.
*   **Three-Way Handshake:** The process (SYN, SYN-ACK, ACK) used to establish a TCP connection reliably.
*   **Flow Control:** A mechanism (using `rwnd`) to prevent a fast sender from overwhelming a slow receiver.
*   **Congestion Window (cwnd):** A TCP state variable that limits the amount of unacknowledged data allowed in the network to prevent network congestion.
*   **Slow Start:** A TCP algorithm where `cwnd` increases exponentially to quickly find available bandwidth.
*   **Congestion Avoidance:** A TCP algorithm where `cwnd` increases linearly to probe for bandwidth cautiously.
*   **Fast Retransmit:** Retransmitting a packet after receiving 3 duplicate ACKs before the timer expires.
*   **Fast Recovery:** An algorithm used after Fast Retransmit that maintains high throughput by avoiding a reset to Slow Start, adjusting `cwnd` based on incoming duplicate ACKs.
