## 1.0 Review and RDT 3.0 Mechanisms

### 1.1 Summary of rdt1.0 ~ rdt3.0
The development of Reliable Data Transfer (RDT) protocols progresses from simple ideal scenarios to complex environments handling errors and loss.

*   **rdt1.0**: Assumes a **reliable channel** with no bit errors and no packet loss.
    *   **Mechanism**: **Nothing**. The sender sends data, and the receiver reads it. No feedback or flow control is required.
*   **rdt2.0**: The channel may have **bit errors** (e.g., flipped bits) but no packet loss.
    *   **Error Detection**: Uses a **checksum** to verify data integrity.
    *   **Receiver Feedback**: The receiver sends control messages: **ACK** (Acknowledgment) if data is good, **NAK** (Negative Acknowledgment) if data is corrupted.
    *   **Retransmission**: If a NAK is received, the sender retransmits the packet.
*   **rdt2.1**: Handles the "Garbled Feedback" problem of rdt2.0 (what if the ACK/NAK is corrupted?).
    *   Adds a **sequence number (1 bit)** to each segment (0 or 1). This allows the receiver to distinguish between a retransmission and a new packet.
*   **rdt2.2**: A "NAK-free" variant of rdt2.1.
    *   Instead of sending a NAK, the receiver sends a **Duplicate ACK** for the last correctly received packet.
    *   Example: Sender sends Pkt1. Receiver gets corrupted data. Receiver sends `ACK0`. Sender interprets `ACK0` (when expecting `ACK1`) as a NAK for Pkt1.
*   **rdt3.0**: The channel has **bit errors** and **packet loss**.
    *   **Timer**: The sender waits a reasonable amount of time for an ACK.
    *   **Retransmission upon timeout**: If the timer expires before an ACK is received, the packet is assumed lost and is retransmitted.
    *   Uses **ACK-only** logic (no NAKs).

```text
+----------+--------------------+---------------------------------------------------+
| Version  | Channel Conditions | Mechanism                                         |
+----------+--------------------+---------------------------------------------------+
| rdt1.0   | Reliable           | Nothing                                           |
+----------+--------------------+---------------------------------------------------+
| rdt2.0   | Bit Errors         | 1. Error detection (Checksum)                     |
|          | (No Loss)          | 2. Feedback (ACK/NAK)                             |
|          |                    | 3. Retransmission upon NAK                        |
+----------+--------------------+---------------------------------------------------+
| rdt2.1   | Same as 2.0        | 4. Sequence # (0 or 1) for each segment           |
+----------+--------------------+---------------------------------------------------+
| rdt2.2   | Same as 2.0        | Duplicate ACK = NAK (No explicit NAK used)        |
+----------+--------------------+---------------------------------------------------+
| rdt3.0   | Bit Errors +       | 5. Retransmission upon timeout                    |
|          | Packet Loss        | (No NAK, only ACK)                                |
+----------+--------------------+---------------------------------------------------+
```

***

### 1.2 Needed Mechanisms for Reliable Data Transfer (rdt3.0)
To achieve reliability over an unreliable channel (like IP), the transport layer implements specific mechanisms:

*   **Error detection**: The receiver must verify the integrity of the payload. Common algorithms include the **Internet Checksum** (used in UDP/TCP) and **CRC** (Cyclic Redundancy Check).
*   **Receiver feedback**: The sender needs to know the status of the packets.
    *   Used mechanism: **ACK + Sequence #**.
    *   **Duplicate ACK**: Acts as a negative acknowledgment, indicating the expected packet was not received.
*   **Timer & sequence #**:
    *   **Sequence Numbers**: Required to distinguish retransmissions from new data. For a stop-and-wait protocol, a 1-bit sequence number ($\ge 2$ states) is sufficient.
    *   **Timer**: Required to detect packet loss. The timeout interval must be reasonable—roughly the Round Trip Time (**RTT**). If too short, unnecessary retransmissions occur; if too long, reaction to loss is slow.
*   **Retransmission**: This is the core error-correction mechanism, triggered by either a **timeout** (loss detection) or a **duplicate ACK** (error/gap detection).

***

### 1.3 Stop-and-Wait Operation and Performance
*   **Operation**: rdt3.0 is a "stop-and-wait" protocol. The sender transmits one packet and refuses to send any more data until it receives an ACK for that specific packet.

```text
      Sender                                 Receiver
        |                                       |
  t=0   |-- Packet 0 -------------------------->| (Prop delay)
        |                                       |
        |  (Waiting for ACK...)                 |-- Checksum OK
        |                                       |   Send ACK 0
  RTT   |<--------------------------- ACK 0 ----|
        |                                       |
t=RTT+  |-- Packet 1 -------------------------->|
 L/R    |                                       |
        v                                       v
```

*   **Performance ($U_{sender}$)**:
    *   **Utilization**: Defined as the fraction of time the sender is actually busy pushing bits onto the wire.
    *   **Formula**:
        $$U_{sender} = \frac{L/R}{RTT + L/R}$$
        *   $L/R$: Time to transmit the packet (Length / Rate).
        *   $RTT$: Round Trip Time (propagation delay to receiver and back).
    *   **Example Calculation**:
        *   Link: 1 Gbps ($R = 10^9$ bps)
        *   Packet size: 8000 bits ($L = 8000$)
        *   RTT: 30ms (15ms each way)
        *   Transmission time ($L/R$) = $8000 / 10^9 = 8 \mu s = 0.008$ ms.
        *   $U_{sender} = 0.008 / (30.008) \approx 0.00027$.
    *   **Conclusion**: The sender is busy only 0.027% of the time. The rdt 3.0 protocol performance **stinks** because the physical resources are idle most of the time waiting for acknowledgments.

## 2.0 Pipelined Protocols

### 2.1 Pipelined Transfer Concept
*   **Pipelining**: To solve the low utilization of stop-and-wait, the sender allows multiple packets to be "in-flight" simultaneously. This means sending packet 2 and 3 before receiving the ACK for packet 1.
*   **Requirements**:
    *   **Sequence Numbers**: The range of sequence numbers must be increased (1 bit is no longer enough to distinguish multiple unique packets in flight).
    *   **Buffering**: Required at the sender (to store unacked packets for potential retransmission) and often at the receiver (to hold out-of-order packets).

```text
      [ Sender ]                                 [ Receiver ]
          |  Packet 1  --->                         |
          |    Packet 2  --->                       |
          |      Packet 3  --->                     |
          |                                         |
     (Multiple packets traversing the "pipe" of the network)
```

***

### 2.2 Pipelining Utilization
*   Pipelining dramatically improves throughput. If we allow 3 packets in the pipeline, the utilization increases by a factor of 3.
*   **Formula Adjustment**:
    $$U_{sender} = \frac{3 \cdot L/R}{RTT + L/R}$$

```text
Sender                                   Receiver
  |--- Pkt 1 --->                            |
  |--- Pkt 2 --->                            |
  |--- Pkt 3 --->                            |
  |                                          |-- Arrives, Send ACK 1
  |                                          |-- Arrives, Send ACK 2
  |<-- ACK 1 ----                            |-- Arrives, Send ACK 3
  | (Send Pkt 4)                             |
  v                                          v
  (Channel is kept much busier than Stop-and-Wait)
```

***

### 2.3 Types of Pipelined Protocols
There are two generic forms of pipelined protocols:

1.  **Go-Back-N (GBN)**:
    *   **Sender**: Can have up to **N** unacked packets in the pipeline.
    *   **Receiver**: Sends a **cumulative ACK**. This confirms the specific sequence number *and* all packets before it. It does **not** ACK a packet if there is a gap (out-of-order).
    *   **Timer**: The sender uses a single timer for the **oldest unacked packet**.
    *   **Recovery**: If the timer expires, the sender retransmits **all** unacked packets (Go Back to N and start again).

2.  **Selective Repeat (SR)**:
    *   **Sender**: Can have up to **N** unacked packets.
    *   **Receiver**: Sends an **individual ACK** for every correctly received packet, regardless of order.
    *   **Timer**: The sender maintains a unique timer for **each unacked packet**.
    *   **Recovery**: If a timer expires, the sender retransmits **only that specific** packet.

## 3.0 Go-Back-N (GBN) Details

### 3.1 GBN Example: No Loss
In a successful GBN scenario with a window size of $N=4$:
1.  Sender transmits 0, 1, 2, 3. Window is full.
2.  Receiver gets 0, sends ACK0.
3.  Sender receives ACK0. The window "slides" forward. It can now send packet 4.
4.  This creates a smooth flow of acknowledgments and new transmissions.

```text
 Sender Window (N=4)                  Transmission
 [0 1 2 3] 4 5 6 7 8      ---> Sends 0, 1, 2, 3
                          <--- Receiver sends ACK0
 0 [1 2 3 4] 5 6 7 8      ---> Sender receives ACK0, slides, sends 4
                          <--- Receiver sends ACK1
 0 1 [2 3 4 5] 6 7 8      ---> Sender receives ACK1, slides, sends 5
```

***

### 3.2 GBN Sender Mechanisms
*   **Window**: The sender maintains a window of size $N$, representing packets that are sent but not yet acknowledged.
*   **Sequence Number**: A $k$-bit sequence number is used in the header.
*   **Cumulative ACK**: GBN uses strictly cumulative acknowledgments. $ACK(n)$ means "I have received everything up to and including $n$".
    *   **Action**: Upon receiving $ACK(n)$, the window moves forward to begin at $n+1$.
*   **Timer**: Only one timer acts on the "base" of the window (the oldest sent-but-unacked packet).
*   **Timeout(n)**: If the timer expires for packet $n$, the sender assumes the pipeline is broken from $n$ onward and retransmits **packet $n$ and all higher sequence number packets** currently in the window.

```text
Sequence Number Space (Sender View):

|--- Already Ack'ed ---|--- Sent, Not Ack'ed ---|--- Usable ---|--- Not Usable ---|
                       ^                        ^              ^
                   send_base               nextseqnum    send_base + N
                       |_____ Window Size N ____|
```

***

### 3.3 GBN Receiver Mechanisms
*   **ACK-only**: The receiver discards out-of-order packets to keep implementation simple (no buffering required). It always sends an ACK for the highest **in-order** packet received.
    *   Requires tracking only one variable: `rcv_base`.
*   **Out-of-order packets**:
    *   If packet $n+1$ is received but packet $n$ is missing, $n+1$ is **discarded**.
    *   The receiver **Re-ACKs** packet $n-1$ (the last good in-order packet). This generates **Duplicate ACKs**.

```text
Sequence Number Space (Receiver View):

|--- Received & Ack'ed ---| |--- Not Received ---|
                          ^
                       rcv_base
```

***

### 3.4 GBN in Action (Loss Scenario)
*   **Scenario**: Packets 0, 1, 2, 3, 4, 5 are sent. Packet 2 is lost in transit.
*   **Receiver Actions**:
    1.  Receives 0, 1 $\rightarrow$ Sends ACK0, ACK1.
    2.  Wait for 2... (Lost).
    3.  Receives 3 $\rightarrow$ Out of order. Discard 3. **Resend ACK1**.
    4.  Receives 4 $\rightarrow$ Out of order. Discard 4. **Resend ACK1**.
    5.  Receives 5 $\rightarrow$ Out of order. Discard 5. **Resend ACK1**.
*   **Sender Actions**:
    1.  Receives ACK0, ACK1. Window slides.
    2.  Receives Duplicate ACK1s (ignores them in simplified GBN, or counts them for fast retransmit in TCP).
    3.  **Timeout for Packet 2**: The timer for Pkt 2 expires.
    4.  **Retransmission**: Sender resends Pkt 2 **AND** Pkt 3, 4, 5 (all current unacked packets).

```text
Sender                       Receiver
  |-- Pkt 2 (LOST) --X          |
  |-- Pkt 3 ------------------->| Detect Gap (Expect 2, got 3)
  |                             | Discard 3, Send ACK 1
  |<--------- ACK 1 ------------|
  |-- Pkt 4 ------------------->| Detect Gap
  |                             | Discard 4, Send ACK 1
  |<--------- ACK 1 ------------|
  |                             |
(Timeout Pkt 2)                 |
  |-- Resend Pkt 2 ------------>|
  |-- Resend Pkt 3 ------------>|
  |-- Resend Pkt 4 ------------>|
```

## 4.0 Selective Repeat (SR) Details

### 4.1 SR Overview & Window
Selective Repeat addresses the inefficiency of GBN (discarding valid packets just because one previous packet was lost).

*   **Receiver**:
    *   **Individually acknowledges** every correctly received packet.
    *   **Buffers** out-of-order packets. They are held until the missing packets arrive to fill the gap.
*   **Sender**:
    *   Maintains a timer for **each** unacked packet.
    *   When a timer expires, it retransmits **only** that specific packet.

```text
Sender Window:
[ . . . . | Sent/NoAck | Sent/NoAck | Usable | . . . ]
             (Timer A)    (Timer B)

Receiver Window:
[ . . . . | Expecting | Buffered | Expecting | . . . ]
```

***

### 4.2 SR Sender and Receiver Events
*   **Sender**:
    *   **Data from above**: Check if sequence number is within the window. If so, send.
    *   **Timeout(n)**: Resend packet $n$ only. Restart timer for $n$.
    *   **ACK(n) received**: Mark packet $n$ as received. If $n$ was the `send_base`, advance the window base to the next unacknowledged sequence number.
*   **Receiver**:
    *   **Packet n in [rcvbase, rcvbase+N-1]**:
        *   Send **ACK(n)**.
        *   If out-of-order: **Buffer** it.
        *   If in-order (fills the base): Deliver it (and any contiguous buffered packets) to the upper layer, then advance the window.
    *   **Packet n in [rcvbase-N, rcvbase-1]**:
        *   This is a duplicate of a previously finished packet.
        *   **Action**: Send **ACK(n)** anyway. (The sender likely didn't get the first ACK and is retransmitting; the receiver must confirm to stop the sender's timer).
    *   **Otherwise**: Ignore.

***

### 4.3 SR in Action (Loss Scenario)
*   **Scenario**: Packets 0-5 sent. Packet 2 is lost.
*   **Receiver**:
    *   Receives 0, 1 $\rightarrow$ Deliver, ACK.
    *   Receives 3, 4, 5 $\rightarrow$ **Buffer**, Send ACK3, ACK4, ACK5.
*   **Sender**:
    *   Receives ACKs for 0, 1 (Window moves).
    *   Receives ACKs for 3, 4, 5 (Marks them as green/received, but window cannot slide past 2).
    *   **Timeout for Pkt 2**: Retransmits **only** packet 2.
*   **Restoration**:
    *   Receiver gets Pkt 2.
    *   Receiver now has consecutive data: 2, 3, 4, 5.
    *   Delivers 2, 3, 4, 5 to the app. Advances window.
    *   Sends ACK2.

```text
Sender                          Receiver
  |-- Pkt 2 (LOST) --X             |
  |-- Pkt 3 ---------------------->| Buffer 3, Send ACK 3
  |<--------- ACK 3 ---------------| Mark 3 Acked
  |-- Pkt 4 ---------------------->| Buffer 4, Send ACK 4
  |<--------- ACK 4 ---------------| Mark 4 Acked
(Timeout Pkt 2)                    |
  |-- Resend Pkt 2 Only ---------->| Recv 2.
                                   | Deliver 2, 3, 4.
                                   | Send ACK 2
```

***

### 4.4 The Selective Repeat Dilemma
*   **Issue**: If the window size is too large relative to the sequence number space, the receiver cannot distinguish between a *new* packet and a *retransmission* of an old packet.
*   **Example**: Sequence numbers 0, 1, 2, 3 (Base 4). Window Size = 3.
    *   **Scenario A (No Loss)**: Receiver gets 0, 1, 2. Advances window. Expects new 3, **0**, 1. Sender sends new packet **0**. Receiver accepts as new. (Correct).
    *   **Scenario B (ACK Loss)**: Receiver gets 0, 1, 2. Advances window. ACKs lost. Sender times out and resends **old packet 0**. Receiver (expecting new 0) accepts old 0 as new data. (**Error**).
*   **Solution**: To avoid this ambiguity, the window size must be less than or equal to half the sequence number space.
    *   **Formula**: **Window Size $\le$ 1/2 Sequence Number Space**.

```text
Scenario A (New Data)      Scenario B (Duplicate Data masquerading as new)
Sender sends: Pkt 0        Sender sends: Old Pkt 0 (Retransmit)
Receiver Expects: New 0    Receiver Expects: New 0
Result: Correct Match      Result: False Match (Data duplication error)
```

## 5.0 TCP Reliable Data Transfer

### 5.1 Summary of Mechanisms Leading to TCP
TCP builds upon the theoretical RDT models:
1.  **rdt3.0**: Provided the foundation for handling errors (checksum) and loss (timeout/retransmit).
2.  **Go-Back-N**: Introduced the sliding window concept (pipelining) and cumulative ACKs.
3.  **Selective Repeat**: Introduced the concept of buffering out-of-order packets for efficiency.
*   **TCP**: Uses a hybrid approach. It uses cumulative ACKs (like GBN) but buffers out-of-order segments (like SR) to avoid unnecessary retransmissions.

### 5.2 Transition to Chapter 3 Highlights
The Transport Layer (Chapter 3) focuses on:
*   **UDP**: Lightweight, connectionless service.
*   **TCP Design**:
    *   **Segment structure**: The specific bits and bytes of the header.
    *   **Reliable data transfer**: Guaranteeing delivery.
    *   **Connection management**: Establishing (Handshake) and tearing down sessions.
    *   **Flow control**: Matching sender speed to receiver capacity.
    *   **Congestion control**: Matching sender speed to network capacity.

### 5.3 TCP RDT Components
TCP reliability relies on three specific implementation details:
1.  **Header fields**: Sequence number, ACK number, Window size.
2.  **Operations**: How the sender and receiver react to events (data arrival, timeouts).
3.  **Timeout estimation**: Algorithms to calculate the Retransmission Timeout (RTO) based on fluctuating RTT.

## 6.0 TCP Segment Structure

### 6.1 Detailed Structure
The TCP header is usually 20 bytes (without options).

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |     |C|E|U|A|P|R|S|F|                               |
| Offset| Res |W|C|R|C|S|S|Y|I|            Window             |
|       |     |R|E|G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

*   **Source & Dest Port**: Addressing for the application (process).
*   **Sequence Number**: Counts bytes (not segments).
*   **Acknowledgement Number**: The next byte expected (Cumulative ACK).
*   **Header Length (Data Offset)**: Length of the TCP header in 32-bit words.
*   **Flags**:
    *   **C, E**: Congestion notification (ECN).
    *   **A (ACK)**: Indicates the Acknowledgment field is valid.
    *   **R, S, F (RST, SYN, FIN)**: Used for connection setup and teardown.
*   **Receive Window**: The number of bytes the receiver is willing to accept (**Flow Control**).
*   **Checksum**: Verifies data integrity.

***

### 6.2 Checksum Details
*   The 16-bit checksum covers the TCP Header, the TCP Payload (data), and a **Pseudo IP Header**.
*   **Pseudo IP Header**: This binds the TCP segment to the IP layer ensuring it reached the correct IP address. It includes:
    *   Source IP & Destination IP.
    *   Protocol number (6 for TCP).
    *   TCP segment length (computed, not a field in the header).

***

### 6.3 Sequence Numbers and ACKs
*   **Sequence Numbers**: TCP views data as an unstructured, ordered stream of bytes. The Seq # in the header is the byte-stream number of the **first byte** in that segment.
*   **Acknowledgements**:
    *   The ACK number is the sequence number of the **next byte expected** from the other side.
    *   It is a **Cumulative ACK**.
    *   **Out-of-order handling**: The TCP spec doesn't strictly define this, leaving it to the implementor, but modern TCP buffers out-of-order segments to prevent retransmission of good data.

**Sequence Number Space Visualization:**
```text
[ Sent & Acked ] [ Sent, Not Acked ] [ Usable (In Window) ] [ Not Usable ]
                 ^                   ^
             SendBase           NextSeqNum
```

**Telnet Example (Piggybacking):**
Host A types 'C'. Host B echoes 'C'.
1.  **A sends**: Seq=42, ACK=79, Data='C'.
2.  **B sends**: Seq=79, ACK=43 (42+1), Data='C' (Echo).
    *   *Note*: B piggybacks the ACK for A's data onto its own data packet.
3.  **A sends**: Seq=43, ACK=80 (79+1). (Acknowledging the echo).

## 7.0 TCP Operational Mechanics

### 7.1 TCP Sender (Simplified)
*   **Event: Data received from app**:
    1.  Create a segment with Seq # = byte-stream number of the first data byte.
    2.  If the timer is not running, start it (associated with the **oldest unACKed segment**).
    3.  Expiration time = `TimeOutInterval`.
*   **Event: Timeout**:
    1.  Retransmit the specific segment that caused the timeout.
    2.  **Restart** the timer.
*   **Event: ACK received**:
    1.  Check if ACK > SendBase (does this acknowledge new data?).
    2.  If yes, advance `SendBase` (slide window).
    3.  If there are still unACKed segments, restart the timer.

***

### 7.2 TCP Receiver: ACK Generation
TCP receivers maximize efficiency by reducing unnecessary ACKs while ensuring the sender knows the status.

1.  **Arrival of in-order segment (All previous ACKed)**:
    *   Action: **Delayed ACK**. Wait up to 500ms. If another packet arrives, ACK both. If not, send ACK.
2.  **Arrival of in-order segment (One other segment pending ACK)**:
    *   Action: Immediately send a **single cumulative ACK** confirming both segments.
3.  **Arrival of out-of-order segment (Gap detected)**:
    *   Action: Immediately send a **duplicate ACK**. The ACK contains the seq # of the *next expected* byte (the bottom of the gap).
4.  **Arrival of segment filling gap**:
    *   Action: Immediately send a new ACK. If the gap is filled completely, this ACK will jump forward to the highest received byte.

***

### 7.3 Retransmission Scenarios

*   **Lost ACK Scenario**:
    *   Host B gets Seq 92, sends ACK 100.
    *   ACK 100 is lost.
    *   Host A times out -> Retransmits Seq 92.
    *   Host B sees duplicate 92 -> Discards data, Resends ACK 100.
*   **Premature Timeout Scenario**:
    *   Host A sends Seq 92, then Seq 100.
    *   Timer for 92 is too short. It fires before ACKs arrive.
    *   Host A retransmits 92.
    *   Host B (who has already received 92 and 100) sends ACK 120 (cumulative).
    *   Host A gets ACK 120. It knows both 92 and 100 arrived safely.
*   **Cumulative ACK Scenario**:
    *   Host A sends Seq 92, then Seq 100.
    *   Host B sends ACK 100 (Lost), then sends ACK 120 (Arrives).
    *   Host A receives ACK 120.
    *   Because ACKs are cumulative, ACK 120 implies ACK 100 was successful. **No retransmission occurs.**

```text
[Lost ACK]          [Premature Timeout]         [Cumulative ACK Save]

A      B            A            B              A             B
|--92->|            |----92----->|              |----92------>|
|      | (RX 92)    |----100---->|              |----100----->|
|<-ACK-| (Lost)     | (Timeout)  |              |<-ACK 100-X  | (Lost)
|      |            |--92 (Re)-->|              |             |
(Timeout)           |<-ACK 100---|              |             |
|--92->| (Dup)      |<-ACK 120---|              |<-ACK 120----|
|      |            |            |              |             |
|<-ACK-|            (A updates base to 120)     (A updates base to 120)
```

***

### **Key Terms**
*   **RDT (Reliable Data Transfer)**: The abstract concept of guaranteeing data delivery over unreliable networks.
*   **Checksum**: A value calculated from data used to detect errors.
*   **ACK (Acknowledgment)**: A signal sent by a receiver to indicate successful receipt of a packet.
*   **NAK (Negative Acknowledgment)**: A signal sent by a receiver to indicate a packet was received with errors.
*   **Sequence Number**: A number assigned to packets to identify them, detect duplicates, and ensure order.
*   **Stop-and-Wait**: A protocol where the sender waits for an ACK for the current packet before sending the next.
*   **Utilization ($U_{sender}$)**: The fraction of time a sender is actively transmitting data.
*   **Pipelining**: A technique allowing multiple unacknowledged packets to be in transit simultaneously.
*   **Go-Back-N (GBN)**: A pipelined protocol using cumulative ACKs and a single timer, retransmitting the whole window upon timeout.
*   **Selective Repeat (SR)**: A pipelined protocol using individual ACKs and timers, retransmitting only lost packets.
*   **Cumulative ACK**: An acknowledgment that confirms receipt of a sequence number and all preceding ones.
*   **Sliding Window**: The range of sequence numbers a sender is allowed to transmit without authorization.
*   **TCP Segment Structure**: The layout of the TCP header fields (Port, Seq, Ack, Flags, etc.).
*   **Pseudo IP Header**: Included in TCP checksum calculation to verify IP endpoint validity.
*   **Flow Control**: Mechanism (Receive Window) to prevent the sender from overwhelming the receiver's buffer.
*   **Delayed ACK**: Waiting briefly before sending an ACK to allow for piggybacking or cumulative acknowledgments.
*   **Duplicate ACK**: An ACK identical to the previous one, sent when an out-of-order packet arrives, indicating a gap.
*   **Piggybacking**: Sending an ACK inside a data packet going the other direction to save bandwidth.
