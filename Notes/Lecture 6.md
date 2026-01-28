# Lecture 6: Transport Layer (TCP and Reliable Data Transfer)

## 1.0 Highlights of Chapter 3

### 1.1 Overview of Topics
This chapter focuses on the services provided by the transport layer, specifically examining the protocols that govern data delivery between applications running on different hosts.
*   **Introduction to transport-layer service:** Understanding how the transport layer provides logical communication between application processes running on different hosts.
*   **Introduction to UDP:** Brief coverage of the User Datagram Protocol (connectionless, unreliable).
*   **TCP protocol** focus areas: The majority of the lecture focuses on the **Transmission Control Protocol (TCP)**, detailing:
    *   **Overview and segment structure:** How data is packaged.
    *   **Reliable data transfer:** Ensuring data arrives correctly despite unreliable networks.
    *   **Connection management:** The handshaking processes for setup and termination.
    *   **Flow control:** Managing data rates to protect the receiver.
    *   **Congestion control:** Managing data rates to protect the network core.

***

## 2.0 TCP Protocol Overview

### 2.1 Key Characteristics
TCP is a complex protocol defined by several RFCs (793, 1122, 2018, 5681, 7323). Its fundamental attributes distinguish it from UDP:

*   **Point-to-point:** There is strictly one sender and one receiver. TCP does not support multicasting or broadcasting; it is a unicast protocol.
*   **Reliable, in-order data transfer:** TCP guarantees that the byte stream sent from the application is delivered to the receiver without error and in the correct order.
    *   **No "message boundaries":** The application writes a stream of bytes. TCP does not inherently respect the boundaries of the "writes" (e.g., one 500-byte write might be delivered in two segments).
*   **Full duplex data transfer:**
    *   Data can flow in both directions simultaneously over the same connection.
    *   **MSS (Maximum Segment Size):** This defines the maximum amount of application data (excluding headers) that can be placed into a TCP segment.
*   **Connection-oriented:** Before any application data is exchanged, a handshaking process (exchange of control messages) occurs to initialize the state variables (sequence numbers, buffers) of both the sender and receiver.
*   **Flow controlled:** A mechanism where the sender limits its rate so it does not overwhelm the **receiver's** buffer or processing capability.
*   **Congestion controlled:** A mechanism where the sender throttles its rate to avoid overwhelming the **Internet** (the network infrastructure between sender and receiver).

### 2.2 TCP Segment Structure
The TCP segment consists of a header and a data field. The standard header is 20 bytes long, but it can be larger if options are used.

*   *Visual Aid: Diagram of the 32-bit TCP Header structure*

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgement Number                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Head | Not |C|E|U|A|P|R|S|F|                                 |
|  Len  | Used|W|C|R|C|S|S|Y|I|         Receive Window          |
|       |     |R|E|G|K|H|T|N|N|                                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (Variable Length)                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Application Data (Variable)                 |
|                                                               |
```

### 2.3 Concrete TCP Header Fields
Each field in the header exists to support essential TCP features:
*   **Reliable data transfer:** Supported by `Sequence Number`, `Acknowledgement Number`, and `Checksum`.
*   **Connection setup and close-down:** Supported by flags like `SYN` (Synchronize) and `FIN` (Finish).
*   **Flow control:** Supported by the `Receive Window` field, which tells the sender how much free buffer space the receiver has.
*   **Congestion control:** Supported by flags like `CWR` and `ECE` (Explicit Congestion Notification), though much of congestion control is inferred from packet loss and delays.

***

## 3.0 Mechanisms for Reliable Data Transfer (rdt)

### 3.1 Context and Goal
*   **Goal:** The objective is to create a **logical end-to-end reliable transport** layer. To the application, it should appear as if a dedicated, error-free pipe exists between processes.
*   **The Reality:** The underlying network layer (IP) is unreliable.
*   **Packet delivery misbehaviors** to handle:
    *   **Bit errors:** 0s turned to 1s or vice versa during transmission.
    *   **Dropped packets (loss):** Routers discarding packets due to congestion.
    *   **Out-of-order delivery:** Packets arriving in a different sequence than sent.
    *   **Duplicate copies:** The same packet arriving more than once.
    *   **Long delayed delivery:** Packets trapped in queues for extended periods.

### 3.2 Service Abstraction vs. Implementation
We distinguish between what the user sees (abstraction) and what actually happens (implementation).

*   *Visual Aid: Reliable Service Abstraction vs. Implementation*

```text
    (A) Service Abstraction                (B) Service Implementation

    [Sending Process]                      [Sending Process]
           |                                      |
           | Data                                 | Data
           v                                      v
  +------------------+                   +------------------+
  |    Reliable      |                   | Sender-side rdt  |
  |    Channel       |==================>|    Protocol      |
  +------------------+                   +------------------+
           |                                      |
           | Data                                 | udt_send()
           v                                      v
    [Receiving Process]                  +------------------+
                                         | Unreliable Chan. | (Network)
                                         +------------------+
                                                  |
                                                  | rdt_rcv()
                                                  v
                                         +------------------+
                                         | Receiver-side    |
                                         |  rdt Protocol    |
                                         +------------------+
                                                  |
                                                  | deliver_data()
                                                  v
                                         [Receiving Process]
```
*   **Implementation:** Requires a protocol on both the sender side and receiver side to manage the unreliable channel (network) using control messages (like ACKs).

### 3.3 Protocol Design Approach
*   We develop the **reliable data transfer protocol (rdt)** incrementally.
*   **Assumption:** We consider unidirectional data transfer for simplicity (data flows A to B), though control info (ACKs) must flow back (B to A).
*   **Tool:** We use **Finite State Machines (FSM)** to specify the behavior.
    *   **States:** Represented by circles (e.g., "Wait for call from above").
    *   **Transitions:** Arrows representing changes in state triggered by events (e.g., "packet received").
    *   **Actions:** Operations performed during a transition (e.g., "send ACK").

***

## 4.0 rdt Protocols: Progressive Design

### 4.1 rdt1.0: Reliable Transfer over a Reliable Channel
*   **Assumption:** The underlying channel is perfectly reliable. There are no bit errors and no packet loss.
*   **Mechanism:** Because the channel is perfect, no flow control or error control is needed.
    *   **Sender:** Simply accepts data from the application, packages it, and sends it (`udt_send`).
    *   **Receiver:** Simply waits for data, receives it (`rdt_rcv`), extracts it, and delivers it.
*   *Visual Aid: FSM for rdt1.0*

```text
      Sender FSM                      Receiver FSM
    +------------+                  +--------------+
    |  Wait for  |                  |   Wait for   |
    |  call from |                  |   call from  |
    |   above    |                  |    below     |
    +-----+------+                  +------+-------+
          |                                ^
          | rdt_send(data)                 | rdt_rcv(pkt)
          | packet=make_pkt(data)          | extract(pkt,data)
          | udt_send(packet)               | deliver_data(data)
          |                                |
          v                                |
    (Loops back to state)           (Loops back to state)
```

### 4.2 The "Stop and Wait" Scenario
Before tackling errors, we adopt a "Stop and Wait" flow strategy:
*   The sender transmits **one** packet.
*   The sender **stops** and **waits** for confirmation from the receiver before sending the next packet.
*   This is inefficient but simplifies protocol design for error handling.

### 4.3 rdt2.0: Channel with Bit Errors
*   **Problem:** The channel is no longer perfect; it may flip bits (corruption), though packets are not lost.
*   **Mechanisms introduced:**
    1.  **Error detection:** A **checksum** (e.g., Internet Checksum) is added to the packet header to allow the receiver to detect corruption.
    2.  **Feedback:**
        *   **ACK (Acknowledgement):** Receiver explicitly tells sender "I received the packet successfully."
        *   **NAK (Negative Acknowledgement):** Receiver explicitly tells sender "The packet had errors."
    3.  **Retransmission:** If the sender receives a NAK, it retransmits the previous packet.
*   *Visual Aid: Timing Diagram (rdt2.0)*

```text
    (Successful Transfer)           (Transfer with Error)
    Sender        Receiver          Sender        Receiver
      |              |                |              |
      |-- Pkt0 ----->|                |-- Pkt0 ----->| (Corrupted!)
      |              |                |              |
      |<---- ACK ----|                |<---- NAK ----|
      |              |                |              |
      |-- Pkt1 ----->|                |-- Pkt0 ----->| (Retransmission)
      |              |                |              |
```

### 4.4 The Fatal Flaw of rdt2.0
*   **Issue:** rdt2.0 assumes the feedback channel (ACK/NAK) is perfect. What if the **ACK or NAK is corrupted**?
*   **Sender Dilemma:** The sender receives a garbled response. It doesn't know if the receiver got the data (ACK) or needs a resend (NAK).
*   **Failed Solution:** If the sender simply retransmits upon receiving garbled feedback, the receiver might receive **duplicate** packets. If the original packet was actually received correctly (but the ACK was garbled), the receiver accepts the retransmission as *new* data, leading to data duplication errors.

### 4.5 rdt2.1: Handling Corrupted Feedback
*   **Solution to Duplicates:** The sender adds a **sequence number** to every packet.
*   **Receiver Logic:** The receiver checks the sequence number. If it matches the packet it just received, it recognizes it as a **duplicate** and discards it (but still sends an ACK to resynchronize the sender).
*   **Sequence Number Size:** In a "Stop and Wait" protocol, we only need to distinguish between the "current" packet and the "previous" packet. Therefore, a **1-bit sequence number** (0 and 1) is sufficient.
*   *Visual Aid: rdt2.1 Action*

```text
    Sender (Seq=0)      Receiver (Expects 0)
         |                       |
         |---- Pkt0 (Seq=0) ---->| (Received OK)
         |                       |
         |<----- ACK (Corrupt) --|
         |                       |
    (Sender doesn't know status, Resends)
         |                       |
         |---- Pkt0 (Seq=0) ---->| (Checks Seq#, sees 0 again)
         |                       | (Identifies as Duplicate!)
         |                       | (Discards data, Resends ACK)
         |<------- ACK ----------|
```

### 4.6 rdt2.2: A NAK-free Protocol
*   Functionality is identical to rdt2.1 (handles bit errors and garbled feedback), but it removes the NAK message to simplify the protocol.
*   **Mechanism:**
    *   The receiver **only uses ACKs**.
    *   When a packet is received correctly, the receiver sends an ACK including the **sequence number** of that packet (e.g., `ACK 1`).
    *   If a packet is corrupted, the receiver resends the ACK for the **last correctly received packet** (e.g., `ACK 0`).
    *   **Duplicate ACK:** When the sender receives two ACKs for packet 0 (a duplicate ACK), it interprets this as a NAK for packet 1 and triggers a retransmission of packet 1.

***

## 5.0 rdt3.0: Channels with Errors and Loss

### 5.1 New Assumption: Packet Loss
*   Real networks can lose packets entirely (buffer overflows at routers).
*   Mechanisms from rdt2.x (checksum, seq #, ACK, retransmission) are not enough. If a packet is lost, the sender never gets an ACK/NAK and waits forever (deadlock).

### 5.2 Handling Lost Packets
*   **Mechanism:** **Timeout and Retransmit**.
*   The sender starts a **countdown timer** upon sending a packet.
*   The sender waits a "reasonable" amount of time for an ACK.
*   **Timeout Event:** If the timer expires before an ACK is received, the sender assumes loss and **retransmits**.
*   **Handling False Alarms (Delays):**
    *   If a packet is merely delayed (not lost), the timeout may fire prematurely, causing a retransmission.
    *   This creates a duplicate packet at the receiver.
    *   Fortunately, the **sequence numbers** (introduced in rdt2.1) already handle duplicates; the receiver will simply discard the second copy.

### 5.3 rdt3.0 in Action
Visualizing the behavior of rdt3.0 under different network conditions:

*   *Visual Aid: Timing diagrams for rdt3.0*

**(a) No Loss**
```text
    Sender           Receiver
      |-- Pkt0 -------->|
      |<------- ACK0 ---|
      |-- Pkt1 -------->|
      |<------- ACK1 ---|
```

**(b) Packet Loss**
```text
    Sender           Receiver
      |-- Pkt0 -------->|
      |<------- ACK0 ---|
      |-- Pkt1 --X      | (Loss)
    [Timer Starts]      |
    [Timeout!]          |
      |-- Pkt1 -------->| (Resend)
      |<------- ACK1 ---|
```

**(c) ACK Loss**
```text
    Sender           Receiver
      |-- Pkt1 -------->| (Recv OK)
      |      X----- ACK1| (ACK Lost)
    [Timeout!]          |
      |-- Pkt1 -------->| (Duplicate!)
      |                 | (Discard, send ACK1)
      |<------- ACK1 ---|
```

**(d) Premature Timeout**
```text
    Sender           Receiver
      |-- Pkt1 -------->|
    [Timer Starts]      |
      |          /----- | (ACK delayed)
    [Timeout!]  /       |
      |-- Pkt1 / ------>| (Duplicate!)
      |       /         | (ACK1 sent)
      |<--ACK1          |
      |<------- ACK1 ---| (Sender ignores dup ACK)
```

### 5.4 Summary of rdt Versions
The progression of mechanisms required as the channel becomes more unreliable:

*   *Visual Aid: Summary Table*

| Version | Channel Assumption | Mechanisms Added |
| :--- | :--- | :--- |
| **rdt1.0** | Reliable channel | None required. |
| **rdt2.0** | Bit errors (no loss) | **Checksum** (error detection), **ACK/NAK** (feedback), **Retransmission**. |
| **rdt2.1** | Bit errors + Corrupted feedback | **Sequence Numbers (0/1)** to handle duplicates caused by retransmitting on ambiguous feedback. |
| **rdt2.2** | Same as 2.1 | **Duplicate ACKs** replace NAKs. (NAK-free). |
| **rdt3.0** | Bit errors + **Packet Loss** | **Timeout** timer. Retransmission triggered by time, not just feedback. |

***

### **Key Terms & Concepts**
*   **TCP (Transmission Control Protocol):** A connection-oriented, reliable transport protocol that provides flow and congestion control.
*   **MSS (Maximum Segment Size):** The largest amount of data (in bytes) that can be placed in a single TCP segment payload.
*   **Reliable Data Transfer (rdt):** The concept of providing an error-free, in-order data stream on top of an unreliable network.
*   **Finite State Machine (FSM):** A mathematical model of computation used to design protocol logic, consisting of states, transitions, and actions.
*   **Stop and Wait:** A simple flow control strategy where the sender waits for an acknowledgement for the current packet before sending the next one.
*   **Checksum:** A value calculated from the packet content used to detect bit errors (corruption) upon reception.
*   **ACK (Acknowledgement):** A control message sent by the receiver to indicate positive receipt of data.
*   **NAK (Negative Acknowledgement):** A control message indicating that a received packet was corrupted.
*   **Sequence Number:** A number assigned to packets to allow the receiver to identify duplicates and order segments.
*   **Duplicate ACK:** In rdt2.2 and TCP, receiving a second ACK for the same packet acts as a signal that the *next* packet was not received correctly (implicit NAK).
*   **Timeout:** An event generated by a timer when an expected ACK does not arrive within a specific duration, triggering retransmission.
