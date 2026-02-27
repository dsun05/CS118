## 1.0 Making Internet Routing Scalable

### 1.1 Idealized vs. Practical Routing
In our foundational understanding of computer networks, we often rely on an idealized model of routing. This idealized routing assumes all routers are identical in capability and that the network is "flat," meaning every router has equal visibility and access to every other router. 

However, this is not true in practice due to two fundamental challenges:
* **Scale:** The Internet consists of billions of destinations. It is physically and computationally impossible for a single router to store all these destinations in its routing tables. Furthermore, the sheer volume of routing table exchanges required in a flat network would consume massive amounts of bandwidth, effectively swamping the network links and leaving no room for actual data transmission.
* **Administrative Autonomy:** The Internet is intrinsically a "network of networks." It is composed of thousands of distinct organizational networks (ISPs, universities, corporations). Each network administrator demands the ability to control routing within their own network, apply specific security policies, and manage their own hardware without external interference.

***

### 1.2 Internet Approach to Scalable Routing
To solve the problems of scale and autonomy, the Internet relies on a hierarchical approach. This is achieved by aggregating routers into distinct administrative regions known as **Autonomous Systems (AS)**, which are also frequently referred to as **domains**.

Routing is thus split into two distinct paradigms:
* **Intra-domain Routing (Intra-AS):** This handles routing *within* the same **Autonomous System (AS)**. 
    * Because the AS is under a single administrative control, all routers within that specific AS must run the same intra-domain routing protocol to ensure consistency.
    * However, different ASes are entirely free to run different intra-domain routing protocols based on their specific performance needs.
    * A **Gateway router** (or border router) sits at the "edge" of its own AS and maintains links to gateway routers in other external ASes.
* **Inter-domain Routing (Inter-AS):** This handles routing *among* different ASes across the global Internet.
    * **Gateway routers** are unique because they must perform both inter-domain routing (to talk to other networks) *and* intra-domain routing (to pass that external data inside to internal routers).

***

### 1.3 AS Numbers and Interconnection
To uniquely identify these administrative domains on the global Internet, each AS is assigned a unique **AS Number (ASN)**. 
* Originally, ASNs were 2-byte numbers, but due to the rapid growth of the Internet, they transitioned to 4-byte numbers (allowing for billions of unique IDs) starting around 2007. 
* Similar to private IP addresses, specific ASN ranges (e.g., 4200000000 ~ 4294967294) are reserved strictly for private usage and are not routed on the public Internet.

When ASes are interconnected, a router's forwarding table is actually configured by a combination of both protocols: intra-AS protocols determine the optimal path to internal destinations, while inter-AS protocols (combined with intra-AS) determine the optimal path to external destinations.

```text
* Visual Aid: Interconnected Autonomous Systems *

       +-------------------------+
       |           AS 1          |
       |  [Internal Router 1a]   |
       |           |             |
       |   [Gateway Router 1c]<--------+
       +-----------|-------------+     |
                   | eBGP link         |
                   v                   | eBGP link
       +-------------------------+     |
       |           AS 2          |     |
       |   [Gateway Router 2a]   |     |
       |           |             |     |
       |  [Internal Router 2d]   |     v
       |           |             |  +-------------------------+
       |   [Gateway Router 2c]<---->| [Gateway Router 3a]     |
       +-------------------------+  |          |              |
                                    | [Internal Router 3d]    |
                                    |          AS 3           |
                                    +-------------------------+
Note: Each router maintains a forwarding table configured by 
its local Intra-AS routing AND the global Inter-AS routing.
```

***

## 2.0 Intra-AS Routing: Routing within an AS

### 2.1 Common Intra-AS Routing Protocols
Administrators have historically relied on several protocols to handle routing within their own domains:
* **`RIP` (Routing Information Protocol):** A classic Distance Vector (DV) protocol where routing vectors are exchanged every 30 seconds. Due to its slow convergence and metric limitations, it is no longer widely used in modern, large-scale networks.
* **`EIGRP` (Enhanced Interior Gateway Routing Protocol):** Another DV-based protocol. It is highly optimized but was strictly Cisco-proprietary for decades (though it became an open standard in 2013).
* **`OSPF` (Open Shortest Path First):** A highly popular **Link-state routing** protocol. (Note: The `IS-IS` protocol is an ISO standard that functions almost identically to OSPF and is also widely used).

***

### 2.2 OSPF (Open Shortest Path First)
**OSPF (Open Shortest Path First)** is arguably the most dominant intra-domain protocol in enterprise networks. 
* The term "Open" signifies that its specification is publicly available (non-proprietary).
* It utilizes classic **Link-state routing**:
    * Each router actively floods OSPF link-state advertisements to all other routers in the entire AS. These messages are sent directly over IP, bypassing transport protocols like TCP or UDP.
    * Network administrators can configure multiple link cost metrics (such as bandwidth capacity or link delay) to fine-tune traffic flow.
    * Because of the link-state flooding, every router constructs a complete topological map of the AS and independently uses Dijkstra’s algorithm to compute its own forwarding table.
* **Security:** To prevent malicious actors from injecting false routing data and hijacking traffic, all OSPF messages can be cryptographically authenticated.

***

### 2.3 Hierarchical OSPF
For massively large AS domains, flooding link-state updates to thousands of routers would create unacceptable overhead. To mitigate this, OSPF can be structured into a two-level hierarchy consisting of a central **backbone** and multiple **local areas**.
* In this hierarchy, link-state advertisements are restricted: they are flooded *only* within a specific local area or *only* within the backbone. 
* Nodes maintain detailed topologies of their own local area, but only know the general "direction" to reach destinations in other areas.

This requires specific specialized router roles:
* **Local routers:** These reside entirely within a local area. They flood link-state info only within their area, compute internal routing, and forward any externally bound packets to the border routers.
* **Area Border Routers:** These sit on the edge of a local area. They "summarize" the distances to all destinations within their own area and advertise these summarized distances into the central backbone.
* **Backbone Router:** A router that runs OSPF routing limited entirely to the backbone area, ferrying traffic between different local areas.
* **Boundary router:** Also known as an AS boundary router, this connects the OSPF domain to other distinct Autonomous Systems.

```text
* Visual Aid: Hierarchical OSPF *

                        To other ASes
                              ^
                              |
                     +-----------------+
                     | Boundary Router |
                     +--------+--------+
                              |
   =========================================================
   |                 OSPF BACKBONE AREA                    |
   |                                                       |
   |      [Backbone Router]       [Backbone Router]        |
   |             /                         \               |
   |  +--------------------+      +--------------------+   |
   ===| Area Border Router |======| Area Border Router |====
      +---------+----------+      +---------+----------+
                |                           |
        +---------------+           +---------------+
        |    Area 1     |           |    Area 2     |
        |               |           |               |
        | [Local Router]|           | [Local Router]|
        | [Local Router]|           | [Local Router]|
        +---------------+           +---------------+
```

***

### 2.4 Hierarchical OSPF Example
When hierarchical OSPF computes paths, it does so in a synchronized sequence:
* **Step 1:** First, OSPF runs independently within each local area (e.g., Area 1). Local routers determine the shortest paths to all other nodes inside their own area.
* **Step 2:** Next, OSPF runs in the backbone. The **Area Border Routers** take the routing data from Step 1, summarize it (e.g., "I can reach all subnets in Area 1 with a cost of X"), and propagate these summaries via link states across the backbone to other border routers.
* **Step 3:** Finally, local area routers in other areas (e.g., Area 3) update their internal tables based on the summary information provided by their own area border routers, allowing them to route traffic cross-backbone to Area 1.

```text
* Visual Aid: Hierarchical OSPF Update Sequence *

[ Step 1: Intra-Area ] 
Area 1 computes internal routes.
(Router A -> Router B = Cost 2)

[ Step 2: Backbone Propagation ]
Area 1 Border Router tells the Backbone:
"Send traffic to me to reach Area 1."
Backbone routers calculate path to Area 1 Border Router.

[ Step 3: Remote Area Update ]
Area 3 Border Router receives Backbone data.
Tells Area 3 Local Routers: 
"To reach Area 1, route through me."
```

***

## 3.0 Inter-AS Routing: BGP (Border Gateway Protocol)

### 3.1 Intro to Internet inter-AS routing: BGP
While OSPF handles the inside of a network, **BGP (Border Gateway Protocol)** is the absolute de facto standard for inter-domain routing. It is universally regarded as the "glue holding the Internet together."
* BGP provides the mechanism that allows a subnet to advertise its existence to the rest of the global Internet, effectively saying, "I am here, and here is who I can reach."
* Crucially, BGP is strictly **Policy-Based Routing**. While intra-AS protocols care about speed and shortest paths, BGP determines "good" routes based on reachability information combined with the administrative and economic *policy preferences* of the ISP.

***

### 3.2 Why Inter-AS Differs from Intra-AS Routing
The distinct separation between Inter-AS and Intra-AS protocols exists because they serve fundamentally different purposes:
* **Policy:** Inter-AS routing is highly political and economic. Network admins want strict control over how traffic is routed and whose traffic they carry (e.g., not routing competitor traffic for free). Intra-AS routing is managed by a single admin, making policy restrictions largely irrelevant internally.
* **Scale:** BGP uses hierarchical routing and path vectors to drastically save on routing table size and reduce constant update traffic across the globe.
* **Performance:** Intra-AS routing can focus exclusively on performance (finding the fastest/lowest-delay path). In Inter-AS routing, administrative policy completely dominates over raw performance. A longer, slower path will be chosen if it aligns with the ISP's business contracts.

***

### 3.3 BGP Basics: A Path-Vector Protocol
BGP is classified as a **Path-Vector Protocol**.
* Instead of routing to individual IP addresses, it operates at the macroscopic AS level.
* Each advertised route contains the entire path (the **vector** or sequence of ASes) required to reach a given destination network prefix. 
* This provides a highly effective mechanism to easily detect routing loops. If an AS receives a route advertisement and sees its own ASN already listed within the path vector, it immediately knows a loop has occurred and rejects the route.

```text
* Visual Aid: BGP Path Vector Loop Detection *

   [ AS 3 ]  advertises: "Reach Subnet X via Path (AS3)"
      |
      v
   [ AS 2 ]  appends itself and advertises: "Reach Subnet X via Path (AS2, AS3)"
      |
      v
   [ AS 1 ]  appends itself and advertises: "Reach Subnet X via Path (AS1, AS2, AS3)"
      |
      v
   [ AS 3 ]  receives advertisement (AS1, AS2, AS3). 
             Looks at the vector. Sees "AS3" is already in the path!
             ACTION: Rejects route to prevent infinite loop.
```

```text
* Visual Aid: CLI output snippet for BGP Table *

Router> show ip bgp
BGP table version is 6128791, local router ID is 4.2.34.165
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop          Metric  LocPrf Weight Path
*  i3.0.0.0         4.0.6.142         1000    50     0      701 80 i
*> i12.3.21.0/23    192.205.32.153    1000    50     0      7018 4264 6468 i
*> e128.32.0.0/16   192.205.32.153    1000    50     0      7018 4264 6468 25 e
```

***

### 3.4 BGP Sessions
To exchange routing information, two BGP routers (referred to as "peers") must establish a **BGP Session**. This is done over a semi-permanent, reliable TCP connection on **Port 179**.
BGP utilizes four primary message types:
* **`OPEN`:** Opens the TCP connection to the peer and securely authenticates the sender.
* **`UPDATE`:** The core message. Used to advertise a new path to a destination or withdraw an old, broken path.
* **`KEEPALIVE`:** Sent periodically to keep the TCP connection alive in the absence of `UPDATE` messages. Also serves as an ACK for the `OPEN` request.
* **`NOTIFICATION`:** Reports errors in a previous message or is used to gracefully close the connection.

```text
* Visual Aid: BGP Session Establishment *

    [ BGP Router 1 ]                      [ BGP Router 2 ]
           |                                     |
           |====== Establish TCP Port 179 =======|
           |                                     |
           |------- 1. OPEN (Auth) ------------->|
           |<------ 2. KEEPALIVE (ACK) ----------|
           |                                     |
           |------- 3. UPDATE (All active routes)|
           |                                     |
           |<------ 4. UPDATE (All active routes)|
           |                                     |
           |==== Session is ALIVE & IDLE ========|
           |                                     |
           |------- 5. UPDATE (Incremental) ---->| (Only sent when topology changes)
           |------- 6. KEEPALIVE --------------->| (Sent periodically)
```

***

### 3.5 eBGP vs. iBGP
BGP actually operates in two different modes to ensure reachability spans both across and within networks:
* **eBGP (External BGP):** Operates between gateway routers in *different* ASes. Its job is to get subnet reachability information from neighboring domains. Think of it as the regional postal service moving mail between completely different cities.
* **iBGP (Internal BGP):** Operates between routers *within the same* AS. Once a gateway learns an external route via eBGP, it uses iBGP to propagate that reachability information to all other internal routers inside the AS. Think of it as the internal distribution center making sure local city post offices know where the out-of-state mail needs to be routed.

```text
* Visual Aid: eBGP vs iBGP Connectivity *

       AS 1                                       AS 2
 +----------------+                         +----------------+
 |                |                         |                |
 |  [Router 1a]   |                         |  [Router 2b]   |
 |       |        |                         |       |        |
 |    (iBGP)      |                         |     (iBGP)     |
 |       |        |                         |       |        |
 |  [Router 1c]<---------( eBGP Link )--------->[Router 2a]  |
 |                |                         |                |
 +----------------+                         +----------------+
```

***

## 4.0 BGP Operations

### 4.1 Entire Routing Process @ BGP Router
The lifecycle of a BGP route inside a router follows a strict pipeline consisting of five distinct steps. A BGP router does not just blindly accept and forward every route it hears; it applies policy at multiple stages.

```text
* Visual Aid: BGP Routing Process Pipeline *

                        +-------------------+
  1. Routes Received    |                   |   3. Decision Process
  from Neighbor ASes--->| 2. Import Policy  |   (Select Best Route)
                        |    Engine         |--------+
                        +-------------------+        |
                                                     v
                        +-------------------+  +-------------------+
  5. Routes Advertised  |                   |  |                   |
  to Neighbor ASes <----| 4. Export Policy  |<-| Routing Tables    |
                        |    Engine         |  | (BGP & IP Table)  |
                        +-------------------+  +-------------------+
```

***

### 4.2 BGP Path Advertisement
BGP gateway routers actively advertise path vectors to different destination network prefixes. As these advertisements propagate via eBGP (across domains) and iBGP (within domains), every router mathematically constructs a map of how to reach global destinations. 

```text
* Visual Aid: Propagation of a BGP Route *

Scenario: Subnet X is inside AS3.

1. AS3 gateway advertises (AS3, X) to AS2 via eBGP.
2. AS2 gateway receives it. 
3. AS2 gateway uses iBGP to distribute (AS3, X) to all internal AS2 routers.
4. AS2 edge router appends its ASN and advertises (AS2, AS3, X) to AS1 via eBGP.
5. AS1 now knows to reach X, it must forward packets to AS2.
```

```text
* Visual Aid: Accumulation of AS Path Vectors *

[Prefix 135.207.0.0/16 Originated at AT&T Research AS 6341]
                      |
                      v
          [AT&T AS 7018]  (Path = 7018 6341)
                      |
          +-----------+-----------+
          v                       v
[Sprint AS 1239]             [Global Crossing AS 3549]
(Path = 1239 7018 6341)      (Path = 3549 7018 6341)
          |                       |
          v                       v
[Ebone AS 1755]              [AS 3]
(Path = 1755 1239 7018 6341) (Path = 1239 3549 7018 6341)

*Notice how the AS-PATH grows leftward as it traverses ISPs.
```

***

### 4.3 Path Attributes
When a BGP route is advertised, it consists of the network prefix combined with several metadata fields known as attributes. Two of the most critical attributes are:
* **AS-PATH Attribute:** As discussed, this records the historical list of ASes through which the prefix advertisement has traversed. It is strictly used for loop prevention and path length calculations.
* **NEXT-HOP Attribute:** This indicates the specific IP address of the border router needed to reach the next-hop AS. It tells the router *exactly where* to send the packet on the local physical link to leave the current AS.

```text
* Visual Aid: AS-PATH Tracking *

Network       | Path to reach destination
-----------------------------------------
180.10.0.0/16 | 300 200 100
170.10.0.0/16 | 300 200
150.10.0.0/16 | 300 400

* By reading the Path column, AS 300 can verify it is not creating a routing loop.
```

***

### 4.4 The Next Hop Attribute & Reachability
A tricky scenario arises with BGP regarding the **NEXT-HOP Attribute**. When an eBGP router receives a route from a neighbor, it learns the neighbor's physical IP address as the NEXT-HOP. However, when this edge router passes the route inward to its internal peers using iBGP, *the NEXT-HOP attribute remains unchanged* by default.

**Problem:** Internal routers deep inside the AS may not have a routing table entry for the external IP address of the neighboring ISP's router, resulting in a "Next Hop Unreachable" issue, causing the traffic to be dropped.

```text
* Visual Aid: The Next-Hop Reachability Issue *

[ External AS 200 ]            [ Internal AS 300 ]
 Router C (IP: 192.10.1.1) <--> Router D <---- iBGP ----> Router F
 
 1. Router C advertises a route to Router D with NEXT-HOP = 192.10.1.1.
 2. Router D passes route to internal Router F via iBGP. NEXT-HOP remains 192.10.1.1.
 3. Router F wants to send data, but its internal OSPF table doesn't know where 192.10.1.1 is! 
```

**Fixes to "Next Hop Unreachable Issue":**
1. **`next-hop-self`:** The preferred method. The network admin configures the Edge Router (Router D) to rewrite the NEXT-HOP attribute to its *own* internal IP address before sending the iBGP update to Router F. Router F knows how to reach Router D, fixing the issue.
2. **Include external Interfaces:** Inject the external peering subnet (between C and D) into the internal OSPF routing protocol, making the external IP routable internally.

***

### 4.5 BGP Route Selection
Because the Internet is heavily interconnected, a router will often learn multiple different paths to the exact same destination. BGP uses a strict elimination process, ranked from highest to lowest priority, to select the single best route:

1. **Local preference value attribute:** A manually configured administrative weight. This is a pure policy decision (e.g., preferring a path that costs the ISP less money).
2. **Shortest AS-PATH:** If preferences are equal, it picks the path traversing the fewest number of Autonomous Systems.
3. **Closest NEXT-HOP router (**Hot Potato Routing**):** If paths are still equal, the router consults the internal OSPF protocol and picks the route that exits the current AS as quickly/cheaply as possible.
4. **Additional criteria:** Tie-breakers (e.g., Cisco's `WEIGHT`, prioritizing eBGP over iBGP, lowest router ID).

```text
* Visual Aid: BGP Best Path Selection (Cisco Example) *

1. Prefer highest WEIGHT (Cisco proprietary).
2. Prefer highest LOCAL_PREF (Policy decision).
3. Prefer locally originated route.
4. Prefer shortest AS_PATH.
5. Prefer lowest origin type.
6. Prefer lowest MED (Multi-Exit Discriminator).
7. Prefer eBGP paths over iBGP paths.
8. Prefer path with the lowest internal IGP (OSPF) metric to NEXT-HOP.
... [Tie breakers based on router IDs and aging] ...
```

***

### 4.6 Achieving Policy-Based Routing
Ultimately, BGP is about enforcing business rules using the import and export policy engines.
* **Import Policy:** Can aggressively accept or reject path vectors. An ISP can set a rule stating, "Never route traffic through competitor AS Y, even if it's the shortest path."
* **Export Policy:** Determines whether an AS will advertise its known paths to its neighbors.

A classic "real world" ISP policy revolves around the difference between **Provider Networks** and **Customer Networks**.
* An ISP (Provider) happily routes traffic to and from its paying Customer networks.
* However, if a customer is a **Dual-Homed Network** (connected to two different ISPs to ensure redundancy), that customer must absolutely *never* advertise routes learned from ISP A over to ISP B. If they did, ISP A and ISP B would start using the customer's personal network as a transit highway, overwhelming the customer's bandwidth.

```text
* Visual Aid: Policy Routing with a Dual-Homed Customer *

      [ Provider B ]-----------[ Provider C ]
            |                        |
            |                        |
            |                        |
      [ Customer x ] <---------------+ (Dual-homed to B and C)

Policy Enforcement: 
- Customer x learns paths to Provider C.
- Customer x intentionally DOES NOT advertise Provider C's paths to Provider B.
- Result: Provider B cannot use Customer x as a transit network to reach C.
```

***

## 5.0 Comprehensive Internet Routing Example

### 5.1 eBGP, iBGP, OSPF, and Hot Potato Routing
To see how all these protocols interlock to route a single packet across the global Internet:
1. **eBGP** is used by gateway routers to learn the overarching AS path vectors to external destinations.
2. **iBGP** is then used to propagate those external path vectors to all internal routers within the AS.
3. **OSPF** is constantly running in the background, computing the shortest physical path between the internal routers and the edge gateways.
4. Finally, **Hot Potato Routing** takes over. If an internal router has two valid gateways to exit the AS, it looks at the OSPF metrics and ruthlessly chooses the gateway with the lowest intra-domain cost. The goal is to offload the traffic (the "hot potato") to another network as quickly as possible to save internal bandwidth resources.

```text
* Visual Aid: Comprehensive Routing Pipeline *

Scenario: Router 2d wants to reach external subnet X.

                 (OSPF Cost: 201)
               +-----------------> [Gateway 2a] ----> Path to X
               |
[ Router 2d ]--+
               |
               +-----------------> [Gateway 2c] ----> Path to X
                 (OSPF Cost: 263)

1. eBGP: Gateways 2a and 2c both learn a path to X. Both AS-PATHs are equal length.
2. iBGP: Gateways 2a and 2c advertise this reachability internally to Router 2d.
3. OSPF: Router 2d calculates its internal distance: 201 to reach 2a, 263 to reach 2c.
4. Hot Potato Routing: Router 2d selects Gateway 2a. It throws the traffic out 
   the "closest" door to minimize internal network burden.
```

***

### **Key Terms**
* **Autonomous Systems (AS)** / **Domains**: Distinct administrative network regions that make up the Internet.
* **Gateway Router**: A router sitting at the edge of an AS that connects to other distinct ASes.
* **AS Number (ASN)**: A unique, 4-byte identifier assigned to every Autonomous System for use in BGP.
* **Intra-domain** / **Inter-domain Routing**: Routing within a single AS (OSPF) versus routing between different ASes (BGP).
* **OSPF (Open Shortest Path First)**: The dominant open-standard intra-AS routing protocol.
* **Link-state Routing**: Routing methodology where routers flood topology data and compute paths via Dijkstra's algorithm.
* **Area Border Router** / **Backbone Router**: Specialized OSPF roles used to scale OSPF hierarchically across large domains.
* **BGP (Border Gateway Protocol)**: The de facto inter-domain routing protocol of the Internet.
* **Path-Vector Protocol**: A routing protocol that transmits the entire sequence of domains a route transverses to prevent loops.
* **BGP Session**: A semi-permanent TCP connection over port 179 established between peering BGP routers.
* **eBGP vs iBGP**: External BGP for inter-AS communication versus Internal BGP for distributing routes within a single AS.
* **AS-PATH Attribute**: The BGP attribute listing the Autonomous Systems a route has traversed.
* **NEXT-HOP Attribute**: The BGP attribute pointing to the IP address of the next router needed to leave the AS.
* **`next-hop-self`**: A configuration command forcing a border router to replace the external NEXT-HOP IP with its own internal IP to ensure internal reachability.
* **Hot Potato Routing**: The practice of routing traffic to the closest available exit point of an AS to minimize internal transit costs.
* **Policy-Based Routing**: Making routing decisions based on administrative, economic, or security rules rather than purely shortest-path metrics.
* **Provider Network** vs. **Customer Network**: ISPs that provide transit (Provider) versus end-point organizations that consume bandwidth (Customer).
* **Dual-Homed Network**: A customer network connected to two or more different providers for redundancy.
