# Chapter 1: Introduction to the Internet

## 1.0 Introduction and Roadmap
### 1.1 Course Goals and Overview (Slide 2)
The primary objective of this course is to develop a "feel" for networking terminology and concepts before diving into deep technical implementation details later. We utilize the Internet as the primary case study to understand networking principles.
*   **Core Topics:** We will define what the Internet is and what **protocols** are. We will examine the network structure by dividing it into the **network edge** (hosts, access networks) and the **network core** (packet switching, routing). Finally, we will analyze performance metrics (loss, delay, throughput) and the layered service models that govern communication.

### 1.2 Textbook Roadmap (Slide 3)
The chapter structure mirrors the logical progression of data:
1.  **1.1:** What is the Internet?
2.  **1.2:** The Network Edge (End systems and access technologies).
3.  **1.3:** The Network Core (Switching and structure).
4.  **1.4:** Performance (Delay, loss, throughput).
5.  **1.5:** Protocol Layers.
*(Note: Security and History are listed but marked as optional or strictly specific to later chapters).*

### 1.3 Key Questions (Slide 4)
This chapter seeks to answer four fundamental questions:
1.  **What is the Internet?** (Defining the hardware and the edge/core distinction).
2.  **How can the Internet grow so big?** (Understanding hierarchical design and scaling).
3.  **What is the Internet software architecture?** (Packet switching, best-effort models, and TCP/IP).
4.  **How good is the Internet's performance?** (Metrics).

***

## 2.0 Question 1: What is the Internet?
### 2.1 Components and Complexity (Slide 5)
While the Internet appears infinitely complex, it is constructed from three simple hardware components:
1.  **Hosts (End Systems):** Devices that run network applications.
2.  **Communication Links:** The physical media connecting devices.
3.  **Routers:** Devices that forward traffic.

Structurally, we divide the Internet into two main areas: the **Network Edge** (where users and applications reside) and the **Network Core** (the infrastructure that moves data).

### 2.2 Global Scale and Complexity (Slides 6-8)
The Internet is a "network of networks." It is not a single entity but a global aggregation of interconnected regional, national, and international networks.
*   **Scale:** It spans the globe, connecting billions of devices.
*   **Regional Networks:** High concentrations of connectivity exist within countries (e.g., the US map showing dense regional blobs).
*   **Operators:** The infrastructure is owned and operated by many distinct players, such as AT&T, Verizon, Sprint, and TimeWarner. There is no single owner of the Internet.

### 2.3 Hardware Building Blocks (Slide 9)
To build the Internet, we utilize specific roles for hardware:
*   **Hosts (End Systems):** These are the billions of connected computing devices (PCs, servers, laptops, smartphones, IoT devices). They are responsible for running **network apps**.
*   **Communication Links:** These are the physical pathways for data, utilizing fiber optics, copper wire, or radio spectrum. The transmission rate of a link is defined as its **bandwidth**.
*   **Routers and Switches:** These devices facilitate **packet switching**. They receive chunks of data (packets) and forward them toward their destination.

### 2.4 Network Structure: Edge vs. Core (Slide 10)
We conceptualize the network topology in two distinct parts:

1.  **Network Edge:** Located on the boundary of the Internet. This includes **hosts** (clients and servers). Note that modern servers are often housed in massive data centers. This area also includes the **access network**—the physical connection linking the host to the first router.
2.  **Network Core:** This is the Internet backbone, consisting of a mesh of interconnected routers responsible for long-distance data transport.

```
       [Mobile Network]          [Global ISP / Backbone]
             |                          (Core)
      (Base Station)                       |
             |                      [Router]---[Router]
        [Smartphone]                   /          \
           (Edge)                 [Router]      [Router]
                                     |             |
                               [Regional ISP]   [Inst. Net]
                                     |             |
                                  [Home Net]    [Server]
                                   (Edge)        (Edge)
```

***

## 3.0 The Network Edge: Access Networks
### 3.1 Overview (Slides 11-12)
The **access network** is the solution to the physical challenge of connecting an end system to the **edge router** (the first point of entry into the core). There are three main environments for access networks:
1.  **Residential:** Home access (Cable, DSL, FTTH).
2.  **Institutional:** Schools and companies (Ethernet).
3.  **Mobile:** Wireless access via cellular networks.

### 3.2 Digital Subscriber Line (DSL) (Slide 13)
**DSL** utilizes the existing telephone infrastructure to provide broadband.
*   **Mechanism:** A dedicated telephone line connects the home to the central office's **DSLAM** (DSL Access Multiplexer).
*   **Frequency Division:** Data and traditional voice calls are transmitted simultaneously on the same wire but at different frequencies.
*   **Dedicated Access:** Unlike cable, the line to the central office is *not* shared with neighbors.
*   **Performance:** It is **asymmetric**; downstream speeds (to the home, < 24 Mbps) are significantly faster than upstream speeds (from the home, < 2.5 Mbps).

```
[Home Phone]--\
               \
                [Splitter]-------(Phone Line)-------[Central Office / DSLAM]
               /                                            |
[PC]--[DSL Modem]                                       [Internet]
```

### 3.3 Cable Network (Slide 14)
**Cable** Internet utilizes the existing television infrastructure.
*   **HFC (Hybrid Fiber Coax):** The network uses fiber optics for the backbone and coaxial cable for the connection to the home.
*   **Shared Access:** Unlike DSL, the cabling is a shared broadcast medium. All homes in a neighborhood share the capacity of the distribution line to the cable headend (managed by the **CMTS** - Cable Modem Termination System). This means performance can degrade if many neighbors are downloading simultaneously.
*   **Performance:** Asymmetric (up to 30 Mbps down, 2 Mbps up).

### 3.4 Home and Enterprise Networks (Slides 15-16)
*   **Home Networks:** typically converge multiple functions into a single "box." This device acts as a Wireless Access Point, an Ethernet router, a Firewall/NAT, and a Modem (Cable/DSL).
*   **Enterprise Access (Ethernet):** Used in companies and universities. End systems connect via wired Ethernet to **Ethernet switches**, which then link to an institutional router. Transmission rates are high (100 Mbps, 1 Gbps, or 10 Gbps).

### 3.5 Wireless Access Networks (Slide 17)
Wireless networks connect end systems to a router via a base station (access point).
1.  **Wireless LANs (WLAN):** Operate within a building (approx. 100 ft range). The dominant standard is **802.11 (Wi-Fi)**.
2.  **Wide-Area Wireless Access:** Provided by telecommunications operators (Cellular). It spans tens of kilometers using technologies like 3G, 4G (LTE), and 5G.

***

## 4.0 Physical Media (Optional)
### 4.1 Concepts and Types (Slides 19-22)
*   **Bit:** The unit of information propagating between transmitter and receiver.
*   **Physical Link:** The matter lying between the transmitter and receiver.

**Guided Media (Solid):**
1.  **Twisted Pair (TP):** Two insulated copper wires. **Category 5** supports 100 Mbps/1 Gbps (Ethernet). **Category 6** supports up to 10 Gbps.
2.  **Coaxial Cable:** Two concentric copper conductors. It is bidirectional and broadband (supports multiple channels).
3.  **Fiber Optic Cable:** Glass fiber carrying light pulses. It features high-speed operation (10s-100s Gbps), low error rates, and immunity to electromagnetic noise.

**Unguided Media (Radio):**
Signals propagate freely through the electromagnetic spectrum. They are susceptible to environmental effects like reflection, obstruction, and interference. Types include terrestrial microwave, LAN (Wi-Fi), cellular, and satellite.

***

## 5.0 The Network Core
### 5.1 Router Functions (Slides 23-24)
The core's primary purpose is moving bits to their destination. The **Router** (or switch) is the specialized device that performs this task. It relays data hop-by-hop across links. A router performs two key functions:
1.  **Forwarding:** The local action of moving incoming data from a specific input link to the appropriate output link.
2.  **Routing:** The global action of determining the source-to-destination path taken by the packets, calculated using **routing algorithms**.

```
      [Input Link] ---> [ Routing Algorithm / Forwarding Table ] ---> [Output Link]
                                    |
                            (Decides: "If dest is X, use Link 2")
```

### 5.2 Summary of Q1 (Slide 25)
*   **Components:** Hosts, Links, Routers.
*   **Edge:** Runs applications; composed of servers, clients, and access networks.
*   **Core:** Interconnected routers responsible for routing and forwarding data bits.

***

## 6.0 Question 2: How does the Internet Scale?
### 6.1 The Scaling Challenge (Slides 27-28)
The Internet connects millions of access networks. The technical and business challenge is interconnecting all of them. Connecting every ISP to every other ISP directly (a full mesh) is impossible because the connections would scale at $O(N^2)$.

### 6.2 Evolution of Interconnection (Slides 29-35)
To solve the scaling problem, the Internet utilizes a **hierarchical structure**:
1.  **Global ISPs:** Access networks connect to global transit ISPs (Tier-1).
2.  **Competition:** Multiple global ISPs exist (Sprint, AT&T, Verizon).
3.  **IXPs and Peering:** Competitor ISPs must connect to ensure their customers can reach each other. They connect at **Internet Exchange Points (IXPs)** or via private **peering links**.
4.  **Regional Networks:** Smaller regional networks connect access networks to the Tier-1 ISPs.
5.  **Content Provider Networks:** Giants like Google and Microsoft build their own private global networks to bypass Tier-1 ISPs and bring content closer to the user.

```
      [Tier-1 ISP A]  <---(Peering/IXP)--->  [Tier-1 ISP B]
            |                                      |
     [Regional ISP]                         [Regional ISP]
            |                                      |
      [Access ISP]                           [Access ISP]
            |                                      |
         (User)                                 (User)
```

### 6.3 Summary of Q2 (Slide 36)
The "secret recipe" for global scaling is **Hierarchy**. By organizing networks into tiers (Access -> Regional -> Tier-1), we manage the complexity of billions of devices.

***

## 7.0 Question 3: Internet Software Architecture
### 7.1 Foundations (Slide 38)
The software architecture is built upon three pillars:
1.  **Enabling Concept:** Packet Switching.
2.  **Service Model:** Best effort.
3.  **Organization:** Layered protocols (TCP/IP).

### 7.2 Enabling Concept: Packet Switching (Slide 39)
Hosts break application messages into smaller chunks called **packets**. These packets are forwarded from router to router. Importantly, each packet is transmitted at the **full link speed**.

### 7.3 Store-and-Forward (Slide 40)
Internet routers operate on a **store-and-forward** basis. The router must receive the *entire* packet (all bits) before it can begin transmitting the first bit of that packet onto the next link. This introduces a transmission delay proportional to the packet size ($L$) and link rate ($R$).

```
[Source] --(sending)--> [Router] (Wait... Packet fully arrived?) --(Yes)--> [Dest]
```

### 7.4 Packet Switching vs. Circuit Switching (Slides 42-47)
The historical alternative to packet switching is **Circuit Switching** (used in traditional telephony).

*   **Circuit Switching:**
    *   End-to-end resources are **reserved** for the duration of a "call."
    *   No sharing; resources are dedicated.
    *   Implemented via **FDM** (Frequency Division Multiplexing) or **TDM** (Time Division Multiplexing).
    *   **Inefficiency:** If a user is silent, the reserved capacity is wasted.

*   **Packet Switching (The Internet Choice):**
    *   Data is sent on demand.
    *   Allows **Statistical Multiplexing**: Users share the bandwidth. Since users are not active 100% of the time, the network can support many more users than circuit switching.

**The Math of Efficiency (Slide 47):**
*   *Scenario:* A 1 Mbps link. Each user needs 100 kbps but is only active 10% of the time ($p=0.1$).
*   **Circuit Switching limit:** Max 10 users ($1 \text{ Mbps} / 100 \text{ kbps}$).
*   **Packet Switching capability:** With 35 users ($N=35$), we use binomial probability to calculate the chance of congestion (more than 10 users active simultaneously).
    $$ P(\text{active} > 10) = 1 - \sum_{x=0}^{10} \binom{35}{x} p^x (1-p)^{35-x} $$
*   The result is less than 0.0004 (0.04%).
*   **Conclusion:** Packet switching allows **3 times the number of users** (35 vs 10) with negligible probability of congestion.
