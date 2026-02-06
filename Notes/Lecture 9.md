## 1.0 TCP Congestion Control Algorithms (RFC 5681)

### Overview
TCP congestion control consists of four specific algorithms defined in RFC 5681. These components work in unison to manage the rate at which data is injected into the network, ensuring efficiency while preventing network collapse.

1.  **Slow Start:** A mechanism designed to **jump-start** the transmission process by rapidly discovering the available bandwidth.
2.  **Congestion Avoidance:** Once the network capacity is roughly estimated, this phase uses **additive increase** to slowly probe for additional bandwidth.
3.  **Fast Retransmit / Fast Recovery:** A set of mechanisms to recover from occasional single packet losses (indicated by duplicate ACKs) without a full restart. It employs **multiplicative decrease** to reduce the sending rate moderately.
4.  **Retransmission upon Timeout:** This is the fail-safe mechanism for **heavy congestion handling**, triggered when no feedback is received for an extended period.

***

### 1.1 Illustrative Examples and Initial Behavior

The behavior of TCP congestion control changes dynamically based on network feedback. The goal is to maximize throughput while avoiding packet loss.

*   **Visual Aid: Congestion Window Evolution**
```text
      Congestion Window (cwnd)
          ^
          |           /|   /|
      60  |          / |  / |
          |         /  | /  |
      40  |        /   |/   | <-- Fast Retransmission/Recovery
          |       /    +----/     (Multiplicative Decrease)
      20  |      /    /    /
          |     /    /    /
      10  |    /    /    / <-- Additive Increase (Linear)
          |   /|   /    /
       0  +--/-+--/----+---------------------> Time
            /  |
      Slow Start (Exponential)   Timeout (Drop to 1)
```

**Behavior when no loss (initially):**
*   **Slow Start Phase:** Upon connection initiation, the sender does not know the network capacity. To utilize available bandwidth quickly, the sending rate grows via **exponential growth**. This is the "startup phase."
*   **Congestion Avoidance Phase:** Once the sending rate reaches a safety threshold (indicating it is close to capacity), the protocol switches to **linear growth**. This cautious increase prevents flooding the network immediately after ramping up.

***

### 1.2 TCP Slow Start Algorithm

**Initial State:**
*   When a TCP connection begins, the congestion window (**`cwnd`**) is initialized to a small value, typically **`cwnd ≤ 2 MSS`** (often set to 1 MSS).
*   **Example:** If MSS = 500 Bytes and RTT = 200 msec, the initial rate is approximately 20 kbps.

**Goal:**
Since the available bandwidth might be significantly larger than the initial rate (MSS/RTT), the goal is to **quickly ramp up** to a respectable speed.

**Mechanism:**
*   The sender increases the rate **exponentially fast**.
*   **Per RTT:** The **`cwnd`** effectively doubles every Round Trip Time (RTT).
*   **Per ACK:** Practically, this is implemented by the formula `cwnd = cwnd + 1MSS` for every new ACK received.
    *   *Rationale:* If you send 1 packet and get 1 ACK, `cwnd` becomes 2. If you send 2 packets and get 2 ACKs, `cwnd` becomes 4.

**Condition:**
This algorithm is used as long as **`cwnd < ssthresh`** (slow-start threshold).

*   **Visual Aid: Slow Start Packet Flow**
```text
 Host A                                      Host B
   |                                           |
   |---(1 segment)---------------------------->| cwnd = 1 MSS
   |<----------------------------------(ACK)---| cwnd becomes 2 MSS
   |                                           |
   |---(2 segments)--------------------------->|
   |==========================================>|
   |<----------------------------------(ACK)---|
   |<----------------------------------(ACK)---| cwnd becomes 4 MSS
   |                                           |
   |---(4 segments)--------------------------->|
   |==========================================>|
   |==========================================>|
   |==========================================>|
   v                                           v
```

***

### 1.3 Transition: Switching from Slow Start to Congestion Avoidance (CA)

**Variable `ssthresh`:**
The **`ssthresh`** (Slow Start Threshold) is a critical variable that acts as a gatekeeper. It dictates when TCP should switch from the aggressive **exponential increase** (Slow Start) to the cautious **linear increase** (Congestion Avoidance).

**Setting `ssthresh`:**
When a loss event occurs, **`ssthresh`** is updated to be **halfway of the maximum value** of the **`cwnd`** recorded right before the loss occurred. This "remembers" the last safe operating window size.

*   **Visual Aid: Transition Point**
```text
 cwnd (segments)
  12 |           /
     |          /
  10 |         /
     |        /
   8 |-------/--------- ssthresh (Threshold)
     |      /
   6 |     /
     |    /  <-- Exponential Growth stops here
   4 |   /
     |  /
   2 | /
     |/
   0 +--------------------> Time
```

***

### 1.4 Congestion Avoidance Algorithm

**Mechanism:**
Once the threshold is crossed, TCP assumes the network is nearly full and becomes conservative.
*   **Behavior:** Increase **`cwnd`** by **1 MSS per RTT**. This is known as **Additive Increase**.
*   **Implementation:**
    *   **Formula per RTT:** `cwnd = cwnd + 1MSS`.
    *   **Formula per ACK:** Since we want to increase by 1 full MSS only after *all* segments in the window are ACKed, the update per non-duplicate ACK is:
        `cwnd = cwnd + (MSS / cwnd) × MSS`
    *   *Note:* As `cwnd` gets larger, the increment per ACK gets smaller, resulting in linear growth overall.

**Condition:**
This phase is active when **`cwnd > ssthresh`** and no packet loss is occurring.

**Note:**
While logic is often described in terms of "segments," the **`cwnd`** variable is technically maintained in **bytes**.

***

### 1.5 Loss Detection and Reaction

TCP reacts differently depending on *how* the loss was detected, as the detection method implies the severity of the congestion.

1.  **Detection through duplicate ACKs:**
    *   **Diagnosis:** This implies **mild congestion**. Some packets are getting through (triggering ACKs), but one was lost.
    *   **Trigger:** Reception of 3 duplicate ACKs.
    *   **Action:** Enter **Fast Retransmit / Fast Recovery**.
    *   **Result:** **Multiplicative decrease** (halving) of `cwnd`, rather than a full reset.

2.  **Detection through retransmission timeout:**
    *   **Diagnosis:** This implies **heavy congestion**. It is likely that the network pipeline is severely blocked, as no ACKs are returning.
    *   **Action:** **Reset and restart from scratch**.
    *   **Result:** `cwnd` is reset to 1 MSS, and the process restarts with Slow Start.

## 2.0 Fast Retransmit / Fast Recovery

### 2.1 Philosophy and Mechanism

**Philosophy:**
*   TCP uses duplicate ACKs as a hint. If only 1 or 2 duplicate ACKs are received, TCP assumes it might just be **transient out-of-order delivery** (packets arriving in a different sequence) and does nothing.
*   However, receiving **3 duplicate ACKs** is statistically strong evidence that a segment was actually lost.
*   Crucially, receiving a duplicate ACK means *other* packets are arriving at the destination. The pipe is not fully broken; therefore, we should not stop transmitting entirely.

**Fast Retransmit:**
*   **Trigger:** Arrival of the 3rd duplicate ACK.
*   **Action:** The sender immediately retransmits the missing segment (without waiting for the timer to expire).
*   **Calculations:**
    *   Update threshold: `ssthresh = max(cwnd/2, 2MSS)`
    *   Inflate window: `cwnd = ssthresh + 3MSS` (The "+3" accounts for the 3 packets that triggered the dup ACKs, which are known to have left the network).

**Fast Recovery:**
*   This algorithm governs transmission while the sender is waiting for the retransmitted packet to be ACKed.
*   **Mechanism:** For every *additional* duplicate ACK received (beyond the 3rd), `cwnd` is increased by **1 MSS**. This utilizes the "packets left the network" principle to inject new data, maintaining high throughput.

### 2.2 Algorithm Logic (Pseudo-code)

1.  **Initial State:** `fastretx = false`
2.  **Upon receiving 3rd duplicate ACK:**
    *   Calculate new safety threshold: `ssthresh = max(cwnd/2, 2MSS)`
    *   Set window to threshold plus buffer: `cwnd = ssthresh + 3MSS`
    *   **Retransmit** the lost TCP packet immediately.
    *   Enter Fast Recovery: `fastretx = true`
3.  **If `fastretx == true` (Receiving Additional Dup ACKs):**
    *   Inflate window: `cwnd += 1MSS`
    *   Transmit a new segment if the new `cwnd` allows it.
4.  **If `fastretx == true` (Receiving New/Non-duplicate ACK):**
    *   This ACK acknowledges the retransmitted data and potentially more.
    *   Deflate window: `cwnd = ssthresh`
    *   Exit Fast Recovery: `fastretx = false`
    *   *Result:* The window has effectively decreased by ~50% (multiplicative decrease) rather than crashing to 1.

## 3.0 Retransmission Timeout & Summary

### 3.1 Retransmission Timeout Operation

**Trigger:** The Retransmission Timer expires (RTO).
**Reasoning:** This is treated as a catastrophic loss of flow (Heavy Congestion).

**Updates:**
1.  **Save State:** `ssthresh = max(cwnd/2, 2MSS)` (Save half the current window as the new target).
2.  **Hard Reset:** **`cwnd = 1MSS`**.
3.  **Action:** Retransmit the lost TCP segment.
4.  **Phase:** Resume immediately in **Slow Start** mode.

***

### 3.2 Summary of Algorithms

*   **Visual Aid: Summary Table**

| Algorithm | Condition | Design / Philosophy | Operation |
| :--- | :--- | :--- | :--- |
| **Slow Start** | `cwnd < ssthresh` | Jump-start transmission | `cwnd` doubles per RTT (Exponential). |
| **Congestion Avoidance** | `cwnd > ssthresh` | Probe bandwidth carefully | Additive increase (+1 MSS per RTT). |
| **Fast Retransmit** | 3 Duplicate ACKs | Infer loss without timeout | `ssthresh = cwnd/2`; Retransmit packet. |
| **Fast Recovery** | Following Fast Retransmit | Maintain flow during recovery | Inflate `cwnd` for dup ACKs; exit on new ACK. |
| **RTO Timeout** | Timer Expiry | Heavy Congestion Handling | Reset `cwnd` to 1; `ssthresh = cwnd/2`. |

## 4.0 Integration of TCP Components

### 4.1 Interaction of Components
A functional TCP implementation integrates three distinct mechanisms to handle reliability and limits.

1.  **Selective Repeat (SR):** Handles the **reliable transfer** of data (managing sequence numbers and retransmissions).
2.  **Congestion Control:** Limits sending rate based on **Internet congestion** (protects the network).
3.  **Flow Control:** Limits sending rate based on the **receiver’s buffer availability** (protects the receiver).

**The Governing Formula:**
The actual amount of unacknowledged data the sender can have in flight (`N`) is determined by the minimum of the congestion window and the receiver's advertised window:
`update N = min(cwnd, rwnd)`

*   **`cwnd`:** Calculated by Slow Start / Congestion Avoidance.
*   **`rwnd`:** Reported by the receiver via Flow Control.
*   **Example:** If Congestion Control allows 20 packets (`cwnd=20`), but the receiver only has space for 10 (`rwnd=10`), the sender stops at 10.

***

### 4.2 Illustrative Example: TCP SR and Congestion Control

**Scenario Settings:**
*   All 4 algorithms active.
*   Initial `ssthresh = 4`.
*   Packet #5 is lost.
*   Packets are 1 byte (unit) for simplicity.

**Step-by-Step Trace:**
*   **Visual Aid: Trace Timeline**
```text
State: Slow Start (ssthresh=4)
1. cwnd=1. Send Pkt 1.
2. Rx Ack 1. cwnd increases to 2. Send Pkt 2, 3.
3. Rx Ack 2. cwnd increases to 3. Send Pkt 4, 5, 6.
4. Rx Ack 3. cwnd increases to 4. Send Pkt 7, 8, 9...
   (Note: Pkt 5 is lost in transit)

State: Loss Detection
5. Rx Ack 4. (Expects Pkt 5).
6. Rx Ack 5 (1st Dup). Sender does nothing.
7. Rx Ack 5 (2nd Dup). Sender does nothing.
8. Rx Ack 5 (3rd Dup). -> TRIGGER FAST RETRANSMIT.

State: Fast Retransmit / Recovery
9. ssthresh set to max(cwnd/2) -> e.g., if cwnd was 5, ssthresh=2.
10. cwnd set to ssthresh + 3 = 5.
11. Retransmit Pkt 5 immediately.
12. Rx Ack 5 (4th Dup). cwnd becomes 6. Send new Pkt 10.

State: Exit Recovery
13. Rx Ack 10 (New ACK covering Pkt 5 and subsequent successful packets).
14. cwnd resets to ssthresh (2).
15. Enter Congestion Avoidance (Linear Growth).
```

### 4.3 Congestion Location
**Bottleneck Link:**
TCP increases its sending rate until the aggregate traffic exceeds the capacity of a specific link in the path.
*   Packets arrive at a router faster than they can be forwarded.
*   The router's queue fills up.
*   When the queue overflows, packets are dropped. This drop is the feedback signal TCP uses to detect the **bottleneck link**.

*   **Visual Aid: Bottleneck Diagram**
```text
 [Source] ===> [Router A] ===> [||||||] ===> [Router B] ===> [Dest]
                                   ^
                           Bottleneck Queue
                          (Packet Loss occurs here)
```

## 5.0 Evolving Transport-Layer Functionality

### 5.1 Modern Challenges and TCP Flavors
TCP and UDP have been the standard for 40 years, but modern internet usage has created scenarios where standard TCP is suboptimal. This led to different "flavors" (e.g., TCP Reno, TCP Tahoe, TCP Cubic).

*   **Long, fat pipes:** On high-bandwidth, high-latency links (e.g., intercontinental fiber), recovering from a loss using standard TCP takes too long to ramp back up.
*   **Wireless networks:** Wireless links often drop packets due to interference or mobility, not congestion. TCP misinterprets this as congestion and lowers the rate unnecessarily.
*   **Data center networks:** These require extremely low latency, and background traffic can interfere with high-priority tasks.

**Trend:** To solve these issues without changing the OS kernel on every machine, developers are moving transport functions to the **application layer**, running on top of UDP.

***

### 5.2 QUIC (Quick UDP Internet Connections)

**Overview:**
*   **QUIC** is an application-layer protocol built on top of **UDP**.
*   It was primarily designed to improve HTTP performance, leading to the **HTTP/3** standard.
*   Heavily deployed by Google (Chrome, YouTube, Search) to optimize user experience.

*   **Visual Aid: Protocol Stack Comparison**
```text
   |   HTTP/2    |          |    HTTP/3     |
   |-------------|          |---------------|
   |     TLS     |          |     QUIC      |
   |-------------|          |---------------|
   |     TCP     |          |     UDP       |
   |-------------|          |---------------|
   |     IP      |          |      IP       |
```

### 5.3 QUIC Features
*   **Adoption of TCP Approaches:** QUIC does not reinvent the wheel. It implements **error and congestion control** algorithms that parallel well-known TCP mechanisms (like Slow Start and Fast Retransmit), but implements them in the application space.
*   **One-RTT Connection:** By combining the transport handshake and the crypto handshake, QUIC establishes a connection much faster.

### 5.4 Comparison: Connection Establishment & HOL Blocking

**1. Connection Establishment:**
*   **TCP + TLS:** Requires **2 serial handshakes**. First, the TCP 3-way handshake (SYN, SYN-ACK, ACK), followed by the TLS handshake (ClientHello, ServerHello, etc.).
*   **QUIC:** Combines reliability, congestion control state, authentication, and encryption keys into **1 handshake**.

*   **Visual Aid: Handshake Comparison**
```text
      TCP + TLS (Sequential)               QUIC (Combined)
 Host A                  Host B      Host A                  Host B
   |  --- TCP SYN --->     |           |                       |
   |  <-- TCP SYNACK --    |           |                       |
   |  --- TCP ACK --->     |           |                       |
   |  --- TLS Hello ->     |           | -- QUIC Handshake --> |
   |  <-- TLS Hello --     |           | <-- (Crypto+Setup) -- |
   |         ...           |           |                       |
   v    (Data Starts)      v           v     (Data Starts)     v
```

**2. Head-of-Line (HOL) Blocking:**
*   **HTTP 1.1 / TCP:** If Packet 1 is lost, Packet 2 and 3 cannot be processed by the application until Packet 1 is retransmitted, even if they arrived perfectly. This is HOL blocking.
*   **HTTP/3 over QUIC:** Supports **Multiplexing**. Different streams (e.g., image 1, css file, text) are independent. A loss in the "image 1" stream does not block the "text" stream.

*   **Visual Aid: HOL Blocking**
```text
       TCP (Serial)                     QUIC (Parallel Streams)
 [Error!] [Pkt 2] [Pkt 3]          [Stream 1 Error!]
    |        |       |             [Stream 2 OK] --> Processed immediately
    |        |       |             [Stream 3 OK] --> Processed immediately
    v        v       v
 All wait for Pkt 1 recovery       Only Stream 1 waits
```

## 6.0 Chapter 3 Summary

This concludes the study of the Transport Layer (Network "Edge").

*   **Core Services:** Multiplexing/Demultiplexing, UDP (connectionless).
*   **TCP Deep Dive:**
    *   **Reliable Data Transfer:** Ensuring data arrives intact.
    *   **Flow Control:** Managing receiver limits.
    *   **Congestion Control:** Managing network health.
*   **Next Steps:** The course moves from the "edge" into the "core" of the network—the **Network Layer**, covering the Data Plane and Control Plane.

***

### **Key Terms & Definitions**

*   **Slow Start:** An algorithm to exponentially increase the sending rate (doubling cwnd per RTT) when a TCP connection begins or restarts after a timeout.
*   **Congestion Avoidance:** An algorithm that increases the congestion window linearly (additive increase) to safely probe for bandwidth limits.
*   **ssthresh (Slow Start Threshold):** A variable marking the boundary between the exponential growth of Slow Start and the linear growth of Congestion Avoidance.
*   **Fast Retransmit:** The mechanism of inferring packet loss via the arrival of 3 duplicate ACKs and retransmitting immediately, bypassing the RTO timer.
*   **Fast Recovery:** A mode following Fast Retransmit where the sender inflates `cwnd` for every duplicate ACK received, allowing flow to continue during recovery.
*   **Additive Increase:** Linearly increasing the congestion window (typically +1 MSS per RTT).
*   **Multiplicative Decrease:** Halving the congestion window upon detecting loss (mild congestion) to alleviate network pressure.
*   **Bottleneck Link:** The specific link in a network path with the lowest throughput or highest congestion, causing queues to fill and packets to drop.
*   **QUIC:** A modern application-layer transport protocol over UDP that integrates reliability, security, and congestion control to solve TCP inefficiencies like HOL blocking.
*   **Head-of-Line (HOL) Blocking:** A delay caused when the processing of a sequence of packets is halted because the first packet is missing or delayed.
