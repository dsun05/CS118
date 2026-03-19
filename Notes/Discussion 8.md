## 1.0 NAT

### 1.1 NAT (Network Address Translation)

*   **Problem:** The Internet faced a severe depletion of IPv4 addresses. Every device needs an IP address to communicate, but the 32-bit IPv4 space (approx. 4.3 billion addresses) was rapidly exhausting. **NAT (Network Address Translation)** was introduced as a short-term solution to mitigate this shortage.
*   **Alternative considered:** IP tunneling was considered, but NAT proved more universally adaptable without requiring pervasive changes to edge device operating systems.
*   **Mechanism:** NAT allows a local area network (LAN) to use a private IP address space (e.g., `10.x.x.x` or `192.168.x.x`). To the outside world (the Wide Area Network or WAN), the entire local network is represented by a single, distinct public IP address provided by the ISP.
*   **Side-benefit:** It inherently provides a layer of security. Because internal hosts have private IP addresses, they are not directly routable or addressable from the public Internet. Malicious actors cannot easily send unsolicited packets directly to a specific internal machine.
*   **Implementation:** The NAT router translates traffic by maintaining a mapping of `<public IP : port>` to `<private IP : port>`. 

***

### 1.2 NAT: detail

The core function of a NAT router is to intercept traffic and rewrite the IP headers to facilitate seamless communication between private and public networks.

*   **Outgoing packets:**
    *   When an internal host sends a packet to the Internet, the NAT router intercepts it.
    *   It replaces the `(source IP address, source port #)` in the IP and Transport layer headers with the router's own `(NAT IP address, new port #)`.
    *   Remote clients and servers on the Internet will receive this packet and respond directly to the router's `(NAT IP address, new port #)` as the destination address.
    *   Crucially, the router stores this specific translation pair mapping in its **NAT translation table** so it remembers which internal host made the request.
*   **Incoming packets:**
    *   When a response arrives from the Internet, the router inspects the destination port.
    *   It consults its **NAT translation table** to find the corresponding internal mapping.
    *   It replaces the `(destination NAT IP address, destination port #)` with the original `(source IP address, port #)` of the internal host and forwards the packet into the LAN.

***

### 1.3 NAT: downside

While NAT successfully extended the life of IPv4, it introduces significant architectural compromises to the network.

*   **Increased complexity:** Network middleboxes like NAT routers violate the strict layered model and the end-to-end principle of the Internet. Routers are supposed to only process up to Layer 3 (Network), but NAT requires routers to read and modify Layer 4 (Transport) port numbers.
*   **Single point of failure:** The NAT router must maintain stateful information (the translation table) for every active connection. If the router crashes, all active connections are dropped.
*   **Inability to run services directly inside a NAT box:**
    *   *Reason:* By default, incoming requests from the Internet cannot reach an internal server because the NAT router has no existing translation table entry for unsolicited incoming traffic. An external client has no way to address the internal server directly.
    *   *Solution:* This requires a static interface via **Static NAT (SNAT)** or "port forwarding." The administrator manually configures a permanent mapping in the translation table (e.g., all traffic hitting public port `80` is automatically forwarded to internal IP `10.0.0.2` port `80`).

***

### 1.4 Quick question

*   *Visual Aid: NAT router replacing source IPs and ports, updating a NAT translation table linking WAN side addresses to LAN side addresses*

```text
+-------------------------------------------------------------------------+
|                         NAT Translation Table                           |
|-------------------------------------------------------------------------|
|        WAN Side Address         |          LAN Side Address             |
|---------------------------------|---------------------------------------|
|     138.76.29.7, 5001           |          10.0.0.1, 3345               |
|            ...                  |                ...                    |
+-------------------------------------------------------------------------+
            ^                                      |
            | (3) Translates and updates table     | (1) Host sends packet
            |                                      v
      [ WAN / Internet ]                     [ LAN / Private Network ]
                                                     
<-- (2) Packet Source rewritten           (4) Packet sent to router -->
    S: 138.76.29.7, 5001                      S: 10.0.0.1, 3345
    D: 128.119.40.186, 80                     D: 128.119.40.186, 80

                  +-------------------------------+
                  |                               | ----> Host 10.0.0.1
  138.76.29.7     |         NAT ROUTER            | 
  ----------------|        (IP: 10.0.0.4)         | ----> Host 10.0.0.2
                  |                               |
                  +-------------------------------+ ----> Host 10.0.0.3
```

***

## 2.0 Routing Algorithms

### 2.1 Routing: concepts

The primary goal of a routing algorithm is to determine good paths (typically least-cost paths) from senders to receivers through the network of routers. Algorithms are categorized by how they gather and process information:

*   **Global vs. Decentralized information:**
    *   **Global:** All routers have complete knowledge of the network topology and all link costs. Algorithms that use this approach are called **"link state" algorithms**.
    *   **Decentralized:** Routers only know their physically-connected neighbors and the costs of the links to those neighbors. They calculate paths through an iterative process of computation and exchanging information with neighbors. Algorithms that use this approach are called **"distance vector" algorithms**.

***

### 2.2 Link state routing

*   **Dijkstra’s algorithm:** The primary algorithm used in link state routing.
    *   It requires that the entire network topology and all link costs are known to all nodes (usually accomplished by nodes broadcasting link-state packets to all other nodes in the network).
    *   The algorithm computes the least-cost paths from one specific node (the 'source' node) to all other nodes in the network.
    *   It is an iterative process: After $k$ iterations of the algorithm, the definitively known least-cost paths to exactly $k$ destination nodes have been found.

***

### 2.3 Link state routing: algorithm

*   *Visual Aid: Initialization and loop phases of Dijkstra's algorithm pseudocode*

```text
1  Initialization:
2    N' = {u}  // N' is the set of nodes whose least-cost path is definitively known
3    for all nodes v
4      if v is adjacent to u
5        then D(v) = c(u,v)
6      else D(v) = ∞
7
8  Loop
9    find w not in N' such that D(w) is a minimum
10   add w to N'
11   update D(v) for all v adjacent to w and not in N':
12     D(v) = min( D(v), D(w) + c(w,v) )  <-- [Link cost update heuristic]
13 until all nodes are in N'
```

*   **Link cost update heuristic:** `D(v) = min( D(v), D(w) + c(w,v) )`. This logic checks if the existing known path to `v` is cheaper, or if routing *through* the newly added node `w` provides a shorter path to `v`.
*   **Variables defined:**
    *   `c(x, y)`: The direct link cost from node $x$ to node $y$ (evaluates to ∞ if they are not direct, physically connected neighbors).
    *   `D(v)`: The current known cost value of the path from the source node to destination node $v$.
    *   `p(v)`: The predecessor node (the previous hop) along the current least-cost path to $v$.
    *   `N'`: The set of nodes for which the least-cost paths are definitively known.

***

### 2.4 Link state routing: example

*   **Scenario:** Using link state routing to set up a forwarding table for a specific node, $u$.

*   *Visual Aid: Graph diagram showing a 6-node network topology (u, v, w, x, y, z) with corresponding link costs*

```text
               (5)
         .-------------.
        /               \
       /    (2)    (3)   v
     [u] ------- [v] -- [w]
       \          | \    | \ (5)
     (1)\      (2)|  \(3)|  \
         \        |   \  |   [z]
          \       |    \ |   /
          [x] -- [y] -- [y]* \ (2)     *Note: the diagram simplifies
             (1)    (1)       \        routing matrix relationships 
```
*(Based on topology: u-v:2, u-x:1, u-w:5, v-x:2, v-w:3, x-w:3, x-y:1, w-y:1, w-z:5, y-z:2)*

***

### 2.5 Let’s work it out (Dijkstra's Table)

*   *Visual Aid: Table showing step-by-step iterations of Dijkstra's algorithm building the shortest paths from node u to all other nodes*

```text
| Step |   N'   | D(v), p(v) | D(w), p(w) | D(x), p(x) | D(y), p(y) | D(z), p(z) |
|------|--------|------------|------------|------------|------------|------------|
|  0   | u      |   2, u     |   5, u     |   1, u     |    ∞       |    ∞       |
|  1   | ux     |   2, u     |   4, x     |            |   2, x     |    ∞       |
|  2   | uxy    |   2, u     |   3, y     |            |            |   4, y     |
|  3   | uxyv   |            |   3, y     |            |            |   4, y     |
|  4   | uxyvw  |            |            |            |            |   4, y     |
|  5   | uxyvwz |            |            |            |            |            |
```
*(Explanation: In step 0, we check u's neighbors. `x` is the cheapest (cost 1). We add `x` to `N'`. In step 1, we update distances routing through `x`. The path to `w` drops from 5 to 4, and we discover `y` at cost 2. We repeat until all nodes are processed.)*

***

### 2.6 Link state routing: complexity

*   **Size:** Let $n$ be the number of nodes in the network.
*   **Work per iteration:** Each iteration requires checking all nodes $w$ that are not yet in the definitively known set $N'$.
*   **Comparisons:** The algorithm requires $n(n+1)/2$ comparisons across all iterations, resulting in a time complexity of $O(n^2)$.
*   **Optimization:** More efficient implementations using advanced data structures (like min-heaps or priority queues to find the minimum `D(w)`) can reduce the complexity to $O(n \log n)$.

***

### 2.7 Distance vector routing

*   **Bellman-Ford equation:** The mathematical foundation of Distance Vector routing, rooted in dynamic programming. Rather than needing global knowledge, nodes solve the shortest-path problem by breaking it into overlapping subproblems using their neighbors' knowledge.
*   **Variable Definition:** `dx(y)` := the cost of the least-cost path from node $x$ to node $y$.
*   **Equation:** `dx(y) = min_v {c(x,v) + dv(y)}`
    *   Where $v$ represents all direct neighbors of $x$.
    *   In plain English: "The cheapest path from $x$ to $y$ is the cheapest direct link to a neighbor $v$, plus that neighbor $v$'s cheapest path to $y$."

***

### 2.8 Distance vector routing: example & working it out

*   **Problem:** What is the cost of the least-cost path for $u \rightarrow z$?
*   *Visual Aid: Applying distance vector to the previous 6-node topology to find path u to z*
```text
      Neighbors of u are v, x, and w.
      We must check the cost through each neighbor to reach z.
```
*   **Applying Bellman-Ford:** `du(z) = min { c(u,v) + dv(z),  c(u,x) + dx(z),  c(u,w) + dw(z) }`
*   Assume the neighbors already computed their shortest distances to z: `dv(z) = 5`, `dx(z) = 3`, `dw(z) = 3`.
*   **Result evaluation:**
    *   Via v: $2 + 5 = 7$
    *   Via x: $1 + 3 = 4$
    *   Via w: $5 + 3 = 8$
    *   $\min\{7, 4, 8\} = 4$. Node $u$ routes through $x$ to reach $z$ at a cost of 4.

***

### 2.9 Distance vector routing: key idea

*   Nodes asynchronously and periodically send their own distance vector estimates (routing tables) to their direct neighbors.
*   Upon receiving a new distance vector estimate from a neighbor, a node updates its own distance vector by recalculating paths using the **Bellman-Ford equation**.
*   If the minimum cost to any destination changes, the node will then notify its neighbors of its new distance vector. This process ripples through the network until convergence.

***

### 2.10 Distance vector routing: caveat

*   **Count-to-infinity problem:** A significant vulnerability of standard Distance Vector routing where bad news (like a dropped link) travels very slowly, leading to routing loops.

*   *Visual Aid: Sequence demonstrating routing loops and the count-to-infinity problem when a link drops*

```text
[A] <---- 1 ----> [B] <---- 2 ----> [C]

Step 1: Normal operations. 
        B reaches C at cost 2. 
        A reaches C at cost 3 (via B).

Step 2: Link B-C goes down.
        B realizes direct link to C is lost. 
        However, A's last update said "I can reach C at cost 3!"

Step 3: The Trap.
        B believes A has an alternate route to C.
        B updates its route to C to go through A. New cost = c(B,A) + A's cost to C = 1 + 3 = 4.
        B tells A its new cost is 4.
        A updates its cost to C: c(A,B) + B's cost to C = 1 + 4 = 5.
        A tells B its cost is 5.
        ... They count to infinity (or until the protocol's defined maximum, e.g., 16 hops).
```

***

### 2.11 Distance vector routing: split horizon & poison reverse

To combat the count-to-infinity problem, two primary techniques are utilized:

*   **Split horizon:** A rule stating that if Node A reaches Destination C by routing *through* Node B, Node A should not advertise its route to C back to Node B.
    *   *Visual Aid: Testing the split horizon logic*
```text
        [Y]
       /   \
    4 /     \ 1
     /       \
   [X] ----- [Z]
         50
   (If Z routes to X via Y, Z tells Y nothing about X).
```
*   **Poison reverse:** An aggressive enhancement of split horizon. Instead of staying silent, Node A actively tells Node B that A's distance to C is *infinite*.
    *   By explicitly advertising the route as unreachable (infinite), B will never attempt to route through A to reach C, immediately breaking potential 2-node loops.
    *   In practice (e.g., in the RIP protocol): Infinite is defined as 16 hops.
*   *Visual Aid: Matrix updates illustrating poison reverse in action*
```text
If Z routes through Y to get to X:
Z tells Y its distance to X is INFINITE (∞).
This prevents Y from ever trying to route back through Z to reach X if the Y-X link fails.
(Table progression at t0, t1, t2... shows link costs updating instantaneously rather than incrementing +1 infinitely).
```

***

### 2.12 Link State v.s. Distance Vector

*   *Visual Aid: Comparison table contrasting Link state and Distance vector algorithms*

| Feature | Link State (LS) | Distance Vector (DV) |
| :--- | :--- | :--- |
| **Message complexity** | With $n$ nodes and $E$ links, requires $O(nE)$ messages sent to broadcast global topology. | Exchanges messages only between direct neighbors. Total messages vary by convergence time. |
| **Convergence speed** | $O(n^2)$ algorithm, requires $O(nE)$ messages to converge. Generally fast. | Convergence time varies. Can be slow and suffer from routing loops (Count-to-Infinity). |
| **Robustness** | High: A node can only advertise an incorrect cost for its *own connected links*. Errors are isolated. | Low: A node can advertise an incorrect path cost for *any destination*. Errors propagate through network. |
| **Implementation** | **OSPF** (Open Shortest Path First) | **RIP** (Routing Information Protocol) |

***

### 2.13 Routing Algorithms: summary

*   **Link-state routing (Dijkstra):** Calculates shortest paths based on a complete, globally known topology map. Nodes compute routes independently.
*   **Distance Vector (Bellman-Ford):** Calculates shortest paths iteratively based only on distance estimates provided by immediate neighbors.

***

## 3.0 Routing Protocols

### 3.1 BGP (Border Gateway Protocol)

*   BGP is the de-facto Inter-domain routing protocol of the Internet. It is the protocol that allows a subnet (or Autonomous System) to advertise its existence to the rest of the Internet ("I am here, and here is how to reach me").
*   **eBGP (External BGP):** Used to obtain subnet reachability information from neighboring Autonomous Systems (ASs).
*   **iBGP (Internal BGP):** Used to propagate that reachability information to all routers *internal* to the AS.
*   BGP determines the best route to external subnets by interacting with intra-domain routing protocols (like **OSPF**) to find the physical path to the correct boundary router.

***

### 3.2 BGP: routing policy

Unlike internal routing which focuses on performance (shortest path), inter-domain routing (BGP) is heavily dictated by business policy, revenue, and commercial agreements.

*   **Provider networks vs. Customer networks:**
*   *Visual Aid: Provider networks (A, B, C) and customer networks (W, X, Y)*

```text
    Provider Network A ------------- Provider Network B
     /             \                         |
    /               \                        |
  [W]               [C] Provider             [X]
Customer             |                     Customer
                    [Y]
                  Customer
```
*   **Routing restrictions:** A customer network (like X) attached to two provider networks (B and C) does *not* want to act as a transit network for B to reach C. Transit costs money. Therefore, X implements a routing policy where it will *not* advertise to B that it has a route to C. BGP enforces these revenue-based routing desires.

***

### 3.3 BGP: practice problems

*   **Loop detection:** BGP is a "Path Vector" protocol. BGP advertisements do not just contain link costs; they contain the *complete sequence* of Autonomous Systems (AS paths) the packet will traverse. If a router receives an advertisement and sees its own AS number already listed in the path, it instantly identifies a routing loop and rejects the route.
*   **Advertisement logic:** Stub networks (end networks like V or W) only advertise routes to themselves. A provider network (B) will not advertise a path to another provider (C) if neither the source nor the destination is a paying customer of B. Providers only route traffic to/from their own customers to maximize revenue and minimize free transit.

***

### 3.4 Routing: overall summary

*   **Intra-domain routing vs. Inter-domain routing:** 
    *   *Intra-domain (OSPF, RIP):* Focuses on technical Performance (speed, shortest path).
    *   *Inter-domain (BGP):* Focuses on business Policy (who pays whom).
*   **Scalability:** The Internet achieves scalability via hierarchical routing (splitting the world into Autonomous Systems) rather than one massive flat routing table.
*   **Algorithm distribution:** 
    *   Distance-vector is a fully-distributed algorithm (nodes work together interactively).
    *   Link-state is decentralized execution of a global algorithm (nodes gather global info, but compute locally).

***

## 4.0 SDN & Network Utilities

### 4.1 SDN: software defined networking

Traditional networking tightly integrates the control plane (routing logic) and the data plane (packet forwarding) inside the same hardware router. **SDN (Software Defined Networking)** separates them.

*   **Logically centralized control plane:** The routing logic is removed from individual switches and handled by a central software controller.
*   **Easier network management:** Administrators can change network behavior from a single dashboard rather than configuring hundreds of individual routers via CLI.
*   **Programmable forwarding table:** Switches rely on the central controller to program their forwarding tables. Communication is often handled via the **OpenFlow API**.
*   **Open implementation:** Control plane software is typically non-proprietary and open, breaking vendor lock-in.
*   **Components:** 
    1.  **Data plane switches:** "Dumb" hardware that quickly executes forwarding rules.
    2.  **SDN controller:** The "brain" managing the network state and communicating via OpenFlow.
    3.  **Network-control apps:** Higher-level software implementing routing, firewalls, and load balancing logic.

***

### 4.2 ICMP: Internet Control Message Protocol

*   **Function:** ICMP is used for network-level feedback, status checking, and error reporting (e.g., "Destination Network Unreachable").
*   It operates at the IP layer but ICMP messages are actually carried *within* IP packets, just like TCP or UDP segments.
*   **`ping`:** Uses ICMP Echo Request and Echo Reply messages to test if a host is alive and reachable.
*   **`traceroute`:** A clever tool utilizing the **TTL (Time to Live)** field in the IP header. The source sends packets with incrementally increasing TTLs (1, then 2, then 3). When the TTL hits 0 at a router, the router drops the packet and sends an ICMP "Time Exceeded" error back to the source, revealing the router's identity and measuring the hop delay.

***

### 4.3 Traceroute: example

*   *Visual Aid: Command line output showing a traceroute execution to Google's public DNS (8.8.8.8) demonstrating hop progression, IP addresses, and latency times*

```text
$ traceroute 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 64 hops max, 52 byte packets
 1  172.30.40.3 (172.30.40.3) 4.055 ms 3.017 ms 3.871 ms
 2  wifi-131-179-60-1.host.ucla.edu (131.179.60.1) 2.545 ms 2.288 ms 2.714 ms
 3  ra00f1.anderson--cr00f2.csb1.ucla.net (169.232.8.12) 3.653 ms 3.506 ms 3.724 ms
 4  cr00f2.csb1--bd11f1.anderson.ucla.net (169.232.4.5) 3.959 ms 4.383 ms 3.483 ms
 5  lax-agg6--ucla-10g.cenic.net (137.164.24.134) 3.951 ms 5.480 ms 3.840 ms
 6  74.125.49.165 (74.125.49.165) 6.558 ms 3.882 ms 3.890 ms
 7  108.170.247.129 (108.170.247.129) 3.192 ms
    108.170.247.193 (108.170.247.193) 93.964 ms
    108.170.247.161 (108.170.247.161) 3.297 ms
 8  108.177.3.127 (108.177.3.127) 3.657 ms
    209.85.255.73 (209.85.255.73) 3.571 ms
    108.177.3.129 (108.177.3.129) 3.261 ms
 9  google-public-dns-a.google.com (8.8.8.8) 5.315 ms 3.770 ms 12.165 ms
```

***

## Key Terms to Define in Notes

*   **NAT (Network Address Translation)**: A method mapping an entire private IP space into a single public IP address to conserve IPv4 addresses.
*   **NAT Translation Table**: The stateful database maintained by a NAT router mapping external `(Public IP, Port)` pairs to internal `(Private IP, Port)` pairs.
*   **Static NAT (SNAT)**: A permanent port-forwarding rule manually entered into a NAT router allowing external devices to initiate connections to an internal server.
*   **Link State Algorithms**: Routing algorithms where every node possesses complete knowledge of the network topology and link costs before calculating routes.
*   **Distance Vector Algorithms**: Decentralized routing algorithms where nodes calculate paths iteratively based only on the routing tables provided by their immediate physical neighbors.
*   **Dijkstra's Algorithm**: A famous link-state algorithm used to compute the absolute shortest path from a source node to all other nodes in a known graph. Time complexity $O(n^2)$.
*   **Bellman-Ford Equation**: The dynamic programming formula driving distance vector routing: `dx(y) = min_v {c(x,v) + dv(y)}`.
*   **Count-to-Infinity Problem**: A routing loop vulnerability in Distance Vector protocols caused by nodes incorrectly updating each other with stale path information after a link failure.
*   **Split Horizon**: A routing rule preventing a router from advertising a path back to the specific neighbor it uses to traverse that path.
*   **Poison Reverse**: An enhancement to split horizon where a router explicitly advertises a path as unreachable (infinite distance) to the neighbor it uses for that path, instantly breaking 2-node loops.
*   **OSPF**: Open Shortest Path First; the primary interior gateway protocol utilizing Link State routing.
*   **RIP**: Routing Information Protocol; a classic interior gateway protocol utilizing Distance Vector routing.
*   **BGP (Border Gateway Protocol)**: The central protocol of the Internet, used to route traffic between different Autonomous Systems based on business policies rather than purely technical speed.
*   **eBGP & iBGP**: External BGP (communication between different Autonomous Systems) and Internal BGP (communication distributing external routes to routers inside a single Autonomous System).
*   **AS (Autonomous System)**: A large network or group of networks under a single administrative routing policy (like an ISP or large university).
*   **SDN (Software Defined Networking)**: A modern networking architecture separating the control plane (logic) from the data plane (forwarding hardware) and centralizing it.
*   **OpenFlow API**: An open standard communication protocol used by an SDN controller to dictate forwarding tables to data plane switches.
*   **ICMP (Internet Control Message Protocol)**: The network-layer protocol used to send error messages and operational information indicating success or failure of communication with another IP address.
*   **Ping / Traceroute**: Network utilities utilizing ICMP. `Ping` tests general reachability via Echo requests, while `Traceroute` maps the exact routing hops to a destination.
*   **TTL (Time to Live)**: A field in an IP packet header decremented by one at each router hop; prevents packets from looping endlessly and is utilized strategically by Traceroute.
