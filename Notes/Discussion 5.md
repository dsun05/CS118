# Lecture Notes: TCP Congestion Control

## 1.0 Introduction
### 1.1 Lecture Topics
This lecture focuses on the mechanisms TCP uses to handle network congestion. It serves as a review and expansion of Transport Layer reliability, specifically examining how the network prevents collapse when too much data is injected at once.
*   **Lecture Review:** TCP Congestion Control mechanisms (Tahoe, Reno, algorithms).
*   **Midterm Exam Review:** Coverage and logistics for the upcoming exam.

---

## 2.0 Midterm Reminder
### 2.1 Exam Details
*   **Date:** Tuesday, Feb 10, 2026
*   **Format:** In-class examination.
*   **Scope:** The exam is cumulative up to the Transport Layer.
    *   **Textbook:** Chapters 1, 2, and 3.
    *   **Assignments:** Project 1 and all homework assignments.
    *   **Materials:** Lecture notes and slides.

---

## 3.0 TCP Review: Sequence Number
### 3.1 Scenario and Calculation
To understand congestion control, we must first master how TCP tracks data using Sequence Numbers (SEQ) and Acknowledgment Numbers (ACK).

*   **Scenario:** Host A sends a TCP segment to Host B.
    *   **Sequence Number (SEQ):** 1234
    *   **Payload Length:** 120 bytes
*   **Question:** If Host B receives this successfully, what ACK number will it send back?
*   **Answer:** **1354**.
*   **Rationale:** TCP ACKs are cumulative and indicate the *next expected byte*.
    *   Calculation: $1234 \text{ (Start Byte)} + 120 \text{ (Bytes of Data)} = 1354$.
*   **General Rule:** Assuming no previous data loss, $ACK = SEQ + Length$.

### 3.2 Special Cases (SYN/FIN)
While data packets consume sequence numbers based on their payload size, control packets like **SYN** (Synchronize) and **FIN** (Finish) are special. They contain no payload data (0 bytes), yet they *must* be acknowledged to ensure the connection state is synchronized.

*   **Visual Aid: Client/Server State Transitions**
```text
      Client (Host A)                          Server (Host B)
      |                                              |
      |--- SYN (SEQ=x) ----------------------------->|  (Payload = 0)
      |                                              |
      |<-- SYN+ACK (SEQ=y, ACK=x+1) -----------------|  (Server consumes SEQ x)
      |                                              |
      |--- ACK (SEQ=x+1, ACK=y+1) ------------------>|  (Client consumes SEQ y)
      |                                              |
      |    ESTABLISHED                               |    ESTABLISHED
```
*   **Three-way handshake:** Even though the SYN packet has length 0, the receiver treats it as "logical" data of length 1. Therefore, the ACK number increments by 1.
*   **Connection Teardown:** Similarly, a FIN packet consumes one sequence number.
*   **Crucial Note:** Normal ACKs (that carry no data and no SYN/FIN flags) do *not* consume sequence numbers.

### 3.3 Duplicate ACKs (RFC 5681)
Duplicate ACKs are a critical signal for congestion control (specifically Fast Retransmit). However, not every repeated ACK number counts as a "Duplicate ACK" in the context of the algorithm.

*   **Visual Aid: Sequence Interaction (RFC 791 Example)**
```text
    Step   Host A (Sender)                    Host B (Receiver)
    ----   ---------------                    -----------------
     1.    (Send Seg 4)  SEQ=101, ACK=301 -->
     2.    (Send Seg 5)  SEQ=101, ACK=301 -->
```
*   **Question:** In the scenario above, Host A sends two segments with the same ACK number (301) to Host B. Does the second segment count as a "Duplicate ACK"?
*   **Answer:** **No.**
*   **RFC 5681 Definition:** Strictly speaking, a **Duplicate ACK** is an incoming acknowledgment that satisfies specific criteria indicating network issues (packet loss/reordering) rather than normal operation:
    1.  The receiver of the ACK (Host A) has outstanding (unacknowledged) data.
    2.  The incoming acknowledgment **carries no data** (payload length is 0).
    3.  The SYN and FIN bits are both off.
    4.  The ACK number is equal to the largest ACK number received so far.
    5.  The advertised window in the ACK has not changed.

---

## 4.0 TCP Review: Flow Control & Congestion Control
### 4.1 Comparison of Goals and Mechanisms
It is common to confuse Flow Control with Congestion Control because both limit the sender's transmission rate. However, they solve different problems.

| Feature | Flow Control | Congestion Control |
| :--- | :--- | :--- |
| **Primary Goal** | Prevent overwhelming the **Receiver**. | Prevent congesting the **Network** (routers/links). |
| **Constraint** | The receiver's processing speed and buffer space. | The bandwidth and traffic load on the intermediate links. |
| **Mechanism** | Explicit feedback via the **`rwnd` (Receive Window)** field in the TCP header. | Implicit inference by the sender using **`cwnd` (Congestion Window)** and **`ssthresh`**. |

---

## 5.0 TCP: Flow Control (Recap)
### 5.1 Key Mechanisms
*   **Purpose:** Ensures the sender does not send data faster than the receiver can write it to the application layer.
*   **The Receive Window (`rwnd`):** This value is sent in every TCP header from receiver to sender. It represents the amount of free space currently available in the receiver's operating system buffer.
*   **Sender Behavior:** The sender guarantees: `LastByteSent - LastByteAcked <= rwnd`.

---

## 6.0 TCP: Congestion Control
### 6.1 Why Congestion Control?
Without congestion control, the internet is susceptible to "Congestion Collapse."
*   **Packet Loss:** When router buffers fill up, packets are dropped.
*   **Retransmission:** Dropped packets trigger retransmissions, which adds *more* load to an already overloaded network.
*   **Reduced Throughput:** The network spends most of its resources sending duplicates rather than useful data. We need **dynamic** control to adjust to changing network loads.

### 6.2 History and Context
*   **October 1986:** The Internet experienced its first major congestion collapse. Connections between LBL and UC Berkeley dropped from **32 kbps** to **40 bps** (a 1000x reduction) due to infinite retransmission loops.
*   **1988:** **Van Jacobson** proposed the TCP congestion control algorithms (Tahoe/Reno).
    *   **End-to-End Principle:** The network (routers) does not explicitly tell the sender to slow down; the sender must infer congestion based on packet loss (ACK behavior).

### 6.3 Window-based Control
*   **Concept:** Limit the number of unacknowledged packets in the "pipe" (network) to a window size $W$.
*   **Throughput Formula:**
    $$\text{Allowed Rate (bps)} = \frac{W \times \text{Message Size}}{RTT}$$
*   **Trade-offs:**
    *   **If $W$ is too small:** The link is under-utilized (idle time waiting for ACKs).
    *   **If $W$ is too large:** The sender floods the link, causing buffer overflows and packet loss.

### 6.4 Effects of Congestion
1.  **Packet Loss:** Router queues overflow.
2.  **Increased Delay:** Packets sit in queues longer.
3.  **Wasted Bandwidth:** Upstream links transmit packets that are eventually dropped downstream, wasting resources.

### 6.5 Basics
*   **Goal:** Utilize available bandwidth efficiently without causing congestion, while sharing the link fairly among users.
*   **Two Windows:** The sender maintains two variables:
    1.  **`rwnd` (Receive Window):** Determined by the receiver (Flow Control).
    2.  **`cwnd` (Congestion Window):** Determined by the sender's estimate of network capacity (Network Control).
*   **Effective Window:** The sender is limited by the bottleneck of the two:
    $$W = \min(cwnd, rwnd)$$
*   *Note:* In most congestion scenarios, `cwnd` is the limiting factor.

### 6.6 TCP Tahoe and Reno Overview
TCP uses an "additive increase, multiplicative decrease" (AIMD) approach. It gently probes for more bandwidth but backs off aggressively when problems occur.
*   **Slow Start (SS):** Despite the name, this is exponential growth. Start with `cwnd = 1 MSS`. Double `cwnd` every Round Trip Time (RTT).
*   **Congestion Avoidance (CA):** Once a threshold (`ssthresh`) is reached, switch to linear growth (increase `cwnd` by 1 MSS per RTT).
*   **Detecting Loss:**
    *   **Timeout:** Severe congestion. Reset `cwnd` to 1.
    *   **3 Duplicate ACKs:** Mild congestion. Use **Fast Recovery** (TCP Reno).

### 6.7 Main Parts
The modern standard (Reno/NewReno) consists of:
1.  **Slow Start**
2.  **Congestion Avoidance**
3.  **Fast Retransmit / Fast Recovery**

### 6.8 Slow Start
*   **Objective:** Quickly ramp up to the available bandwidth.
*   **Initialization:** `cwnd = 1 MSS`.
*   **Growth:**
    *   **Per RTT:** $cwnd \leftarrow 2 \times cwnd$
    *   **Per ACK:** $cwnd \leftarrow cwnd + 1 \text{ MSS}$
*   **Transition:** Stop Slow Start and enter Congestion Avoidance when `cwnd` $\ge$ `ssthresh` (Slow Start Threshold).

### 6.9 Congestion Avoidance
*   **Objective:** Gently probe for the maximum limit without causing immediate loss.
*   **Condition:** `cwnd > ssthresh`.
*   **Growth:**
    *   **Per RTT:** $cwnd \leftarrow cwnd + 1 \text{ MSS}$ (Additive Increase)
    *   **Per ACK:** $cwnd \leftarrow cwnd + \frac{MSS \times MSS}{cwnd}$ (Approximation of 1 segment per full window acknowledged).

### 6.10 Packet Loss
TCP assumes that **Packet Loss = Network Congestion**.
*   **Method 1: RTO Timeout:** The timer expires before an ACK arrives. This implies severe congestion where no data is getting through.
*   **Method 2: 3 Duplicate ACKs:** The sender receives 3 redundant ACKs for the same sequence number. This implies that while one packet was lost, subsequent packets *did* arrive (triggering the ACKs). The network is still flowing, so the reaction is less severe than a timeout.

### 6.11 Fast Retransmit/Recovery
This algorithm prevents the "stop-and-wait" behavior caused by waiting for a timeout.
*   **On 3 Duplicate ACKs:**
    1.  **Adjust Threshold:** `ssthresh` $\leftarrow cwnd / 2$.
    2.  **Inflate Window:** `cwnd` $\leftarrow ssthresh + 3 \text{ MSS}$ (The +3 accounts for the 3 packets that left the network to trigger the dup ACKs).
    3.  **Retransmit:** Immediately resend the missing segment.
    4.  **Fast Recovery:**
        *   For every *additional* duplicate ACK received, increment `cwnd` by 1 MSS (inflate window further).
        *   Transmission continues if allowed by the new `cwnd`.
    5.  **On New ACK (Recovery Complete):**
        *   Set `cwnd` $\leftarrow ssthresh$.
        *   Enter **Congestion Avoidance**.
*   **On Timeout (RTO):**
    1.  `ssthresh` $\leftarrow cwnd / 2$.
    2.  `cwnd` $\leftarrow 1 \text{ MSS}$.
    3.  Enter **Slow Start**.

### 6.12 Summary
*   **AIMD:** Additive Increase (during CA), Multiplicative Decrease (cut `ssthresh` by half on loss).
*   **Fairness vs Efficiency:** TCP attempts to be fair to other connections while maximizing link usage.
*   **RFC 5681 Formula:** `cwnd` $\leftarrow cwnd/2 + 3MSS$ during Fast Recovery entry.

### 6.13 Algorithm Summary Table
*   **Visual Aid: Summary of 4 Algorithms**

```text
+-------------------+--------------------+--------------------------+----------------------------------+
| Algorithm         | Condition          | Design Pattern           | Operation                        |
+-------------------+--------------------+--------------------------+----------------------------------+
| Slow Start        | cwnd <= ssthresh   | Exponential Growth       | cwnd += 1 MSS per new ACK        |
|                   |                    | (Double per RTT)         |                                  |
+-------------------+--------------------+--------------------------+----------------------------------+
| Congestion        | cwnd > ssthresh    | Additive Increase        | cwnd += (MSS*MSS)/cwnd per ACK   |
| Avoidance         |                    | (Linear: +1 MSS per RTT) |                                  |
+-------------------+--------------------+--------------------------+----------------------------------+
| Fast Retransmit   | 3 Duplicate ACKs   | Multiplicative Decrease  | ssthresh = max(cwnd/2, 2*MSS)    |
|                   |                    |                          | cwnd = ssthresh + 3*MSS          |
|                   |                    |                          | Immediately retransmit lost pkt  |
+-------------------+--------------------+--------------------------+----------------------------------+
| Fast Recovery     | After Fast Retx    | Window Inflation         | On each dup ACK: cwnd += 1 MSS   |
|                   |                    |                          | On NEW ACK: cwnd = ssthresh      |
+-------------------+--------------------+--------------------------+----------------------------------+
| RTO Timeout       | Timer Expires      | Reset ("The Nuclear Option")| ssthresh = max(cwnd/2, 2*MSS) |
|                   |                    |                          | cwnd = 1 MSS                     |
|                   |                    |                          | Enter Slow Start                 |
+-------------------+--------------------+--------------------------+----------------------------------+
```

---

## 7.0 Step-by-Step Example (Scenario 1)
### 7.1 Example Setting
*   **Algorithms:** Slow Start, CA, Fast Retransmit/Recovery.
*   **Initial State:** `cwnd = 1`, `ssthresh = 4`, `rwnd` = infinite.
*   **Packet Loss:** Packet #5 is lost exactly once.
*   **Setup:** The sender transmits packets 1, 2, 3, 4, 5, 6...

### 7.2 Illustration
*   **Visual Aid: Scenario 1 Timeline (Loss of Packet 5 via Fast Retransmit)**

```text
Step | Action / State Change                                      | cwnd  | ssthresh
-----|------------------------------------------------------------|-------|----------
1.   | Start (Slow Start). Send Pkt 1.                            | 1     | 4
2.   | Receive ACK 1. cwnd increases by 1. Send Pkt 2, 3.         | 2     | 4
3.   | Receive ACK 2, 3. cwnd increases by 2. Send Pkt 4,5,6,7.   | 4     | 4
4.   | Receive ACK 4. cwnd becomes 5 ( > ssh). Enter CA.          | 5     | 4
     | Send Pkt 8, 9.                                             |       |
5.   | ** Packet 5 is LOST **                                     |       |
6.   | Pkt 6 arrives. Receiver sends ACK 4 (Dup #1).              | 5     | 4
     | Pkt 7 arrives. Receiver sends ACK 4 (Dup #2).              | 5     | 4
     | Pkt 8 arrives. Receiver sends ACK 4 (Dup #3).              | 5     | 4
7.   | ** 3 DUP ACKS DETECTED ** -> Fast Retransmit Pkt 5.        |       |
     | Update ssthresh = cwnd / 2 = 5 / 2 = 2.5 (~2).             |       | 2
     | Update cwnd = ssthresh + 3 = 2 + 3 = 5.                    | 5     | 2
8.   | ** Fast Recovery **                                        |       |
     | Receive ACK 4 (Dup #4 via Pkt 9). cwnd = 5 + 1 = 6.        | 6     | 2
     | Window allows sending new Pkt 10.                          |       |
9.   | Receive ACK 10 (Covers retransmitted Pkt 5 and others).    |       |
     | ** EXIT Fast Recovery **                                   |       |
     | Update cwnd = ssthresh.                                    | 2     | 2
10.  | Enter Congestion Avoidance.                                | 2     | 2
```

---

## 8.0 Step-by-Step Example (Scenario 2)
### 8.1 Example Setting
*   **Initial State:** `ssthresh = 6`, `cwnd = 1`.
*   **Losses:** Packet #7, Packet #13, Packet #17 (lost once each).
*   **Note:** Fast Retransmit triggers on the 3rd duplicate ACK. It does nothing on the 1st or 2nd.

### 8.2 Illustration
*   **Visual Aid: Scenario 2 Timeline (Complex Recovery)**

```text
Phase 1: The Loss of Packet 7 (Fast Retransmit)
-----------------------------------------------
1. Slow Start grows cwnd: 1 -> 2 -> 4 -> 8 (exceeds ssthresh 6).
2. During window of 8, Packet 7 is lost.
3. Subsequent packets (8, 9, 10...) trigger 3 Dup ACKs for Pkt 7.
4. Fast Retransmit Pkt 7.
   - New ssthresh = cwnd / 2 = 8 / 2 = 4 (Assume cwnd was ~8).
   - New cwnd = 4 + 3 = 7.
5. Fast Recovery acts on additional Dup ACKs until NEW ACK 7 arrives.
6. Upon receiving NEW ACK for 7 (actually ACK 12 cumulatively):
   - Exit Fast Recovery.
   - cwnd = ssthresh = 4.

Phase 2: The Loss of Packet 13 (Fast Retransmit)
------------------------------------------------
1. Congestion Avoidance (CA) active. cwnd grows linearly.
2. Packet 13 is lost.
3. 3 Dup ACKs received.
4. Fast Retransmit Pkt 13.
   - New ssthresh = cwnd / 2. (If cwnd was ~5, ssh=2).
   - New cwnd = 2 + 3 = 5.
5. Receive NEW ACK for 13 (ACK 17).
   - Exit Fast Recovery.
   - cwnd = ssthresh = 2.

Phase 3: The Loss of Packet 17 (Timeout)
----------------------------------------
1. Congestion Avoidance resumes. cwnd = 2.
2. Packet 17 is lost.
3. Because cwnd is small (2), there are not enough subsequent packets 
   sent to generate 3 Duplicate ACKs.
4. Sender waits... RTO Timer Expires.
5. ** TIMEOUT **
   - New ssthresh = cwnd / 2 = 2 / 2 = 1.
   - New cwnd = 1.
   - Enter Slow Start.

Final State (Question from slides):
- Upon receiving ACK #25 eventually (after re-building):
- cwnd has grown back to 4.
- ssthresh is 2.
```

---

### **Key Terms & Concepts**
*   **`rwnd` (Receive Window):** A flow control variable sent by the receiver indicating free buffer space.
*   **`cwnd` (Congestion Window):** A congestion control variable maintained by the sender limiting data in flight.
*   **`ssthresh` (Slow Start Threshold):** The "ceiling" for exponential growth. Above this, TCP grows linearly.
*   **Slow Start:** The phase of exponential window growth (doubling).
*   **Congestion Avoidance:** The phase of linear window growth (+1 MSS).
*   **Fast Retransmit:** Triggered by 3 duplicate ACKs; resends data immediately without waiting for a timer.
*   **Fast Recovery:** Allows the sender to keep transmitting new data while recovering from a single packet loss, avoiding a throughput-killing return to Slow Start.
*   **RTO (Retransmission Timeout):** The "nuclear option" for loss detection, resetting the connection to minimum speed.
