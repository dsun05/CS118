## 1.0 Web Advanced Features

### 1.1 Overview of Advanced Features
The basic Web (HTTP/1.0 and early 1.1) provided fundamental object retrieval. Modern web usage requires advanced features to handle state, performance, and efficiency.
*   **Extensions to the basic Web:**
    *   **Web cookies mechanism:** Solves the problem of HTTP statelessness by tracking user sessions.
    *   **Web caches (Proxy Servers):** Stores copies of data closer to the user to reduce latency and bandwidth usage.
    *   **Conditional GET:** Optimizes bandwidth by ensuring only updated content is downloaded.
*   **Recent developments:**
    *   **HTTP 2/3:** New iterations of the protocol designed to reduce latency in multi-object page loads.

### 1.2 Web Cookies
*   **Maintaining user/server state: Cookies**
    *   **The Problem:** The HTTP protocol is **stateless**. This means the server treats every request as a completely new transaction, with no memory of previous interactions.
    *   **Implication:** Without a mechanism like cookies, complex interactions like "shopping carts" or "logging in" would be impossible because the server wouldn't know that the request to "checkout" came from the same user who just "added an item."

    *   *Visual Aid: Diagram showing stateful protocol vs. stateless HTTP interaction*
    ```text
    +-----------------------+              +-----------------------+
    |   Stateless (HTTP)    |              |   Stateful Protocol   |
    +-----------------------+              +-----------------------+
    | Client      Server    |              | Client      Server    |
    |   |           |       |              |   |           |       |
    |   |--Req 1--->|       |              |   |--Login--->|       |
    |   |<--Resp 1--|       |              |   |<--OK------|       |
    |   |           |       |              |   |           |       |
    |   |--Req 2--->|       |              |   |--Data---->| (Server knows
    |   |<--Resp 2--|       |              |   |<--OK------|  client is
    |   |           |       |              |   |           |  logged in)
    | (Server has forgotten |              |   |--Logout-->|       |
    |  Req 1 by now)        |              |   |<--OK------|       |
    +-----------------------+              +-----------------------+
    ```

*   **Components of Cookie System:**
    To implement state over a stateless protocol, four components work together:
    1.  **Cookie header line in HTTP response message:** The server sends a `Set-cookie: <id>` header to the user.
    2.  **Cookie header line in next HTTP request message:** The browser automatically includes `Cookie: <id>` in future requests to that server.
    3.  **Cookie file kept on user’s host:** Managed by the browser, this file stores the ID associated with the specific domain.
    4.  **Back-end database at Web site:** The server stores the actual data (cart contents, user profile) linked to that ID.

*   **Example workflow (Susan visiting e-commerce site):**
    1.  Susan visits a site for the first time. The request is standard.
    2.  The server creates a unique ID (e.g., 1678) and creates an entry in its backend database.
    3.  The server responds with `Set-cookie: 1678`.
    4.  Susan's browser stores this ID.
    5.  One week later, Susan visits again. Her browser sees the stored ID and sends `Cookie: 1678` in the request.
    6.  The server uses ID 1678 to retrieve her specific data (e.g., recommendations) from the database.

    *   *Visual Aid: Sequence diagram of Client/Server interaction involving Set-cookie and Cookie headers*
    ```text
    Client (Susan)                          Server (Amazon)
       |                                         |
       |-------- HTTP Request (Msg) ------------>|
       |                                         | (Creates ID 1678)
       |                                         | (Creates Backend Entry)
       |<------- HTTP Response ------------------|
       |         Set-cookie: 1678                |
       |                                         |
    [Browser stores 1678 in cookie file]         |
       |                                         |
    [One week later]                             |
       |                                         |
       |-------- HTTP Request ------------------>| (Accesses Backend
       |         Cookie: 1678                    |  using ID 1678)
       |                                         |
       |<------- HTTP Response ------------------|
       |         (Customized Content)            |
       |                                         |
    ```

*   **Usage and Comments:**
    *   **Uses:** Authorization (staying logged in), shopping carts (remembering items), recommendations (based on history), and maintaining user session state.
    *   **Privacy Concerns:** Cookies allow sites to learn a significant amount about user behavior. **Third-party persistent cookies** (tracking cookies) allow advertising networks to track a user across multiple *different* websites, building a comprehensive profile of their browsing habits.
    *   **Challenge:** The core technical challenge cookies solve is how to maintain state at the endpoints (sender/receiver) over multiple transactions when the transport layer (HTTP) itself does not carry state.

### 1.3 Web Caches (Proxy Servers)
*   **Goal:** Satisfy a client's request without involving the **origin server**.
*   **Mechanism:**
    *   The user configures their browser to point to a **Web cache** (proxy).
    *   The browser sends *all* HTTP requests to the cache, not the destination.
    *   **Hit:** If the object is in the cache, the cache returns it immediately.
    *   **Miss:** If the object is not in the cache, the cache requests it from the origin server, stores a copy locally, and then returns it to the client.

    *   *Visual Aid: Diagram of Client communicating with Proxy Server, which communicates with Origin Server*
    ```text
            +--------+        HTTP Request        +--------+
            | Client | -------------------------> | Proxy  |
            |        | <------------------------- | Server |
            +--------+       HTTP Response        +--------+
                                                  ^    |
                                     HTTP Request |    | HTTP Response
                                     (If miss)    |    | (Contains object)
                                                  |    v
                                              +--------+
                                              | Origin |
                                              | Server |
                                              +--------+
    ```

*   **Why Web caching?**
    1.  **Reduce Response Time:** The cache is typically geographically or topologically closer to the client than the origin server.
    2.  **Reduce Traffic:** It significantly lowers traffic on the institution's access link to the Internet.
    3.  **Enable Content Delivery:** It allows "poor" content providers (small servers) to deliver content effectively by offloading requests to caches.

### 1.4 Conditional GET
*   **Goal:** Do not send an object if the cache has an up-to-date version. This solves the problem of "stale" cache data while avoiding unnecessary data transfer.
    *   No transmission delay for the object payload.
    *   Lower link utilization.
*   **Mechanism:**
    *   When the cache requests an object it already has (to check for updates), it adds the header: `If-modified-since: <date>`.
    *   **Scenario A (Not Modified):** If the object has *not* changed since that date, the server sends a response with no body (payload) and status code: `HTTP/1.0 304 Not Modified`.
    *   **Scenario B (Modified):** If the object *has* changed, the server sends the full object with status code: `HTTP/1.0 200 OK`.

    *   *Visual Aid: Flowchart comparing 304 Not Modified response vs. 200 OK response*
    ```text
    Client/Cache                                      Server
         |                                              |
         |---- GET Request ---------------------------->|
         |    If-modified-since: <date>                 |
         |                                              |
         |                                      [Check Last Modified]
         |                                              |
         |<--- Response --------------------------------|
    IF CHANGED:                                   IF UNCHANGED:
    HTTP/1.0 200 OK                               HTTP/1.0 304 Not Modified
    [Full Data Included]                          [No Data Included]
    ```

### 1.5 HTTP/2 and HTTP/3
*   **HTTP 1.1 Issues:**
    *   **FCFS (First-Come-First-Served):** Requests are processed in order.
    *   **Head-of-line (HOL) blocking:** If a client requests a large object followed by small objects, the small objects must wait for the large one to finish.
    *   **Loss Recovery:** A single lost TCP segment stalls the transmission of all objects on that connection.
*   **HTTP/2 Key Features:**
    *   **Goal:** Decrease delay in multi-object requests.
    *   **Object Priority:** Transmission order is based on client-specified priority, not just arrival time.
    *   **Push:** The server can push unrequested objects (e.g., style sheets) it knows the client will need.
    *   **Framing (Mitigating HOL blocking):** Objects are divided into small "frames". The server interleaves frames from different objects.
        *   *Result:* Small objects can slip in between the frames of a large object, preventing them from being stuck behind it.

    *   *Visual Aid: Comparison diagram of HTTP 1.1 serial loading vs. HTTP/2 interleaved framing*
    ```text
    HTTP 1.1 (FCFS / HOL Blocking):
    [Large Object 1 .........................] [Obj 2] [Obj 3]
    (Objects 2 and 3 must wait for 1 to finish completely)

    HTTP/2 (Interleaved Framing):
    [Frame 1-A] [Frame 2-A] [Frame 1-B] [Frame 3-A] [Frame 1-C] ...
    (Parts of Object 2 and 3 are delivered while Object 1 is still sending)
    ```

*   **HTTP/3:**
    *   **The TCP Problem:** Even with HTTP/2 interleaving, everything runs on one TCP connection. If one packet is lost, TCP pauses *everything* to recover that packet.
    *   **Solution:** HTTP/3 runs over **UDP** (using the QUIC protocol).
    *   **Benefit:** It provides security, and handles error/congestion control on a **per-object** basis. Losing a packet for "Image A" does not stall the stream for "Image B".

### 1.6 Summary on Web and HTTP
*   **Application:** Client-server architecture.
*   **Transfer:** Uses HTTP connections (persistent or non-persistent) over TCP.
*   **Messages:** Request and Response paradigm.
*   **Protocol Stack:**
    *   *Visual Aid: Diagram of the Internet protocol stack focusing on Application and Transport layers*
    ```text
    +-------------------------+
    | Application (HTTP, SMTP)|  <-- Processes communicate here
    +-------------------------+
    | Transport (TCP, UDP)    |  <-- Socket interface
    +-------------------------+
    | Network (IP)            |
    +-------------------------+
    | Link                    |
    +-------------------------+
    | Physical                |
    +-------------------------+
    ```

***

## 2.0 Electronic Mail (E-mail)

### 2.1 E-mail Overview
E-mail is a foundational asynchronous Internet application.
*   **Components:**
    1.  **User agent:** The interface for the human user.
    2.  **Mail servers:** The infrastructure that stores and routes mail.
    3.  **Simple Mail Transfer Protocol (SMTP):** The inter-server language for moving mail.

### 2.2 E-mail Architecture
*   **User Agent (aka "mail reader"):**
    *   Allows composing, editing, and reading messages.
    *   Examples: Microsoft Outlook, Apple Mail.
    *   Messages are stored on the server; the User Agent retrieves them.
*   **Mail Servers:**
    *   **Mailbox:** A storage area for incoming messages destined for a user.
    *   **Message queue:** A buffer for outgoing mail messages waiting to be sent to other servers.
    *   **SMTP protocol:** Runs between mail servers to transmit messages.

    *   *Visual Aid: Network diagram showing User Agents connected to Mail Servers, connected via SMTP*
    ```text
    [User Agent] --(1)--> [Mail Server] --(2: SMTP)--> [Mail Server] --(3)--> [User Agent]
       (Alice)             (Sender)                     (Receiver)             (Bob)
                               |                            ^
                               |                            |
                         [Message Queue]                [Mailbox]
    ```

### 2.3 SMTP Protocol Characteristics
*   **Transport:** Uses **TCP** to reliably transfer email on port 25.
*   **Delivery:** Direct transfer. The sending server acts as a client and connects directly to the receiving server.
*   **Three phases of transfer:**
    1.  **Handshaking (Greeting):** Establishment of the TCP connection and SMTP greeting.
    2.  **Transfer of messages:** The actual email content is sent.
    3.  **Closure:** The connection is terminated gracefully.

### 2.4 Scenario: Alice Sends E-mail to Bob
1.  Alice uses her User Agent (UA) to compose a message to `bob@someschool.edu`.
2.  Alice's UA sends the message to her mail server; the message is placed in the **message queue**.
3.  The client side of SMTP (on Alice's server) opens a TCP connection with Bob's mail server.
4.  The SMTP client sends Alice's message over the TCP connection.
5.  Bob's mail server places the message in Bob's **mailbox**.
6.  Bob invokes his UA to read the message.

    *   *Visual Aid: Step-by-step diagram of the email transmission path*
    ```text
    Alice (UA) -> Alice's Server (Queue) -> Internet (SMTP) -> Bob's Server (Mailbox) -> Bob (UA)
       [1]             [2]                    [3, 4]                  [5]              [6]
    ```

### 2.5 Sample SMTP Interaction
*   SMTP is text-based. It involves a Command/Response interaction between the client (C) and server (S).
*   **Commands:** ASCII text (e.g., `HELO`, `MAIL FROM`, `RCPT TO`, `DATA`, `QUIT`).
*   **Responses:** Status codes and phrases (e.g., `250 OK`, `354 Start mail input`).

    *   *Visual Aid: Code transcript of an SMTP conversation*
    ```text
    S: 220 hamburger.edu
    C: HELO crepes.fr
    S: 250 Hello crepes.fr, pleased to meet you
    C: MAIL FROM: <alice@crepes.fr>
    S: 250 alice@crepes.fr... Sender ok
    C: RCPT TO: <bob@hamburger.edu>
    S: 250 bob@hamburger.edu ... Recipient ok
    C: DATA
    S: 354 Enter mail, end with "." on a line by itself
    C: Do you like ketchup?
    C: .
    S: 250 Message accepted for delivery
    C: QUIT
    S: 221 hamburger.edu closing connection
    ```

### 2.6 Comparison: SMTP vs HTTP
*   **Interaction Model:**
    *   **HTTP:** **Pull** protocol (the user requests data).
    *   **SMTP:** **Push** protocol (the sender pushes data to the receiver).
*   **Similarities:** Both use ASCII command/response interactions and status codes.
*   **Differences:**
    *   **Encapsulation:** HTTP encapsulates each object in its own response message. SMTP sends multiple objects (text, attachments) in a single multipart message.
    *   **Encoding:** SMTP historically requires **7-bit ASCII**. Binary data (images) must be encoded to text.
    *   **Termination:** SMTP uses `CRLF.CRLF` (a period on a line by itself) to determine the end of a message.

### 2.7 Mail Message Format
*   Defined in **RFC 822**. This is the syntax for the content of the email, separate from the SMTP commands used to send it.
*   **Header lines:** `To:`, `From:`, `Subject:`.
*   **Body:** The actual "message" (ASCII characters).
*   **Separation:** A single blank line separates the header from the body.

    *   *Visual Aid: Diagram distinguishing Header, Blank line, and Body*
    ```text
    +----------------------------------+
    | To: bob@hamburger.edu            |  <-- Header
    | From: alice@crepes.fr            |
    | Subject: Ketchup                 |
    +----------------------------------+
    |                                  |  <-- Blank Line
    +----------------------------------+
    | Do you like ketchup?             |  <-- Body
    | How about pickles?               |
    +----------------------------------+
    ```

### 2.8 Mail Access Protocols
*   **SMTP:** Used *only* for delivery between servers and for sending from the sender's UA to their server. It is a "push" protocol.
*   **Mail Access Protocol:** Used by the receiver to "pull" mail from their server to their UA.
    *   **IMAP (Internet Mail Access Protocol):** Messages are stored on the server. Allows users to organize messages in folders on the server and keeps state across devices.
    *   **HTTP:** Web-based interface (e.g., Gmail, Yahoo). The browser speaks HTTP to the server; the server handles SMTP/IMAP in the background.

    *   *Visual Aid: Diagram showing SMTP for sending and Access Protocol for retrieval*
    ```text
    [Sender] --SMTP--> [Sender's Server] --SMTP--> [Receiver's Server] --Access Protocol--> [Receiver]
                                                                        (IMAP or HTTP)
    ```

### 2.9 Summary on E-mail
*   **App:** Client-server (User agent, mail servers).
*   **Protocol:** SMTP (Push logic).
*   **Format:** Header/Body separated by a blank line (RFC 822).
*   **Access:** IMAP (allows server-side storage and manipulation).

***

## 3.0 Domain Name System (DNS)

### 3.1 DNS Overview and Functions
*   **Basic function:** **IP address translation**. It finds the IP address given a host name (e.g., translating `www.google.com` to `173.194.204.99`).
*   It serves as the "phone book" of the Internet.

### 3.2 Motivation for DNS
*   **Host/Process identifiers:** Computers locate each other using an **IP address** and **port number**.
*   **Problem:** Humans are bad at remembering 32-bit numerical strings (e.g., 173.194.204.99).
*   **Solution:** We need a mapping service between the user-friendly **name** and the machine-friendly **IP address**.

### 3.3 DNS Services and Centralization Issues
*   **DNS Services:**
    1.  Hostname to IP translation.
    2.  **Host aliasing:** Mapping alias names (easy to type) to canonical names (complex internal names).
    3.  **Mail server aliasing:** Handling mail routing records (MX).
    4.  **Load distribution:** Rotating the IP address returned for a popular site (like Google) to balance load across replicated web servers.
*   **Why not centralize?** Why not have one giant server in the world with all the names?
    *   **Single point of failure:** If it crashes, the Internet stops.
    *   **Traffic volume:** It could not handle billions of simultaneous queries.
    *   **Distant database:** Latency would be terrible for users far from the central server.
    *   **Maintenance:** Updating a single massive database is impractical.
    *   **Conclusion:** Centralization **doesn't scale**.

### 3.4 DNS System Definition
*   **Distributed database:** The data is spread out over a hierarchy of many name servers.
*   **Application-layer protocol:** Hosts and name servers communicate to **resolve** names.
*   It is a core Internet function implemented at the network's "edge" via software, rather than inside the routers (core).

### 3.5 Key Concepts: Hierarchy
*   **Hierarchical names:** Names are structured as a tree. E.g., `kiwi.cs.ucla.edu`.
    *   Highest level: `edu`, `com`, `org` (TLDs).
    *   Next level: `ucla`, `mit` (Organizations).
    *   Lower level: `cs` (Department).
*   **Name servers:** The servers themselves are organized into this same hierarchy.
*   **Name resolution:** The process of looking up a name follows this hierarchy from top to bottom.

### 3.6 DNS Server Hierarchy
*   **Structure:**
    1.  **Root DNS Servers:** The starting point.
    2.  **Top Level Domain (TLD) servers:** Handle `.com`, `.org`, `.edu`, etc.
    3.  **Authoritative DNS servers:** Managed by organizations (e.g., `amazon.com`), these hold the actual IP records.
*   **Distributed structure:** No single server has all the data. Each knows only how to reach the next level down.

    *   *Visual Aid: Tree diagram illustrating the hierarchy from Root to Authoritative servers*
    ```text
                         [Root DNS Servers]
                                 |
           +---------------------+---------------------+
           |                     |                     |
    [.com DNS Servers]   [.org DNS Servers]    [.edu DNS Servers]
           |                     |                     |
    [amazon.com DNS]       [pbs.org DNS]       [nyu.edu DNS]
                                                       |
                                                [engineering.nyu.edu]
    ```

### 3.7 Root Name Servers
*   **Contact-of-last-resort:** If a local server doesn't know a name, it asks the root.
*   There are 13 logical root servers worldwide, but they are replicated hundreds of times (Anycast).
*   Managed by **ICANN** (Internet Corporation for Assigned Names and Numbers).
*   Includes **DNSSEC** capabilities for authentication and integrity.

### 3.8 TLD and Authoritative Servers
*   **TLD Servers:** Responsible for top-level domains.
    *   Generic TLDs: `.com`, `.net`, `.edu`.
    *   Country Code TLDs: `.uk`, `.fr`, `.cn`.
*   **Authoritative Servers:** An organization's own DNS server. It provides the final authoritative hostname-to-IP mappings for the organization's named hosts (e.g., `www.amazon.com`, `mail.amazon.com`).

### 3.9 Local DNS Name Servers
*   Strictly speaking, these do not belong to the hierarchy but are crucial for operation.
*   Provided by the ISP (Residential, University, Company).
*   **Role:** Acts as a **proxy**. When a host makes a DNS query, it is sent to the Local DNS server.
*   **Function:** It forwards the query into the hierarchy and maintains a **local cache** to speed up future requests for the same name.

### 3.10 DNS Resolution: Iterated Query
*   The burden of resolution is on the **Local DNS Server**.
*   The contacted server replies with the name of the server to contact next.
*   **Logic:** "I don't know the IP for `www.google.com`, but here is the IP for the `.com` TLD server. Ask them."

    *   *Visual Aid: 8-step diagram showing an Iterated Query flow*
    ```text
    [Client] --(1) Query--> [Local DNS] --(2) Query--> [Root DNS]
                                    <--(3) Referral to TLD DNS--
                                    |
                                    --(4) Query--> [TLD DNS]
                                    <--(5) Referral to Auth DNS--
                                    |
                                    --(6) Query--> [Auth DNS]
                                    <--(7) IP Address--
             <--(8) IP Address--
    ```

### 3.11 DNS Resolution: Recursive Query
*   The burden of resolution is placed on the **contacted name server**.
*   **Logic:** "I don't know the IP, so *I* will go find it for you and return the final answer."
*   **Drawback:** This places a heavy load on the upper levels of the hierarchy (Root/TLD servers).

    *   *Visual Aid: Diagram showing a Recursive Query flow*
    ```text
    [Client] -> [Local DNS] -> [Root DNS] -> [TLD DNS] -> [Auth DNS]
                                                                |
    [Client] <- [Local DNS] <- [Root DNS] <- [TLD DNS] <--------+ (Answer returns up chain)
    ```

### 3.12 DNS Resource Records (RR)
*   The DNS database stores **Resource Records (RR)**.
*   **Format:** `(name, value, type, ttl)`
*   **Types:**
    *   **Type=A:** `name` is a hostname, `value` is its IP address. (Standard lookup).
    *   **Type=NS:** `name` is a domain, `value` is the hostname of the authoritative name server for that domain. (Routing).
    *   **Type=CNAME:** `name` is an alias, `value` is the canonical (real) name. (Aliasing).
    *   **Type=MX:** `value` is the name of the mail server associated with `name`. (Email).

### 3.13 Caching and Updating Records
*   **Caching:** Once a name server learns a mapping, it caches the mapping.
*   **TTL (Time To Live):** A value in the record that determines when the cache entry expires (disappears).
*   **Consistency:** Cached entries may be **out-of-date**. If an IP changes, the world might not know until the TTL expires. DNS provides "best-effort" translation.

### 3.14 DNS Protocol Messages
*   Query and Reply messages use the exact same format.
*   **Header:**
    *   **Identification:** A 16-bit number used to match a reply to a specific query.
    *   **Flags:** Indicate if the message is a query or reply, if recursion is desired, and if the reply is authoritative.
*   **Data Sections:** Questions, Answers (RRs), Authority (RRs), Additional Info.

    *   *Visual Aid: Diagram of the DNS message structure and byte layout*
    ```text
    +---------------------------+---------------------------+
    | Identification (2 bytes)  | Flags (2 bytes)           |
    +---------------------------+---------------------------+
    | # of Questions            | # of Answer RRs           |
    +---------------------------+---------------------------+
    | # of Authority RRs        | # of Additional RRs       |
    +---------------------------+---------------------------+
    | Questions (Variable length: name, type)               |
    +-------------------------------------------------------+
    | Answers (Variable length: RRs)                        |
    +-------------------------------------------------------+
    | Authority (Variable length: RRs)                      |
    +-------------------------------------------------------+
    | Additional Info (Variable length: RRs)                |
    +-------------------------------------------------------+
    ```

### 3.15 Inserting Records into DNS
*   When creating a new startup (e.g., "Network Utopia"):
    1.  Register the domain name at a **DNS registrar** (e.g., Network Solutions).
    2.  Provide the registrar with the names and IPs of your primary and secondary authoritative name servers.
    3.  The registrar inserts two records into the TLD server (e.g., the `.com` server):
        *   **NS record:** Maps `networkutopia.com` to `dns1.networkutopia.com`.
        *   **A record:** Maps `dns1.networkutopia.com` to the IP address of the DNS server.
    4.  Configure the local authoritative server with the actual **A** records (for web servers) and **MX** records (for mail servers).

### 3.16 Summary on DNS
*   **Database:** It is a distributed, hierarchical database.
*   **Scalability:** Scalability is achieved by adding hierarchies (Root -> TLD -> Local).
*   **Operation:** Resolves names using Iterated (standard) or Recursive queries.
*   **Protocol:** Runs on UDP (primarily) and TCP port 53.
