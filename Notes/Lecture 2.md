# Chapter 1: Computer Networks and the Internet (Part 2)

## 1.0 Packet Switching vs. Circuit Switching (Efficiency Analysis)

### 1.1 Efficiency Example and Probability
To understand why the Internet uses packet switching rather than the legacy circuit switching used in traditional telephony, we must analyze the efficiency of network resource sharing.

**Scenario:** Consider a network link with a capacity of **1 Mb/s** (1,000 kb/s). Users on this network generate data at a rate of **100 kb/s** when they are active. However, users are not active 100% of the time; they are "bursty." Specifically, a user is active only **10%** of the time ($p = 0.1$).

**Comparison:**
1.  **Circuit Switching:** This method reserves resources regardless of usage. Since each user requires 100 kb/s, the link can support exactly:
    $\frac{1,000 \text{ kb/s}}{100 \text{ kb/s}} = 10 \text{ users}$
    Even if these users are silent, the bandwidth is locked, preventing others from using it.

2.  **Packet Switching:** This method allocates resources on demand. Statistical multiplexing allows us to support many more users (e.g., $N = 35$) because it is statistically unlikely that all users will be active simultaneously.

**Probability Calculation:**
We can calculate the probability that the network becomes congested (i.e., more than 10 users are active simultaneously, exceeding the 1 Mb/s link capacity). Using the binomial distribution formula:

$$P(\text{congestion}) = P(active\_users > 10) = \sum_{x=11}^{35} \binom{35}{x} p^x (1-p)^{35-x}$$

*   Where $p = 0.1$ (probability of a user being active).
*   The result is less than **0.0004**.

**Conclusion:** Packet switching is vastly more efficient for bursty traffic. It supports over 3 times the number of users (35 vs 10) with a negligible probability of congestion.

```ascii
       Packet Switching Architecture
      +-------+
      | User 1| \
      +-------+  \      +--------+      ================
      | User 2| --\---->| Switch |====> | 1 Mbps Link  |---> Internet
      +-------+  /      +--------+      ================
      |  ...  | /      (Multiplexing)
      +-------+
```

### 1.2 Pros and Cons of Packet Switching
**Is packet switching a "slam dunk" winner?**
While highly efficient, it introduces specific challenges that network architects must manage.

*   **Advantages:**
    *   **Resource Sharing:** Ideally suited for "bursty" data patterns (e.g., web browsing where you request a page and then read it).
    *   **Simplicity:** No complex call setup phase is required before data transmission begins.
*   **Disadvantages:**
    *   **Congestion:** Because there is no reservation, excessive demand can lead to packet delays (queuing) and packet loss (buffer overflow).
    *   **Reliability:** Protocols are required to handle reliable data transfer and congestion control since the network itself makes no promises.

**Analogy:**
*   **Circuit Switching:** A restaurant requiring a reservation. Your table is empty and waiting for you even if you are late (guaranteed resource, potentially wasteful).
*   **Packet Switching:** A restaurant with no reservations. You simply walk in. It is more efficient for the restaurant (tables fill up fast), but you risk having to wait (queue) if it is busy.

### 1.3 Summary of Packet Switching
*   **Enables on-demand network sharing:** Resources are not wasted on inactive users. Those who have more to send can send more, provided bandwidth is available.
*   **Ideal for computer applications:** Computer traffic is inherently bursty (on/off behavior), unlike constant-bit-rate voice calls.
*   **Limitations:** It does not promise timely delivery. Sending too much data invariably leads to network congestion.

***

## 2.0 Internet Software Architecture

### 2.1 Core Components
Internet software architecture is built upon three pillars:
1.  **The Enabling Concept:** Packet switching (moving data in chunks).
2.  **The Service Model:** How the network promises to deliver data.
3.  **Layering:** The organization of software components.

### 2.2 Internet Service Model
The Internet operates on the **"Best Effort"** service model. This is arguably the simplest possible service model.
*   **Promise:** The network will try its best to deliver packets from source to destination.
*   **No Guarantees:** The network makes **no** guarantees regarding:
    1.  Successful delivery (packets may be lost).
    2.  Timing (packets may arrive late).
    3.  Order (packets may arrive out of sequence).
    4.  Throughput (no minimum speed is assured).

***

## 3.0 Protocols

### 3.1 Definition and Role
Protocols are the "rules of the road" for the Internet. They control the sending and receiving of messages between hardware devices.
*   **Examples:** HTTP (Web), TCP (Transport), IP (Network), 802.11 (WiFi).
*   **Standardization:** To ensure global interoperability, protocols are standardized by the **IETF** (Internet Engineering Task Force) in documents called **RFCs** (Request for Comments).

### 3.2 Human vs. Network Protocols
A protocol defines the format and order of messages exchanged between two or more communicating entities, as well as the actions taken on the transmission and/or receipt of a message or other event.

**Analogy:**
*   **Human Protocol:** I say "Hi" (initiate). You say "Hi" (acknowledge). I ask "Time?" (request). You say "2:00" (reply).
*   **Network Protocol (TCP Connection):** Client sends `Connection Request`. Server sends `Connection Response`. Client sends `Get URL`. Server sends `File`.

```ascii
    Human Protocol vs. Network Protocol (TCP)

      Human A             Human B            Client (PC)          Server
         |                   |                   |                   |
         | "Hi"              |                   |  TCP Conn. Req.   |
         |------------------>|                   |------------------>|
         |                   |                   |                   |
         | "Hi"              |                   |  TCP Conn. Resp.  |
         |<------------------|                   |<------------------|
         |                   |                   |                   |
         | "Time?"           |                   |  GET http://...   |
         |------------------>|                   |------------------>|
         |                   |                   |                   |
         | "2:00"            |                   |    <File Data>    |
         |<------------------|                   |<------------------|
         |                   |                   |                   |
      (Format, Order, Action)             (Format, Order, Action)
```

**Three Key Elements of a Protocol:**
1.  **Format:** What the message looks like (syntax).
2.  **Order:** The sequence in which messages are exchanged.
3.  **Actions:** What the entity does upon receiving a message.

### 3.3 Protocol Organization
The Internet involves hundreds of protocols working simultaneously. To manage this complexity, software is organized using **Layering**.

***

## 4.0 Layered Architecture

### 4.1 The Layering Concept
Layering decomposes the complex problem of reliable, global data delivery into smaller, manageable pieces.
*   **Explicit Structure:** It allows us to identify the relationship between complex system pieces (a reference model).
*   **Modularization:** It eases maintenance. If a protocol in one layer changes (e.g., upgrading from IPv4 to IPv6), the layers above and below it ideally do not need to change, provided the service interface remains the same.

### 4.2 Real-life Analogy: Air Travel
Air travel is a layered system. You perform a series of steps to travel, and each step relies on the successful completion of the step below it.

1.  **Ticket Layer:** Purchase ticket $\leftrightarrow$ Complain about ticket.
2.  **Baggage Layer:** Check bags $\leftrightarrow$ Claim bags.
3.  **Gate Layer:** Load (Board) $\leftrightarrow$ Unload (Deplane).
4.  **Runway Layer:** Takeoff $\leftrightarrow$ Landing.
5.  **Routing Layer:** Airplane flight path.

Each layer implements a service via its own internal actions and relies on the services provided by the layer immediately below it.

### 4.3 The Internet Protocol Stack (5 Layers)
The Internet uses a 5-layer stack (often referred to as the TCP/IP stack).

1.  **Application Layer:**
    *   **Function:** Supports network applications.
    *   **Protocols:** FTP, SMTP, HTTP.
    *   **Data Unit:** **Message**.
2.  **Transport Layer:**
    *   **Function:** Process-to-process data transfer.
    *   **Protocols:** TCP (Reliable), UDP (Unreliable).
    *   **Data Unit:** **Segment**.
3.  **Network Layer:**
    *   **Function:** Routing of datagrams from source to destination.
    *   **Protocols:** IP, Routing protocols.
    *   **Data Unit:** **Datagram**.
4.  **Link Layer:**
    *   **Function:** Data transfer between neighboring network elements (hops).
    *   **Protocols:** Ethernet, WiFi (802.11), PPP.
    *   **Data Unit:** **Frame**.
5.  **Physical Layer:**
    *   **Function:** Moving bits "on the wire."

```ascii
   The Internet Protocol Stack (Hourglass Shape)
   +-------------------------+
   |      Application        |  (HTTP, SMTP)
   +-------------------------+
   |       Transport         |  (TCP, UDP)
   +-------------------------+
   |        Network          |  (IP)
   +-------------------------+
   |          Link           |  (Ethernet, WiFi)
   +-------------------------+
   |        Physical         |  (Bits, Cables)
   +-------------------------+
```

### 4.4 Encapsulation & Decapsulation
As data moves from the source to the destination, it passes through the stack.
1.  **Encapsulation (Source):** The application sends a **Message**. The Transport layer adds a header ($H_t$) to create a **Segment**. The Network layer adds a header ($H_n$) to create a **Datagram**. The Link layer adds a header ($H_l$) to create a **Frame**.
2.  **Intermediate Devices:**
    *   **Switches:** Process up to the Link layer (Physical $\rightarrow$ Link).
    *   **Routers:** Process up to the Network layer (Physical $\rightarrow$ Link $\rightarrow$ Network).
3.  **Decapsulation (Destination):** As the data moves up the stack, headers are removed at each layer until the original message reaches the application.

***

## 5.0 Internet Performance Metrics

### 5.1 Overview
To evaluate if the Internet is working "well," we measure three specific metrics:
1.  **Throughput**
2.  **Loss**
3.  **Delay**

### 5.2 Throughput
**Definition:** The rate (bits/time unit) at which bits are successfully transferred between sender and receiver.
*   **Instantaneous Throughput:** The rate at a specific moment.
*   **Average Throughput:** The rate over a longer period.

**Fluid Analogy:** Think of data as fluid flowing through a pipe. The diameter of the pipe represents the bandwidth (bits/sec).

```ascii
     Server (Sender)                               Client (Receiver)
       +-------+                                     +-------+
       | Fluid |================>O==================>|Bucket |
       +-------+      Pipe 1            Pipe 2       +-------+
                    Rate: Rs          Rate: Rc
```

### 5.3 Bottleneck Link
The end-to-end throughput is constrained by the link with the *lowest* capacity along the path. This is called the **Bottleneck Link**.
*   If $R_s < R_c$, the average throughput is $R_s$.
*   If $R_s > R_c$, the average throughput is $R_c$.
*   **Throughput = min($R_s, R_c, R_1, R_2...$)**

In the real Internet, the core network (backbone) is very fast. The bottleneck is typically the **Access Network** (the link connecting the end user to the ISP).

### 5.4 Packet Loss
Routers utilize **queues (buffers)** to hold packets waiting to be transmitted. These buffers have finite capacity.
*   **Packet Loss** occurs when a packet arrives at a router but the queue is full. The router has no choice but to drop (discard) the packet.
*   Lost packets may be retransmitted by the previous node or source, or ignored entirely (depending on the protocol).

***

## 6.0 Packet Delay

### 6.1 Transmission Delay vs. Propagation Delay
It is crucial to distinguish between these two concepts.

1.  **Transmission Delay ($d_{trans}$):**
    *   The time required to push all the bits of the packet onto the wire.
    *   Formula: $L / R$
        *   $L$ = Packet length (bits)
        *   $R$ = Link Bandwidth (bits/sec)
    *   **Store-and-Forward:** A router must receive the *entire* packet ($L/R$ seconds) before it can begin transmitting it to the next link.

2.  **Propagation Delay ($d_{prop}$):**
    *   The time required for a bit to travel from the beginning of the link to the end.
    *   Formula: $d / s$
        *   $d$ = Length of physical link (meters)
        *   $s$ = Propagation speed ($\approx 2 \times 10^8$ m/sec for copper/fiber).

### 6.2 Queuing Delay ($d_{queue}$)
*   **Definition:** Time waiting at the output link for transmission.
*   **Factor:** Dependent on the congestion level of the router.
*   **Traffic Intensity:** Defined as $(L \cdot a) / R$, where $a$ is the average packet arrival rate.
    *   If Intensity $\approx 0$: Queuing delay is small.
    *   If Intensity $\rightarrow 1$: Queuing delay grows exponentially.
    *   If Intensity $> 1$: More work arrives than can be handled; delay becomes infinite (instability).

```ascii
      Exponential Growth of Queuing Delay
      |
    D |            |
    e |            |
    l |           /
    a |         _/
    y |________/
      |_________________
      0       Traffic Intensity (La/R)      1
```

### 6.3 Total Nodal Delay
The total delay a packet experiences at a single router hop ($d_{nodal}$) is the sum of four components:

$$d_{nodal} = d_{proc} + d_{queue} + d_{trans} + d_{prop}$$

1.  **$d_{proc}$ (Nodal Processing):** Time to check bit errors and determine the output link. Typically < milliseconds.
2.  **$d_{queue}$ (Queuing):** Time waiting for the link to become free.
3.  **$d_{trans}$ (Transmission):** Time to push bits onto the link.
4.  **$d_{prop}$ (Propagation):** Time to travel to the next node.

```ascii
          Router Node Delay Sources
            
   Incoming Link        +---------+        Outgoing Link
   -------------------->| Router  |-------------------->
      (Propagation)     +---------+     (Propagation)
                            |
           [Processing] <---+---> [Queuing] -> [Transmission]
             (Checks)              (Buffer)     (Push onto wire)
```

***

## 7.0 Real Internet Measurement: Traceroute

### 7.1 How Traceroute Works
**Traceroute** is a diagnostic program used to measure delay and discover the path from source to destination.
*   **Mechanism:** It sends small packets to the destination with increasing "Time To Live" (TTL) values.
*   **Probes:** For every router $i$ on the path, the sender sends three packets. When the packet reaches router $i$, the router discards it (due to TTL expiration) and sends a reply message back to the source.
*   **Timing:** The sender calculates the time elapsed between sending the packet and receiving the reply.

### 7.2 Interpreting Output
*   The output lists each router hop (IP address/name) and three delay measurements (e.g., `1 ms`, `2 ms`, `1 ms`).
*   **Asterisks (`* * *`)**: Indicate packet loss or a router that is configured not to reply.
*   **Latencies:** Large jumps in latency often indicate long physical distances, such as trans-oceanic links.

***

## 8.0 Chapter 1 Summary

We have established the "feel" and fundamental architecture of the Internet.
1.  **The Internet:** A network of end hosts and routers, divided into the **Network Edge** (users) and **Network Core** (routers).
2.  **Scale:** It uses a hierarchical structure (access networks, ISPs) to scale globally.
3.  **Architecture:**
    *   Relies on **Packet Switching** for efficiency.
    *   Uses a **Best Effort** service model.
    *   Organized into a **5-layer Protocol Stack**.
4.  **Performance:** Evaluated via **Throughput**, **Loss**, and **Delay** (Proc + Queue + Trans + Prop).
