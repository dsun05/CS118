# Lecture Outline: Chapter 2 - Application Layer

## 1.0 Introduction and Roadmap

### 1.1 Chapter Context and Roadmap (Slides 1-2)
*   **Source Material:** These notes are based on *Computer Networking: A Top-Down Approach*, 8th edition, by Jim Kurose and Keith Ross.
*   **Chapter Roadmap:** This chapter focuses on the principles of network applications, moving from theory to specific protocols.
    *   **Principles of Internet applications:** Conceptual foundations including architectures and transport requirements.
    *   **Web and HTTP:** The most ubiquitous application and its stateless protocol.
    *   **E-mail, SMTP, IMAP:** Asynchronous communication protocols.
    *   **Domain Name System (DNS):** The internet's directory service.
    *   **P2P applications:** Decentralized file sharing.
    *   **Video streaming:** CDNs and content distribution.
    *   *Note: Socket programming is covered separately in TA materials.*

### 1.2 Recall: Layered Internet Protocol Stack (Slides 3-5)
The Internet works on a layered architecture where each layer provides a specific service to the layer above it.

*   **Application Layer:** Supports network applications. This is where protocols like **FTP**, **SMTP**, and **HTTP** reside. It deals with user-facing data.
*   **Transport Layer:** Handles process-process data transfer. Key protocols are **TCP** (reliable) and **UDP** (unreliable).
*   **Network Layer:** Responsible for routing datagrams from the source host to the destination host. Includes **IP** and routing protocols.
*   **Link Layer:** Handles data transfer between neighboring network elements (nodes) on the same physical link. Examples include **Ethernet**, **WiFi** (802.11), and **PPP**.
*   **Physical Layer:** Handles the actual transmission of bits "on the wire" or over the air.

```markdown
* Visual Aid: The 5-Layer Internet Protocol Stack *

+---------------------+
|     Application     |  <-- Supporting Network Apps (HTTP, SMTP)
+---------------------+
|      Transport      |  <-- Host-to-Host Data Transfer (TCP, UDP)
+---------------------+
|       Network       |  <-- Routing of Datagrams (IP)
+---------------------+
|        Link         |  <-- Neighboring Data Transfer (Ethernet, WiFi)
+---------------------+
|      Physical       |  <-- Bits on the wire
+---------------------+
```

*   **Stack Dependency (The Hourglass Model):** The architecture relies on lower layers providing services to upper layers.
    *   Applications (running @hosts) are built on **Reliable (or unreliable) transport**.
    *   Transport is built on **Best-effort global packet delivery** (IP).
    *   Global delivery is built on **Best-effort local packet delivery**.
    *   Local delivery is built on the **Physical transfer of bits**.

*   **Application Location:**
    *   Network applications run **only on hosts** (end systems), such as laptops, servers, and smartphones.
    *   They do **not** run on network core devices like routers or switches. This allows for rapid development because the network core does not need to be updated to support a new application.

***

## 2.0 Two Key Questions in Chapter 2 (Slide 6)

### 2.1 Core Questions
To understand the Application Layer, we must answer two fundamental questions:

1.  **How to build an Internet application?**
    *   We will explore the two primary paradigms used to program applications: Client-Server and Peer-to-Peer.
2.  **What are the popular examples of Internet applications?**
    *   We will analyze the architecture and protocols of 5 specific examples:
        1.  **Web services:** Considered the "Killer App" that brought the Internet to the mainstream.
        2.  **Internet emails:** The backbone of business communication.
        3.  **Internet DNS service:** The essential naming service translating hostnames to IP addresses.
        4.  **P2P file sharing:** Decentralized content distribution.
        5.  **Video streaming:** High-bandwidth media delivery.

***

## 3.0 Q1: How to Build Internet Applications?

### 3.1 Development Paradigms (Slides 7-12)
When developing a network app, you write code that runs on different end systems and communicates over the network. You generally choose between two architectural paradigms.

**Option 1: Client-Server Paradigm**
In this traditional model, there is a distinct hierarchy between the communicating parties.
*   **Server (The Service Provider):**
    *   It is an **always-on host**.
    *   It possesses a **permanent IP address** so clients can always find it.
    *   It is often housed in massive data centers to handle scaling (serving millions of requests).
*   **Clients (The Service Requesters):**
    *   They contact the server to request resources.
    *   They may be intermittently connected (e.g., a laptop turned off at night).
    *   They often have **dynamic IP addresses** (changing every time they connect to a new network).
    *   **Crucial distinction:** Clients do **not** communicate directly with each other.
    *   *Examples:* Web (HTTP), Email (IMAP), File Transfer (FTP).

```markdown
* Visual Aid: Client-Server Architecture *

      [ Server (Data Center) ]
      (Always-on, Fixed IP)
             ^   ^
            /     \
           /       \  <-- Clients contact Server
          /         \
     [Client A]   [Client B]
     (No direct communication between A and B)
```

**Option 2: Peer-to-Peer (P2P) Architecture**
In this model, the distinction between client and server blurs.
*   **Each peer is both a server & a client:** A node requests files (client) but also uploads chunks of files to others (server).
*   **Self-Scalability:** This is the major advantage. As new peers join, they create new demand, but they *also* add new service capacity (bandwidth/storage) to the system.
*   **Characteristics:**
    *   Peers communicate directly with one another.
    *   Peers are intermittently connected and change IP addresses frequently.
    *   There is **no always-on server** required to mediate data transfer.
*   **Management:** This architecture is complex to manage due to the dynamic nature of the nodes.
*   *Example:* BitTorrent.

### 3.2 Procedures to Construct an Internet Application (Slides 13-18)
Building a network app requires understanding three basic concepts:

1.  **Internet Application Processes**
    *   A **Process** is a program running within a host.
    *   **IPC (Inter-process communication):** Used when two processes on the *same* host communicate (governed by the OS).
    *   **Messages:** Used when processes on *different* hosts communicate.
    *   **Client Process:** The process that *initiates* communication.
    *   **Server Process:** The process that *waits* to be contacted.

2.  **Addressing Processes (The Unique ID)**
    *   To receive a message, a process must have a unique identifier.
    *   An **IP Address** (32-bit) identifies the *host* device, but this is insufficient because a host runs many processes.
    *   **Identifier = IP Address + Port Number**.
    *   The **Port Number** identifies the specific receiving process on the host.
    *   *Standard Ports:* Web Server (HTTP) uses port **80**; Mail Server (SMTP) uses port **25**.

3.  **Internet Sockets**
    *   A **Socket** is the software interface between the application process and the transport layer.
    *   It is analogous to a **door**. The sending process shoves a message out the door (socket) and relies on the transport infrastructure on the other side to deliver it to the receiving process's door.
    *   The application developer controls everything *above* the socket; the OS controls everything *below* it.

```markdown
* Visual Aid: The Socket Interface *

   [ Application Process ]  (Controlled by App Developer)
            |
            V
       [ Socket ]           <-- The "Door" (Interface)
            |
            V
   [   Transport Layer   ]  (Controlled by OS - TCP/UDP)
            |
   [      Internet       ]
```

### 3.3 Step 2: Select Transport Service (Slides 19-22)
When creating a socket, the developer must choose a transport layer protocol based on the application's needs.

**Option 1: TCP Service (Transmission Control Protocol)**
*   **Connection-oriented:** Requires a "handshake" to setup a connection between client and server before data flows.
*   **Reliable Transport:** Guarantees that data sent is received without error and in order.
*   **Automated Rate Control:**
    *   **Flow Control:** Prevents the sender from overwhelming the receiver.
    *   **Congestion Control:** Throttles the sender when the network core is overloaded.
*   **Limitations:** Does NOT provide timing guarantees (latency) or minimum bandwidth guarantees.

**Option 2: UDP Service (User Datagram Protocol)**
*   **Connectionless:** No setup required; data is sent immediately.
*   **Unreliable Data Transfer:** Data may arrive out of order, corrupted, or not at all.
*   **No Rate Control:** The application can send data as fast as it wants (useful for streaming).
*   **Limitations:** No reliability, flow control, or congestion control.

**Criteria for Choosing a Service:**
1.  **Data Integrity:** Does the app need 100% distinct data transfer (e.g., File Transfer, Web, Banking)? Use TCP. Can it tolerate loss (e.g., Live Audio/Video)? UDP may be acceptable.
2.  **Throughput:** Does the app need a guaranteed throughput (Multimedia)? Or is it an **elastic app** that adapts to whatever bandwidth is available (Email, File transfer)?
3.  **Timing:** Does the app require low delay to be effective (Internet Telephony, Gaming)?
4.  **Security:** Transport layers can be enhanced with TLS/SSL for encryption.

**Table: Application Requirements on Transport Services**
The following table summarizes what different applications require from the transport layer.

| Application | Data Loss | Throughput | Time Sensitive? |
| :--- | :--- | :--- | :--- |
| File transfer/download | No loss | Elastic | No |
| E-mail | No loss | Elastic | No |
| Web documents | No loss | Elastic | No |
| Real-time audio/video | Loss-tolerant | Audio: 5Kbps-1Mbps<br>Video: 10Kbps-5Mbps | Yes, 10s msec |
| Streaming audio/video | Loss-tolerant | Same as above | Yes, few secs |
| Interactive games | Loss-tolerant | Kbps+ | Yes, 10s msec |
| Text messaging | No loss | Elastic | Yes and no |

### 3.4 Step 3: Define Application-Layer Protocol (Slides 23-24)
The application-layer protocol defines how the processes (running on different end systems) pass messages to each other. It must define:
1.  **Types of messages:** (e.g., Request, Response).
2.  **Message Syntax:** What fields exist in the message and how they are delineated.
3.  **Message Semantics:** The meaning of the information in the fields.
4.  **Rules:** When and how a process sends messages and responds to them.

**Types of Protocols:**
*   **Open Protocols:** Defined in **RFCs** (Request for Comments). They allow interoperability (e.g., Chrome talking to an Apache server). Examples: HTTP, SMTP.
*   **Proprietary Protocols:** Owned by a private company. Not published. Examples: Skype, Zoom.

**Table: Examples of Application-Layer Protocols**
The following table maps common applications to their specific Application-layer protocols (RFCs) and the Transport protocol they utilize.

| Application | Application Layer Protocol | Transport Protocol |
| :--- | :--- | :--- |
| File transfer/download | **FTP** [RFC 959] | TCP |
| E-mail | **SMTP** [RFC 5321] | TCP |
| Web documents | **HTTP 1.1** [RFC 7320] | TCP |
| Internet telephony | **SIP** [RFC 3261], **RTP** [RFC 3550], or proprietary | TCP or UDP |
| Streaming audio/video | **HTTP** [RFC 7320], **DASH** | TCP |
| Interactive games | WOW, FPS (proprietary) | UDP or TCP |

### 3.5 Summary of Q1 (Slide 25)
To build an app, you need:
1.  **Processes** (Client/Server or Peer).
2.  **Addressing** (IP + Port).
3.  **Sockets** (The interface).
4.  **Transport Choice** (TCP for reliability, UDP for speed).
5.  **Protocol Definition** (Syntax and Rules).

***

## 4.0 Q2: Real-World Examples (Slide 27)
We will now apply the concepts above to analyze five real-world applications:
1.  **Web**
2.  **Email**
3.  **DNS services**
4.  **P2P file sharing**
5.  **Video streaming**

For each, we look at the architecture (Client-Server vs. P2P) and the protocol design.

***

## 5.0 First Example: Web & HTTP

### 5.1 Introduction to Web & HTTP (Slides 28-31)
*   **Context:** The Web is an application; **HTTP** is the protocol that powers it.
*   **Architecture:** The Web uses the **Client-Server** model.
    *   **Client (Browser):** Examples include Firefox, Chrome, Safari. It initiates requests.
    *   **Server:** Examples include Apache, Nginx, IIS. It waits for requests and serves objects.
*   **Communication:** They communicate via sockets using **HTTP messages**.

### 5.2 #1: Content - Web Pages (Slides 32-33)
*   A Web page is not a single file; it is a collection of **objects**.
*   **Object:** A file such as an HTML file, a JPEG image, a Java applet, or an audio clip.
*   **Structure:** A web page consists of a **base HTML-file** which contains references (links) to other objects.
*   **Addressing:** Each object is addressable by a unique **URL** (Uniform Resource Locator).
    *   *URL Format:* `host name` + `path name` (e.g., `www.someschool.edu/someDept/pic.gif`).

### 5.3 #2: Transfer - HTTP Protocol Overview (Slides 34-38)
*   **Definition:** **HTTP** (Hypertext Transfer Protocol) is the Web's application-layer protocol.
*   **Transport Protocol:** HTTP uses **TCP**.
    1.  Client initiates a TCP connection (creates socket) to server at **port 80**.
    2.  Server accepts the connection.
    3.  HTTP messages (application layer) are exchanged between the browser and web server.
    4.  The TCP connection is closed.
*   **Statelessness:**
    *   **HTTP is a stateless protocol.**
    *   The server maintains **no information** about past client requests. If you ask for the same object twice in 10 seconds, the server sends it again, having forgotten you just asked for it.
    *   *Why?* Protocols that maintain state are complex. If a server crashes, the state is lost, leading to inconsistencies between client and server views.

### 5.4 #2: HTTP Connections (Slides 39-51)
HTTP can operate in two modes regarding how it utilizes the underlying TCP connection.

**1. Non-persistent HTTP (HTTP 1.0)**
*   **Mechanism:**
    1.  TCP connection opened.
    2.  **At most one object** is sent over the TCP connection.
    3.  TCP connection is closed immediately.
*   **Performance Analysis:**
    *   **RTT (Round-Trip Time):** The time for a small packet to travel from client to server and back.
    *   **Step A (Base HTML):** Requires 1 RTT to set up TCP + 1 RTT to request/receive the HTML file.
    *   **Step B (Referenced Objects):** For *each* object (e.g., 10 images), the browser must open a *new* TCP connection (1 RTT) and request the file (1 RTT).
    *   **Total Response Time per Object:** $2 \times RTT + \text{file transmission time}$.
*   **Issues:** Requires 2 RTTs per object. The Operating System (OS) must allocate resources for every new connection, creating high overhead.
*   **Parallel Connections:** Modern browsers use parallel TCP connections (opening 5-10 at once) to download objects simultaneously, mitigating the slowness but increasing server load.

**2. Persistent HTTP (HTTP 1.1)**
*   **Mechanism:**
    1.  TCP connection opened.
    2.  The server leaves the connection **open** after sending a response.
    3.  **Multiple objects** are sent over the **single** TCP connection.
    4.  The connection closes only after a timeout or explicit close instruction.
*   **Performance Analysis:**
    *   The client sends requests as soon as it encounters a referenced object (pipelining).
    *   **Efficiency:** As little as **One RTT** for all referenced objects (after the initial connection). This essentially cuts the response latency in half compared to non-persistent.

```markdown
* Visual Aid: Non-persistent vs. Persistent Timing *

[Non-Persistent]             [Persistent]
   Client    Server             Client    Server
     |         |                  |         |
     |--SYN--->| (TCP Setup)      |--SYN--->| (TCP Setup)
     |<--ACK---|                  |<--ACK---|
     |--GET--->| (Request Obj)    |--GET--->| (Request Obj)
     |<--File--|                  |<--File--|
     (Conn Closes)                (Conn Stays Open)
     |         |                  |         |
     |--SYN--->| (New TCP)        |--GET--->| (Request Obj 2)
     |<--ACK---|                  |<--File--|
     |--GET--->|                  |         |
     |<--File--|                  |         |
```

### Calculation Example: Retrieving a Web Page (Slide 50)
The following example quantifies the difference in performance between connection types.

**Scenario:**
*   A user wants to download a web page.
*   The page consists of **1 Base HTML file** and **10 small referenced objects** (e.g., images).
*   *Assumption:* Ignore the actual file transmission time (focus only on RTTs).

**Case 1: Non-persistent HTTP (Serial)**
In this mode, the client must download objects one at a time, opening a new connection for each.
*   **Cost for Base HTML:** 2 RTT (1 for TCP setup, 1 for Request).
*   **Cost for Objects:** 10 objects $\times$ 2 RTT per object = 20 RTT.
*   **Total:** $2 \text{ RTT} + 20 \text{ RTT} = \mathbf{22 \text{ RTT}}$.

**Case 2: Persistent HTTP**
In this mode, the connection stays open after the Base HTML is retrieved.
*   **Cost for Base HTML:** 2 RTT.
*   **Cost for Objects:** The client requests the subsequent 10 objects using the existing open connection. This requires only 1 RTT per object request cycle.
*   **Total:** $2 \text{ RTT} + 10 \text{ RTT} = \mathbf{12 \text{ RTT}}$.

**Case 3: Non-persistent HTTP with Parallel Connections**
In this mode, the browser opens parallel connections to fetch objects simultaneously. Assume **5 parallel connections**.
*   **Cost for Base HTML:** 2 RTT.
*   **Batch 1 (Objects 1-5):** 5 connections open simultaneously. Total time for this batch is 2 RTT.
*   **Batch 2 (Objects 6-10):** The next 5 connections open simultaneously. Total time for this batch is 2 RTT.
*   **Total:** $2 \text{ RTT (Base)} + 2 \text{ RTT (Batch 1)} + 2 \text{ RTT (Batch 2)} = \mathbf{6 \text{ RTT}}$.

### 5.5 #3: HTTP Messages (Slides 53-59)
HTTP involves two types of messages: Requests (Client to Server) and Responses (Server to Client).

**1. HTTP Request Message**
*   **Format:** ASCII (human-readable).
*   **Structure:**
    1.  **Request Line:** Contains the Method (GET, POST, HEAD), the URL, and the HTTP Version.
    2.  **Header Lines:** Metadata such as `Host:`, `User-Agent:` (browser type), `Connection:` (keep-alive), etc.
    3.  **End of Headers:** Indicated by an empty line (`\r\n`).
    4.  **Entity Body:** Used in POST requests to send data (e.g., form inputs) to the server.
*   **Methods:**
    *   **GET:** Request an object (data is in the URL).
    *   **POST:** Send data to the server (data is in the entity body).
    *   **HEAD:** Request headers only (for debugging or caching checks).
    *   **PUT:** Upload a file to a specific path.

**2. HTTP Response Message**
*   **Structure:**
    1.  **Status Line:** Protocol Version, **Status Code**, Status Phrase.
    2.  **Header Lines:** `Date:`, `Server:`, `Content-Length:`, `Content-Type:` (e.g., text/html).
    3.  **Entity Body:** The actual requested data (the HTML file or image).
*   **Common Status Codes:**
    *   **200 OK:** Request succeeded; object is in the body.
    *   **301 Moved Permanently:** Object has moved; new URL is in the headers.
    *   **400 Bad Request:** Server could not understand the message.
    *   **404 Not Found:** Document does not exist on the server.
    *   **505 HTTP Version Not Supported**.

### 5.6 #4: Web Advanced Features (Slide 61)
The basic Web is extended by several features to improve usability and performance:
*   **Cookies:** Used to maintain state (e.g., shopping carts, login status) despite HTTP being stateless.
*   **Web Caches (Proxy Servers):** Store copies of recent content closer to the user to reduce delay and network traffic.
*   **Conditional GET:** Allows a cache to verify if an object is up-to-date without downloading the whole file again ("If-Modified-Since").
*   **HTTP 2/3:** Newer versions focus on reducing latency through multiplexing and other optimizations.

***

## Key Terms
*   **Client-Server Paradigm:** Architecture where always-on servers provide services to requesting clients.
*   **Peer-to-Peer (P2P):** Architecture where peers communicate directly, acting as both servers and clients; highly scalable.
*   **Process:** A program running within a host.
*   **Socket:** The interface between the application process and the transport layer protocol (the "door").
*   **IP Address:** 32-bit unique identifier for a host.
*   **Port Number:** Identifier for a specific process on a host (e.g., 80 for Web).
*   **TCP (Transmission Control Protocol):** Connection-oriented, reliable transport service with flow/congestion control.
*   **UDP (User Datagram Protocol):** Connectionless, unreliable transport service without rate control.
*   **HTTP (Hypertext Transfer Protocol):** The application-layer protocol for the Web.
*   **Stateless Protocol:** A protocol (like HTTP) where the server maintains no information about past client requests.
*   **Non-persistent HTTP:** Opens a new TCP connection for each object request.
*   **Persistent HTTP:** Reuses the same TCP connection for multiple object requests.
*   **RTT (Round-Trip Time):** Time for a small packet to travel from client to server and back.
*   **Parallel TCP Connections:** Opening multiple connections simultaneously to download objects faster.
*   **Request/Response Messages:** The two types of messages exchanged in HTTP.
*   **Status Codes:** Codes (e.g., 200, 404) indicating the result of an HTTP request.
