# Chapter 7: Wireless and Mobile Networks

Welcome to the comprehensive study notes for Chapter 7, focusing on Wireless and Mobile Networks. These notes expand upon the foundational concepts of wireless networking, the physical characteristics of wireless links, link-layer protocols like Wi-Fi, and the architectural complexities of managing mobility on the Internet. 

***

## 1.0 Chapter 7: Wireless and Mobile Networks

### 1.1 Textbook Roadmap for Chapter 7
The study of wireless and mobile networks is broadly divided into two interrelated yet distinct domains: **Wireless** communications and **Mobility** management. 

In the **Wireless** domain, we explore:
*   The fundamental characteristics of wireless links and how they differ from traditional wired networks.
*   **CDMA** (Code Division Multiple Access), a foundational channel partitioning scheme.
*   **Wi-Fi** (IEEE 802.11), the dominant standard for wireless Local Area Networks (LANs).
*   Cellular networks, specifically the architecture and evolution of 4G and 5G technologies.

In the **Mobility** domain, we examine how networks handle devices that physically move:
*   The overarching principles of mobility management.
*   Practical mobility management implemented in 4G/5G cellular networks.
*   **Mobile IP**, the standard protocol for handling device mobility at the network layer.
*   The cascading impact of physical mobility on higher-layer protocols, such as TCP and application-level routing.

### 1.2 Highlights of Chapter 7
By the conclusion of this chapter, you will understand:
1.  The core components that define a wireless network.
2.  The unique physical anomalies and characteristics of wireless links.
3.  The mathematical and conceptual foundation of **CDMA**.
4.  How **Wi-Fi** functions as a wireless equivalent to Ethernet LANs.
5.  How continuous network connectivity is maintained via **Mobile IP** when a device transitions between physical locations.

***

## 2.0 Wireless Networks

### 2.1 #1. What is a wireless network?
A wireless network replaces physical cables with electromagnetic signaling over the air. It consists of several foundational elements:

*   **Wireless Hosts**: These are the end-system devices running user applications, such as laptops, smartphones, and Internet of Things (IoT) devices. It is crucial to distinguish between *wireless* and *mobile*; a desktop computer connected to Wi-Fi is a wireless host, but it is typically stationary (non-mobile).
*   **Base Station**: This is a centralized network node typically connected to the rigid, wired infrastructure. It acts as a relay, responsible for bridging the gap by sending and receiving packets between the wired network and the wireless hosts within its coverage area. Common examples include cellular towers and 802.11 access points.
*   **Wireless Link**: The physical medium (radio frequency band) used to connect mobile devices to a base station or directly to each other. Because multiple devices share the same wireless medium, a **multiple access protocol** is heavily utilized to coordinate link access and prevent data collisions. These links vary greatly in transmission rates, effective distances, and frequency bands.
*   **Infrastructure Mode**: In this dominant network topology, all wireless hosts communicate exclusively through a base station that ties them to the broader wired network. If a mobile device moves out of range of one base station and into another, it performs a **handoff**—seamlessly changing its associated base station to maintain connectivity.
*   **Ad Hoc Mode**: In this decentralized topology, there are no base stations. Nodes directly transmit to other nodes within their immediate link coverage. The nodes must self-organize into a functional network, participating cooperatively to route data among themselves.

```text
Visualizing a Wireless Network Topology:

     [Wired Network Infrastructure (Internet)]
               |                  |
        +------+------+    +------+------+
        |Base Station |    |Base Station |  <-- (Infrastructure Mode)
        |  (e.g., AP) |    | (Cell Tower)|
        +------+------+    +------+------+
          /    |    \             |     \
       ( )    ( )   ( )          ( )    ( )
       H1     H2    H3           H4     H5  <-- Wireless Hosts (Laptops, Phones)
       
       
       ( ) <-----> ( )
        |  Ad Hoc   |      <-- (Ad Hoc Mode: No Base Station)
       ( ) <-----> ( )
```

### 2.2 Wireless network taxonomy
Wireless networks can be categorized based on their structural reliance on infrastructure and the number of logical "hops" required to reach a destination. 

```text
+---------------------+---------------------------------+---------------------------------+
|                     |           Single Hop            |          Multiple Hops          |
+---------------------+---------------------------------+---------------------------------+
|   Infrastructure    | Host connects directly to a     | Host may have to relay through  |
|     (e.g., APs)     | base station (WiFi, cellular)   | several wireless nodes to       |
|                     | which connects to the larger    | connect to the larger Internet. |
|                     | Internet.                       | (e.g., Wireless Mesh Networks)  |
+---------------------+---------------------------------+---------------------------------+
|  No Infrastructure  | No base station, no connection  | No base station, no connection  |
|                     | to the larger Internet. Devices | to larger Internet. May relay   |
|                     | communicate directly.           | through nodes to reach others.  |
|                     | (e.g., Bluetooth, Ad hoc nets)  | (e.g., MANET, VANET)            |
+---------------------+---------------------------------+---------------------------------+
```
*(Note: **MANET** stands for Mobile Ad Hoc Network, and **VANET** stands for Vehicular Ad Hoc Network.)*

### 2.3 Example of no infrastructure, single-hop wireless
A ubiquitous real-world example of a single-hop, no-infrastructure wireless network is the Apple AirDrop feature. 

AirDrop bypasses the need for an access point by creating a highly localized, peer-to-peer connection. The underlying protocol utilizes a combination of technologies for security and efficiency:
1.  **Discovery**: It first utilizes low-energy Bluetooth to broadcast and discover nearby devices that are available to receive files.
2.  **Data Transfer**: Once a target is identified and connection parameters are negotiated, AirDrop shifts to **Wi-Fi Direct** (a localized **ad-hoc** link, referred to by Apple as multipeer connectivity). This creates a secure, high-speed, peer-to-peer Wi-Fi connection strictly between the two devices to swiftly transfer the payload.

### 2.4 Popular Wireless Network Technologies
Wireless technologies are engineered to optimize the trade-off between maximum data rate and operational range. 

```text
Data Rate vs Range Mapping of Popular Technologies:

  46 Gbps |   [802.11be/bn (Wi-Fi 7/8)]
  10 Gbps |               [5G Advanced]
          |
 9.6 Gbps |   [802.11ax (Wi-Fi 6)]
 6.9 Gbps |     [802.11ac (Wi-Fi 5)]
 150 Mbps |                               [4G LTE]
  54 Mbps |   [802.11g]
  11 Mbps |   [802.11b]
   2 Mbps | [Bluetooth]
          |_______________________________________________________
               Indoor          Outdoor        Midrange       Long range
              (10-30m)        (50-200m)      (200m-4Km)      (4Km-15Km)
```

***

## 3.0 Wireless Link Characteristics

### 3.1 2. Wireless link characteristics
Unlike wired networks running over predictable copper or fiber-optic cables, wireless communications utilize the open atmosphere as a shared medium. This introduces severe physical challenges:

*   **Decreased signal strength (Signal Attenuation)**: Electromagnetic radio signals inevitably attenuate (weaken) as they propagate through matter and empty space. This is known as **path loss**. The farther the signal travels, the weaker it becomes.
*   **Interference from other sources**: Because the airwaves are unregulated in certain bands (like the popular 2.4 GHz Industrial, Scientific, and Medical band), multiple distinct technologies share the exact same frequencies. A wireless host may suffer **interference** from other Wi-Fi networks, cellular devices, or even electromagnetic noise generated by household appliances (e.g., microwaves or electric motors).
*   **Multipath propagation**: As radio waves ripple outward, portions of the signal reflect off physical objects (walls, desks, the ground). These reflected signals travel slightly longer paths and arrive at the destination receiver at slightly different times, causing overlapping, self-interfering signal echoes.

To measure link quality, engineers use the **Signal-to-Noise Ratio (SNR)**, which compares the level of the desired signal to the level of background noise. A higher SNR means it is easier for a receiver to mathematically extract the correct data from the physical noise.

**SNR versus BER (Bit Error Rate) Tradeoffs:**
The **Bit Error Rate (BER)** represents the probability that a transmitted bit will be received incorrectly.
*   *Given a specific physical layer protocol*: Increasing the transmission power increases the SNR, which inherently decreases the BER. However, more power drains batteries faster and causes more interference for neighbors.
*   *Given a specific SNR*: Network interfaces dynamically choose different physical layer modulation techniques based on current conditions. For example, if a mobile device moves further from the base station (lowering SNR), the wireless card will automatically "downshift" to a more robust, slower modulation technique (like BPSK) to maintain an acceptable BER. If the SNR is high, it will shift to a fast, dense modulation technique (like QAM256) to maximize throughput.

### 3.2 3. Implications on link-layer operations
Because wireless networks share an open medium across varied geographic topography, unique link-layer problems arise that do not exist on wired Ethernet. 

**The Hidden Terminal Problem:**
Physical obstacles can prevent nodes from "hearing" one another. Consider three nodes: A, B, and C. Node B is in the middle. Node A and C want to talk to B, but a physical mountain exists between A and C. Node A senses the channel is idle and starts transmitting to B. Node C, unable to hear A due to the mountain, also senses the channel is idle and transmits to B. The two signals collide and corrupt each other at B. Nodes A and C are "hidden" from one another.

**Signal Attenuation:**
Even without a physical mountain, distance alone can cause the hidden terminal problem. If Node A and Node C are too far apart for their signals to reach one another, but both can reach the centrally located Node B, their signals will still collide at B.

```text
Diagram: Hidden Terminal & Signal Attenuation

         (Mountain / Distance)
       /   \ 
     /       \            <-- A and C cannot hear each other!
   (A)       (C)          <-- Both transmit simultaneously.
     \       / 
      \     / 
        (B)               <-- Signals collide here at the receiver!
```

***

## 4.0 Code Division Multiple Access (CDMA)

### 4.1 3. Code Division Multiple Access (CDMA)
While technologies like Time Division (TDMA) slice channels by time, and Frequency Division (FDMA) slice channels by frequency, **Code Division Multiple Access (CDMA)** slices channels mathematically. 

Although largely phased out in modern cellular architecture (completed shutdown in US public mobile networks around 2022-2024 in favor of 4G LTE/5G), CDMA is a foundational concept in signal processing. In CDMA, all users broadcast on the *exact same frequency* at the *exact same time*. To avoid catastrophic collisions, a unique mathematical "code" (a chipping sequence) is assigned to each user.

*   **Encoding**: The sender transmits the inner product of their original data bit and their unique chipping sequence. 
*   **Decoding**: The receiver calculates the summed inner product of the incoming data stream and the specific sender's chipping sequence. Because the codes assigned to users are strictly **orthogonal** (mathematically independent), interference from other senders effectively cancels out to zero when the receiver applies the correct decoding sequence.

```text
CDMA: Two-Sender Interference ASCII Model

Sender 1 Data (d1) --> [ * c1 (chipping code 1) ] --+
                                                    |
Sender 2 Data (d2) --> [ * c2 (chipping code 2) ] --+--> Channel Sums Together
                                                         Z = (d1*c1) + (d2*c2)
                                                                 |
                                                                 V
Receiver uses c1 to decode Sender 1:                  [ Z * c1 ] -> Yields d1!
(Interference from Sender 2 is nullified because c1 and c2 are orthogonal)
```
This elegant mathematical property allows the receiver to extract Sender 1's original data seamlessly from a completely mixed sum of channel data.

***

## 5.0 Wi-Fi: IEEE 802.11 Wireless LAN

### 5.1 4. Wi-Fi: IEEE 802.11 Wireless LAN
The IEEE 802.11 family of standards dictates how Wi-Fi operates. All 802.11 standards fundamentally utilize the **CSMA/CA** (Carrier Sense Multiple Access with Collision Avoidance) protocol for media access control, and they support both infrastructure and ad-hoc architectures.

```text
Evolution of IEEE 802.11 Standards:
+-------------------+------+-----------+-------+-------------------+
| Standard          | Year | Max Rate  | Range | Frequency Bands   |
+-------------------+------+-----------+-------+-------------------+
| 802.11b           | 1999 | 11 Mbps   | 30m   | 2.4 GHz           |
| 802.11g           | 2003 | 54 Mbps   | 30m   | 2.4 GHz           |
| 802.11n (Wi-Fi 4) | 2009 | 600 Mbps  | 70m   | 2.4, 5 GHz        |
| 802.11ac (Wi-Fi 5)| 2013 | 3.47 Gbps | ~35m  | 5 GHz             |
| 802.11ax (Wi-Fi 6)| 2019 | 14 Gbps   | ~35m  | 2.4, 5, 6 GHz     |
| 802.11be (Wi-Fi 7)| 2024 | 46 Gbps   | ~30m  | 2.4, 5, 6 GHz     |
| 802.11bn (Wi-Fi 8)| 2026 | 46 Gbps   | ~45m+ | 2.4, 5, 6 GHz     |
+-------------------+------+-----------+-------+-------------------+
```

### 5.2 4. 802.11 LAN architecture
In infrastructure mode, the fundamental building block of an 802.11 network is the **Basic Service Set (BSS)** (commonly referred to as a "cell"). A BSS contains:
1.  One or more wireless hosts.
2.  A central base station known as an **Access Point (AP)**.

Multiple APs are often wired into a central switch or router, combining several BSSs into a wider network capable of routing traffic to the broader Internet.

```text
802.11 LAN Architecture:

       BSS 1                                   BSS 2
  +-------------+                         +-------------+
  |    (H1)     |                         |    (H3)     |
  |      \      |      [Internet]         |      /      |
  |       (AP1)-+----------|--------------+-(AP2)       |
  |      /      |      [Switch/Router]    |      \      |
  |    (H2)     |                         |    (H4)     |
  +-------------+                         +-------------+
```

### 5.3 4. 802.11: Channels, association
The 802.11 frequency spectrum is divided into specific channels. The network administrator designates which channel an AP will operate on. However, if neighboring APs use overlapping channels, heavy interference can occur.

When a new wireless host arrives, it must first **associate** with an AP before it can access the network:
1.  **Scanning**: The host scans the frequency channels, listening for *beacon frames*. These frames contain vital network information, including the AP's name (the SSID) and its MAC address.
2.  **Selection**: The host evaluates signal strengths and selects a specific AP to associate with.
3.  **Authentication**: The host authenticates (often using WPA2/WPA3 protocols).
4.  **IP Allocation**: Finally, the host typically runs DHCP (Dynamic Host Configuration Protocol) over the new wireless link to obtain an IP address mapped to the AP's subnet.

### 5.4 4. 802.11: passive/active scanning
Hosts can discover APs through two distinct scanning methodologies:

*   **Passive Scanning**: The host sits on a channel and silently listens. APs periodically broadcast Beacon frames. Upon hearing a desirable beacon, the host replies with an Association Request.
*   **Active Scanning**: The host actively broadcasts a Probe Request frame into the void. Any APs in range that hear this probe will respond directly with a Probe Response frame. The host then chooses an AP and sends an Association Request.

```text
Passive Scanning:                  Active Scanning:
(AP1)    (H1)    (AP2)             (AP1)    (H1)    (AP2)
  |       |       |                  |       |       |
  |Beacon>|       |                  |<Probe>|       | 
  |       |<Beacon|                  |       |<Probe>|
  |       |       |                  |       |       |
  |<AssocReq      |                  |ProbeRes>      |
  |       |       |                  |       |<ProbeRes
  |AssocRes>      |                  |       |       |
  |       |       |                  |<AssocReq      |
                                     |       |       |
                                     |AssocRes>      |
```

### 5.5 4. IEEE 802.11: multiple access
Because 802.11 shares an open medium, collisions occur if two nodes transmit simultaneously. To manage this, 802.11 uses **CSMA/CA** (Carrier Sense Multiple Access with Collision **Avoidance**).

Unlike wired Ethernet (which uses CSMA/CD to *detect* collisions by listening while sending), wireless hardware *cannot* easily detect collisions. The signal an antenna transmits is immensely powerful compared to the weak, faded signal it receives; listening while talking over wireless is akin to trying to hear a whisper while shouting. Furthermore, the *hidden terminal problem* guarantees that a node cannot sense all incoming collisions anyway. Therefore, the goal of 802.11 is strictly to *avoid* collisions before they happen.

### 5.6 4. IEEE 802.11 MAC Protocol: CSMA/CA
The protocol utilizes strict inter-frame spacing timers and random backoffs to prevent nodes from talking over one another.

**Sender Logic:**
1.  Sense the channel. If it is idle for a designated period called the **DIFS (Distributed Inter-Frame Space)**, transmit the entire frame immediately.
2.  If the channel is busy, defer transmission and start a random backoff timer. The timer strictly counts down *only* while the channel remains idle. When the timer hits zero, transmit. If the transmission fails (no ACK received), double the random backoff interval and try again.

**Receiver Logic:**
Upon successfully receiving a non-corrupted frame, the receiver waits a very brief period called the **SIFS (Short Inter-Frame Space)** and then immediately broadcasts an Acknowledgment (`ACK`). The ACK is mandatory because the sender has no other way to confirm that a collision or hidden terminal interference didn't destroy the packet.

```text
CSMA/CA Timing Diagram:

Sender   |--DIFS--|==============[ DATA ]==============|
                                                       |
Receiver |        |                                    |--SIFS--|==[ ACK ]==|
```

### 5.7 4. Collision Avoidance
To further combat the hidden terminal problem for large, important packets, 802.11 optionally allows senders to "reserve" the channel using tiny reservation packets.

1.  The sender transmits a brief **Request-to-Send (RTS)** frame to the AP. RTS frames are so small that collisions are rare and cheap to recover from.
2.  If successful, the AP broadcasts a **Clear-to-Send (CTS)** frame.
3.  The CTS is heard by *all* nodes in the BSS (even the hidden terminals). It explicitly instructs all other stations to defer transmission for a specific duration.
4.  The sender transmits the large data frame uninhibited.

```text
RTS-CTS Exchange Timing:

   Node A (Sender)          Access Point (AP)          Node B (Hidden Terminal)
         |                         |                         |
         |-----(RTS to AP)-------->|                         |
         |                         |------(CTS to ALL)------>| (Node B sees CTS and  
         |<----(CTS to A)----------|                         |  defers its timer)
         |                         |                         |
         |=======[DATA]===========>|                         | 
         |                         |                         |
         |<======[ACK]=============|                         |
```

### 5.8 4. 802.11 frame: addressing
Wi-Fi frames are significantly more complex than standard Ethernet frames because they must account for the wireless leap between the mobile host and the AP routing into the wired network. As such, an 802.11 frame contains *four* MAC address fields.

```text
Simplified 802.11 Frame Structure:
+---------+----------+-------+-------+-------+---------+-------+---------+-----+
| Frame   | Duration | Addr1 | Addr2 | Addr3 | Seq Ctl | Addr4 | Payload | CRC |
| Control |          |       |       |       |         |       |         |     |
| 2 bytes | 2 bytes  | 6 B   | 6 B   | 6 B   | 2 bytes | 6 B   | 0-2312B | 4 B |
+---------+----------+-------+-------+-------+---------+-------+---------+-----+
```
*   **Address 1**: MAC address of the device directly *receiving* the wireless frame (e.g., the AP).
*   **Address 2**: MAC address of the device directly *transmitting* the wireless frame (e.g., the wireless host).
*   **Address 3**: MAC address of the router interface attached to the AP (crucial for bridging into the wired network).
*   **Address 4**: Only used in specific Ad Hoc / mesh routing scenarios.

When an AP bridges traffic, it strips the 802.11 headers and maps the MAC addresses into a standard 802.3 Ethernet frame, moving the target Router MAC to the destination field and the original Host MAC to the source field.

### 5.9 4. 802.11: mobility within same subnet
When a mobile host (H1) physically moves from the coverage area of BSS 1 to BSS 2, if both APs are connected to the same underlying layer-2 switch, the IP address remains constant. 

The wired switch handles this mobility via standard **self-learning**. When H1 associates with the new AP in BSS 2 and sends its first packet, the wired switch observes the new incoming port for H1's MAC address and updates its internal forwarding tables seamlessly. The broader internet routing remains entirely oblivious to this micro-level physical move.

***

## 6.0 Mobility

### 6.1 5. What is mobility?
From the network architecture's perspective, "mobility" exists on a spectrum of complexity:
*   **No Mobility**: A user shuts their laptop down, drives to a coffee shop, and reconnects. The IP address changes, but all connections were cleanly severed and restarted. This requires no special network protocol handling.
*   **Medium Mobility**: Moving between APs within the exact same provider subnet (as seen in section 5.9). Handled entirely by link-layer switches.
*   **High Mobility**: A user is actively downloading a file or on a VoIP call while driving down the highway, passing through different provider networks and underlying IP subnets, all while *maintaining ongoing logical connections*. 

This chapter focuses on managing the extreme complexity of high mobility.

### 6.2 5. Idea from contacting a mobile friend
Consider an analogy: If a friend travels constantly around the globe, how do you locate them to send a letter? You don't blindly search global phone books. Instead, you rely on a fixed "home base" (like calling their parents or checking a static Facebook profile). 

The internet requires a similar "home"—a definitive, static point of contact that always maintains up-to-date records on the current, physical location of the mobile entity.

### 6.3 5. Mobility: vocabulary
To establish this architecture, specific terminology is used:
*   **Home Network**: The permanent IP network assigned to the mobile device.
*   **Permanent Address**: The static IP address within the home network. Applications and correspondents *always* use this address to initiate contact with the mobile device.
*   **Home Agent**: A specialized router located on the home network. It performs mobility tracking and routing on behalf of the mobile device when the device is away.
*   **Visited Network**: The external network currently hosting the mobile device.
*   **Care-of-Address (COA)**: A temporary IP address assigned to the mobile device by the visited network.
*   **Foreign Agent**: A router in the visited network that assists with mobility functions, acting as a liaison.
*   **Correspondent**: Any remote host on the internet attempting to communicate with the mobile device.

### 6.4 5. Mobility: registration
When a mobile device connects to a new visited network, a registration handshake must occur so the internet knows where to route its traffic:
1.  The mobile device contacts the local **Foreign Agent** upon entering the visited network.
2.  The Foreign Agent reaches out over the internet to the device's **Home Agent** and effectively says, *"This specific mobile device is currently residing in my network at this Care-of-Address."*
3.  *Result*: The Home Agent now possesses a mapping of the mobile's permanent address to its current Care-of-Address.

### 6.5 5. Indirect Routing
In **Indirect Routing**, communication strictly passes through the Home Agent.

1.  The Correspondent addresses packets blindly to the mobile's **Permanent Address**.
2.  The Home Agent intercepts these packets. Using the registration data, the Home Agent encapsulates the packet and forwards it to the **Foreign Agent** at the Care-of-Address.
3.  The Foreign Agent unpacks the data and delivers it to the mobile device.
4.  The Mobile replies *directly* back to the Correspondent over the internet (bypassing the home agent for the return trip).

This approach is highly advantageous because it is completely *transparent* to the Correspondent. Ongoing connections (like TCP streams) do not break when the mobile moves to a new network; the Home Agent simply updates the COA and redirects the stream seamlessly.

However, it introduces the **Triangle Routing** problem. If a Correspondent in London is trying to talk to a Mobile device that is *also* currently visiting London, but the Mobile's Home Agent is in Australia, every single packet must travel from London -> Australia -> London. This is massively inefficient.

```text
Indirect Routing & Triangle Routing Inefficiency:

 [Correspondent] (London)
        |    ^
        |     \  (4) Direct Reply
     (1)|      \
        V       \
   [Home Agent] ->----(2 & 3)----> [Foreign Agent / Mobile]
   (Australia)                      (Visiting London)
```

### 6.6 5. Mobility via direct routing
To solve the triangle routing inefficiency, **Direct Routing** can be utilized:

1.  The Correspondent explicitly queries the Home Agent to retrieve the mobile's current Care-of-Address.
2.  The Correspondent then addresses packets *directly* to the Foreign Agent.
3.  The Foreign Agent delivers the packets to the mobile device.
4.  The Mobile replies directly.

While this creates optimal, straight-line routing, it introduces severe complexity. Direct Routing is *non-transparent* to the Correspondent. If the mobile user drives into a new visited network, the Care-of-Address changes instantly. The correspondent's ongoing connections would violently break unless a highly complex handoff protocol updates the correspondent mid-stream.

***

## 7.0 Mobile IP

### 7.1 5. Mobile IP
**Mobile IP** (defined in RFC 3344) is the standardized IETF architecture that implements these mobility concepts. It strictly utilizes Home Agents, Foreign Agents, registration, and **encapsulation**. 

Mobile IP comprises three major components:
1.  Indirect routing of datagrams.
2.  Agent discovery.
3.  Registration with the home agent.

### 7.2 Mobile IP: indirect routing
Mobile IP uses **encapsulation** (a packet-within-a-packet) to tunnel traffic from the Home Agent to the Foreign Agent. The Home Agent takes the original packet (destined for the Permanent Address) and wraps it entirely inside a new IP header destined for the Care-of-Address. Once it reaches the Foreign Agent, the outer header is stripped, and the original packet is delivered to the mobile.

```text
Encapsulation in Mobile IP:
+---------------------------------------------------+
| Outer IP Header (Dest: 79.129.13.2 / Care-of)     |  <-- Applied by Home Agent
+---------------------------------------------------+
| Inner IP Header (Dest: 128.119.40.186 / Permanent)|  <-- Applied by Correspondent
+---------------------------------------------------+
| TCP/UDP Payload Data                              |
+---------------------------------------------------+
```

### 7.3 Mobile IP: agent discovery
For a mobile node to find a Foreign Agent, Mobile IP utilizes **Agent Advertisement**. Foreign and Home Agents periodically broadcast standard ICMP messages (Type = 9) onto their local links. The mobile device listens for these ICMP extensions to learn the Care-of-Addresses available on the visited network and determine if registration is required.

### 7.4 Mobile IP: registration example
The standard registration flow involves a coordinated exchange of messages:
1.  The mobile node hears the ICMP Agent Advertisement containing the available Care-of-Address (COA).
2.  The mobile sends a Registration Request to the Foreign Agent.
3.  The Foreign Agent forwards this Request to the Home Agent, detailing the permanent address, the new COA, and an encapsulation format.
4.  The Home Agent processes the update and returns a Registration Reply to the Foreign Agent.
5.  The Foreign Agent passes the final confirmation back to the mobile node. Mobility is now established.

***

## 8.0 Chapter 7 Summary

### 8.1 Summary
Chapter 7 thoroughly maps the two primary pillars of disconnected computing:
*   **Wireless Link Operations**: Understanding the physical limitations of radio frequency (attenuation, interference), mathematical access partitions (CDMA), and the implementation of Wi-Fi (802.11) relying on CSMA/CA and RTS/CTS to avoid wireless collisions.
*   **Mobility Management**: Establishing the logical framework required to track physical device movement across the global internet. Using principles of indirect routing, tunneling, and Mobile IP architecture (Home Agents and Foreign Agents) to maintain continuous, transparent network connections regardless of physical location.
