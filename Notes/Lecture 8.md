## **1.0 TCP Retransmission Scenarios**

This section details how TCP handles data loss and acknowledgment (ACK) anomalies to ensure reliable data transfer.

### **1.1 Retransmission Scenarios and Lost ACKs**
TCP relies on acknowledgments to confirm delivery. When these mechanisms fail, specific retransmission scenarios occur.

*   **Lost ACK Scenario:**
    *   **The Event:** Host A sends a segment (e.g., Sequence 92). Host B receives it successfully and sends `ACK=100`. However, this ACK is lost in the network.
    *   **The Reaction:** Host A's countdown timer expires because it never received the ACK. Host A retransmits the same segment (Seq 92).
    *   **Resolution:** Host B receives the retransmission. It recognizes the data is a duplicate (based on sequence number) and discards the packet to prevent data corruption. Crucially, Host B *re-sends* `ACK=100` to inform Host A that the data is safe.

*   **Premature Timeout:**
    *   **The Event:** Host A sends a segment. The network delay is high (but no packet is lost), causing Host A's timer to expire before the ACK arrives.
    *   **The Reaction:** Host A retransmits the segment thinking it was lost.
    *   **Resolution:** Host B receives the duplicate and sends an ACK. Meanwhile, the original ACK finally arrives at Host A.
    *   **Outcome:** This results in wasted bandwidth due to unnecessary retransmission, though data integrity is maintained.

*   **Cumulative ACK:**
    *   **The Concept:** TCP ACKs are cumulative, meaning `ACK=120` confirms delivery of *all* bytes up to 119.
    *   **The Scenario:** Host A sends two segments: Seq 92 and Seq 100. Host B sends `ACK=100` (for the first) and `ACK=120` (for the second). The first ACK (`ACK=100`) is lost.
    *   **Resolution:** If `ACK=120` arrives at Host A before the timer for Seq 92 expires, Host A knows that *both* segments arrived safely. No retransmission occurs.

```text
* Visual Aid: TCP Retransmission Timing Diagrams *

Scenario 1: Lost ACK            Scenario 2: Premature Timeout
   Host A         Host B           Host A          Host B
      |              |                |               |
      |--Seq=92----->|                |--Seq=92------>|
      |              |                |               |
(Timer|      X-------| (ACK Lost)     |       /-------| (ACK Delayed)
Start)|              |          (Timer|      /        |
      | (Timeout)    |          Expires)    /         |
      |--Seq=92----->|                |----/--------->|
      |              |                |   / (Dup Rcvd)|
      |              |                |<--            |--ACK=100--
      |<--ACK=100----|                |               |
```

***

### **1.2 TCP Fast Retransmit**
Waiting for a timeout is often too slow, as timeouts are generally set conservatively long to avoid premature firing. **Fast Retransmit** allows the sender to detect loss and resend data *before* the timer expires.

*   **Trigger Mechanism:** The sender receives **3 additional ACKs** (duplicate ACKs) for the same sequence number.
*   **The Logic:**
    *   If packets 1, 2, 3, 4, and 5 are sent, and packet 2 is lost:
    *   Receiver gets 1 (sends ACK 2).
    *   Receiver gets 3 (detects gap, sends ACK 2 again).
    *   Receiver gets 4 (detects gap, sends ACK 2 again).
    *   Receiver gets 5 (detects gap, sends ACK 2 again).
    *   Sender sees the initial ACK + 3 duplicates. This pattern strongly suggests packet 2 was lost, not just delayed.
*   **Action:** The sender immediately retransmits the unacknowledged segment (the one requested by the duplicate ACKs) without waiting for the timeout.

```text
* Visual Aid: Fast Retransmit Timeline *

Host A                            Host B
  |---Seq=92, 8 bytes-------------->|
  |---Seq=100, 20 bytes (LOST)--X   |
  |---Seq=120, 15 bytes------------>| (Out of order arrival)
  |<--ACK=100-----------------------| (ACK for Seq 92)
  |                                 |
  |<--ACK=100-----------------------| (Duplicate #1: Saw Seq 120)
  |---Seq=135... ------------------>|
  |<--ACK=100-----------------------| (Duplicate #2: Saw Seq 135)
  |---Seq=150... ------------------>|
  |<--ACK=100-----------------------| (Duplicate #3: Saw Seq 150)
  |                                 |
  | (3 Dups Received -> Resend 100) |
  |---Seq=100 (Retransmission)----->|
```

***

## **2.0 TCP Round Trip Time and Timeout**

TCP must determine how long to wait for an ACK before assuming a packet is lost. This duration is the **TimeoutInterval**.

### **2.1 Timeout Estimation Basics**
*   **The Challenge:**
    *   **Too Short:** Causes premature timeouts and unnecessary retransmissions (bloating the network).
    *   **Too Long:** The sender waits too long to repair dropped packets, reducing throughput.
    *   **Requirement:** The timeout must be longer than the **RTT** (Round Trip Time), but RTT varies constantly due to network load.
*   **SampleRTT:** The measured time from segment transmission until ACK receipt. TCP ignores retransmissions when calculating this to avoid ambiguity.

### **2.2 Computing Estimated RTT**
Since `SampleRTT` fluctuates wildly, TCP uses an **Exponential Weighted Moving Average (EWMA)** to calculate a smooth, stable value called `EstimatedRTT`.

*   **Formula:** `EstimatedRTT = (1- α)*EstimatedRTT + α*SampleRTT`
*   **Significance of α (Alpha):** Typically set to **1/8 (0.125)**. This means the new sample only affects the average by 12.5%, while 87.5% relies on historical data. This prevents a single outlier sample from skewing the timer.

```text
* Visual Aid: SampleRTT vs EstimatedRTT *

 RTT
  ^
  |      *   (SampleRTT - Jagged/Volatile)
  |     / \ *
  |    *   \
  |   /     *       ----------------- (EstimatedRTT - Smooth)
  |  *       \     /
  |           \___/
  |_______________________> Time
```

### **2.3 Setting the Timeout Interval**
The timeout requires a "safety margin" to account for variance. If the RTT is stable, the margin is small; if the RTT fluctuates heavily, the margin increases.

*   **DevRTT:** Measures the deviation (variability) of the RTT. It is also calculated using EWMA.
    *   Formula: `DevRTT = (1-β)*DevRTT + β*|SampleRTT - EstimatedRTT|`
    *   Typical **β = 1/4**.
*   **TimeoutInterval Calculation:**
    *   Formula: `TimeoutInterval = EstimatedRTT + 4*DevRTT`
*   **Calculation Order (RFC6298):** You must update `DevRTT` using the *old* `EstimatedRTT` first, and then update `EstimatedRTT`.

### **2.4 Karn’s Algorithm and Timer Management**
*   **Karn’s Algorithm:** Do **not** update `SampleRTT` using retransmitted segments.
    *   *Reasoning:* If you send a packet, it times out, you resend it, and then get an ACK, you don't know if the ACK is for the first (delayed) transmission or the second one. Using this data would corrupt the RTT estimate.
*   **Managing the RTO (Retransmission TimeOut) Timer:**
    1.  Start timer when a packet with new data is sent.
    2.  Stop timer when all outstanding data is acknowledged.
    3.  Restart timer when an ACK arrives for *some* (but not all) outstanding data.
*   **Binary Exponential Backoff:** Upon a timeout, TCP **doubles** the TimeoutInterval (`RTO = RTO * 2`). This aggressive slowing down prevents the sender from flooding a congested network with retransmissions.

### **2.5 Summary of TCP Reliable Data Transfer**
*   **Mechanisms:** Uses Sequence Numbers, Cumulative ACKs, and Checksums.
*   **Recovery:** Uses Fast Retransmit (3 dup ACKs) for speed and Timeouts (with exponential backoff) as a failsafe.
*   **Calculations:** Sophisticated RTT estimation (Mean + Variance) ensures the timer adapts to network conditions.

***

## **3.0 TCP Connection Management**

Before data transfer begins, TCP establishes a logical connection.

### **3.1 Connection Setup (Handshake)**
*   **Goal:** The client and server must agree they are willing to communicate and synchronize their **Initial Sequence Numbers (ISNs)**.
*   **Variables:** They also establish buffer sizes (`rcvBuffer`).
*   **States:** `LISTEN` (Server waiting), `SYNSENT` (Client initiated), `SYN RCVD` (Server received request), `ESTAB` (Connection active).
*   **The 3-Way Handshake:**
    1.  **SYN:** Client sends segment with `SYN=1` and a random sequence number `Seq=x`.
    2.  **SYNACK:** Server allocates buffers, sets `SYN=1`, `ACK=1`, acknowledges client's sequence (`ACKnum=x+1`), and sets its own sequence `Seq=y`.
    3.  **ACK:** Client allocates buffers, sets `ACK=1`, and acknowledges server's sequence (`ACKnum=y+1`). The connection is now Established.

```text
* Visual Aid: 3-Way Handshake State Transitions *

      Client State                                Server State
        [CLOSED]                                    [LISTEN]
           |                                           |
    (Send SYN, Seq=x) --------------------------> (Receive SYN)
      [SYN_SENT]                                       |
           |                                  (Send SYNACK, Seq=y,
           | <---------------------------      ACK=x+1, SYN=1)
    (Receive SYNACK)                              [SYN_RCVD]
           |                                           |
    (Send ACK, ACK=y+1) ------------------------> (Receive ACK)
       [ESTAB]                                      [ESTAB]
```

### **3.2 Connection Close-Down**
A TCP connection consists of two one-way streams. Each side must close its stream independently using a **FIN** (Finish) segment.

*   **Mechanism:** Sending a segment with `FIN=1` indicates "I have no more data to send."
*   **4-Way Handshake:**
    1.  Client sends FIN -> enters `FIN_WAIT_1`.
    2.  Server receives FIN, sends ACK -> enters `CLOSE_WAIT`. Client enters `FIN_WAIT_2`.
    3.  Server sends its own FIN -> enters `LAST_ACK`.
    4.  Client receives FIN, sends ACK -> enters `TIMED_WAIT`. Server receives ACK -> `CLOSED`.
*   **TIMED_WAIT:** The client stays in this state for **2 * Maximum Segment Lifetime (MSL)** (typically 2 minutes).
    *   *Purpose:* To ensure the final ACK reached the server. If the ACK was lost, the server will resend its FIN, and the client needs to be there to re-ACK it. It also ensures old packets die out in the network.

***

## **4.0 TCP Flow Control**

### **4.1 Concept and Mechanism**
*   **Goal:** To prevent the sender from overwhelming the **receiver's** buffer. This is a speed-matching service between the sender's transmission rate and the receiving application's ability to read data.
*   **RcvBuffer:** The OS allocates a buffer (default often 4096 bytes). If the application reads slowly, this buffer fills up.
*   **Mechanism:**
    *   The Receiver advertises the amount of free space remaining in the buffer via the **rwnd** (Receive Window) field in the TCP header.
    *   **The Constraint:** The Sender must ensure: `Unacknowledged Data (In-Flight) <= rwnd`.

```text
* Visual Aid: Receive Window (rwnd) Concept *

Receiver's Buffer (RcvBuffer)
|-----------------------------------------------|
|  Buffered Data (Waiting for App)  |   rwnd    |
|-----------------------------------|-----------|
^                                   ^           ^
Start of Buffer                 Current     End of Buffer
                                Write Pointer
```

### **4.2 Flow Control Summary**
*   The sender dynamically adjusts its window size `N` based on the `rwnd` value returned in every ACK.
*   If `rwnd = 0`, the sender stops sending large segments (but may send small "probes" to see if space has opened up).

***

## **5.0 Internet Congestion Control Principles**

### **5.1 Nature of Congestion**
*   **Definition:** Congestion occurs when "too many sources are sending too much data too fast for the **network** to handle."
*   **Difference from Flow Control:**
    *   **Flow Control:** 1 Sender vs. 1 Receiver (Receiver is the bottleneck).
    *   **Congestion Control:** Many Senders vs. The Network (Routers/Links are the bottleneck).
*   **Symptoms:** Packet loss (router buffers overflow) and long queuing delays.

### **5.2 Causes and Costs**
*   **Infinite Buffers Scenario:** If router buffers were infinite, no packets would be lost, but **queuing delays** would approach infinity as packets wait in line.
*   **Finite Buffers Scenario:** In reality, buffers are finite. When they fill, routers drop packets.
    *   **The Cost:** The sender must retransmit. This wastes network resources because the link capacity is used to transport the same packet multiple times (original + retransmission).

***

## **6.0 TCP Congestion Control Implementation**

### **6.1 General Approach**
*   **End-to-End Control:** The network (routers) does not explicitly tell TCP to slow down. The TCP **Sender** must infer congestion based on observed behavior.
*   **Inference:** If packets are lost (Timeout or 3 Duplicate ACKs), TCP assumes the network is congested and slows down.

### **6.2 Key Issues in TCP Congestion Control**
1.  **Detection:** How does the sender know there is congestion?
2.  **Reaction:** How does the sender adjust its rate?
3.  **Improvement:** How to maximize throughput without causing collapse?

### **6.3 Issue 1: Detection**
*   **Assumption:** Wired networks are reliable. Therefore, **packet loss is essentially always caused by buffer overflow** (congestion), not corruption.
*   **Signals:**
    *   **Timeout:** Severe congestion (no data is getting through).
    *   **3 Duplicate ACKs:** Mild congestion (some packets getting through, but one dropped).

### **6.4 Issue 2: Reaction (AIMD Rule)**
TCP uses a **Congestion Window (`cwnd`)** to control the rate: `Rate ≈ cwnd / RTT`. To maintain stability and fairness, TCP follows the **AIMD** principle:

*   **Additive Increase (AI):** If no loss occurs, TCP "probes" for more bandwidth.
    *   Increases `cwnd` by **1 MSS** (Maximum Segment Size) every RTT.
    *   Graphically, this creates a linear ramp-up (slope of 1).
*   **Multiplicative Decrease (MD):** If loss is detected, TCP assumes the network is saturated.
    *   It cuts `cwnd` by **50%** (halves the rate).
    *   This clears the router queues quickly.
*   **Fairness:** AIMD ensures that multiple TCP connections sharing a link will eventually converge to equal bandwidth usage.

```text
* Visual Aid: TCP Sawtooth Behavior *

 Rate (cwnd)
  ^
  |          /
  |         /  | (Loss detected: Cut by 50%)
  |        /   |
  |   /   /    V
  |  /|  /     /
  | / | /     /
  |/  |/
  |___________________> Time
  (Linear Increase)
```

### **6.5 Issue 3: Improvements and Algorithms**
Standard AIMD is too slow to start and not aggressive enough for severe timeouts. Modern TCP (RFC 5681) uses four phases:

1.  **Slow Start:**
    *   Used at connection startup.
    *   Instead of linear increase, `cwnd` grows **exponentially** (doubles every RTT) to fill the pipe quickly.
    *   Transitions to AIMD when it hits a threshold (`ssthresh`).
2.  **Congestion Avoidance:**
    *   The standard Additive Increase phase (linear growth).
3.  **Fast Retransmit/Fast Recovery:**
    *   Triggered by **3 Duplicate ACKs**.
    *   Performs Multiplicative Decrease (halves `cwnd`) but skips Slow Start. This is "Fast Recovery."
4.  **Retransmission upon Timeout:**
    *   Triggered by a **Timeout** (no ACKs).
    *   This implies heavy congestion.
    *   TCP acts drastically: **Resets `cwnd` to 1 MSS** and re-enters Slow Start.

***

### **Key Terms & Definitions**
*   **SampleRTT:** The raw, measured time taken for a packet to travel to the receiver and the ACK to return.
*   **EstimatedRTT:** An average of SampleRTT values, smoothed using EWMA to filter out noise.
*   **DevRTT:** A measure of how much the SampleRTT typically deviates from the average; used to calculate safety margins.
*   **Fast Retransmit:** The mechanism of resending a lost packet immediately after receiving 3 duplicate ACKs, bypassing the timer.
*   **Karn’s Algorithm:** The rule that prohibits calculating RTT samples from retransmitted packets to avoid ambiguity.
*   **RTO (Retransmission TimeOut):** The calculated duration the sender waits before retransmitting.
*   **RWND (Receive Window):** The amount of free buffer space advertised by the receiver for flow control.
*   **CWND (Congestion Window):** The limit on "in-flight" data imposed by the sender based on its estimate of network congestion.
*   **AIMD (Additive Increase Multiplicative Decrease):** The dominant algorithm for congestion control stability (Add 1 MSS per RTT, Cut half on loss).
*   **MSS (Maximum Segment Size):** The largest amount of data (payload) TCP will put in a single segment.
*   **Slow Start:** The initial phase of TCP where `cwnd` increases exponentially to quickly find the network's capacity.
