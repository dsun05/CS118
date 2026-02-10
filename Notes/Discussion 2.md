## 1.0 Course Overview
### 1.1 Agenda
The discussion session covers the following key areas to bridge theoretical lecture concepts with practical implementation details required for upcoming assignments:
*   **Socket Programming Review:** A deep dive into the C-based API for TCP connections.
*   **Lecture Review:** Reinforcing concepts regarding packet delay, loss, throughput, and the application layer.
*   **Applications:** Specifically focusing on HTTP.
*   **Project 1 Preview:** Introduction to HTTPS, SSL/TLS, and the reverse proxy architecture.
*   **Homework 1 Q&A:** Addressing statistical questions related to network traffic modeling.

***

## 2.0 Socket Programming Review
### 2.1 TCP Socket Flow
Understanding the sequence of function calls is critical for establishing a TCP connection. The client and server have distinct lifecycles that must synchronize via a "handshake."

*   **Visual Aid:** TCP Client vs. TCP Server Function Call Sequence
```text
      TCP Client                                   TCP Server
      ==========                                   ==========
      socket()                                     socket()
         |                                            |
         |                                         bind()
         |                                            |
         |                                         listen()
         |                                            |
      connect() -------------------------------->  accept()
         |        (3-Way Handshake)                   | <--- Blocks until connection
         |                                            |      arrives
      write()   -------------------------------->  read()
         |              Data Request                  |
         |                                         process request
         |                                            |
      read()    <--------------------------------  write()
         |              Data Reply                    |
         |                                            |
      close()   -------------------------------->  read() -> returns 0 (EOF)
                                                      |
                                                   close()
```

*   **TCP Server Lifecycle:**
    1.  **`socket()`**: Creates the endpoint.
    2.  **`bind()`**: Associates the socket with a specific IP and Port.
    3.  **`listen()`**: Puts the socket in a passive mode, creating a queue for incoming connection requests.
    4.  **`accept()`**: This is a **blocking** call. The server process pauses here until a client tries to connect. It returns a *new* file descriptor specifically for this connection.
    5.  **`read()` / `process` / `write()`**: The server reads the request, acts on it, and sends a response.
    6.  **`close()`**: Terminates the connection.

*   **TCP Client Lifecycle:**
    1.  **`socket()`**: Creates the endpoint.
    2.  **`connect()`**: Initiates the 3-way handshake with the server.
    3.  **`write()` / `read()`**: Sends the request and waits for the response.
    4.  **`close()`**: Terminates the connection.

### 2.2 Server Implementation Details
Writing a server in C involves manipulating specific data structures to define network addresses.

*   **Variable Setup:**
    *   **File descriptors:** You typically need two—`sockfd` (the main listener) and `new_fd` (the descriptor generated for each specific client connection).
    *   **Address structs:** `struct sockaddr_in` is used to hold internet addresses. You need `my_addr` (server info) and `their_addr` (incoming client info).

*   **Step 1: Create Socket:**
    *   Call: `socket(PF_INET, SOCK_STREAM, 0)`
    *   `PF_INET`: IPv4 Protocol Family.
    *   `SOCK_STREAM`: Indicates TCP (reliable, byte-stream).
    *   **Error Handling:** Always check if the return value is `-1`.

*   **Step 2: Set Address Info:**
    *   `my_addr.sin_family = AF_INET;`
    *   `my_addr.sin_port = htons(MYPORT);` — **Crucial:** You must use `htons` (host to network short) to convert the integer port number to Network Byte Order (Big Endian).
    *   `my_addr.sin_addr.s_addr = htonl(INADDR_ANY);` — `INADDR_ANY` tells the OS to bind to *all* available network interfaces on the machine.
    *   **Zeroing:** Use `memset` (or `bzero`) to clear the rest of the struct padding to zero.

*   **Step 3: Bind:**
    *   `bind()` assigns the address specified in Step 2 to the socket created in Step 1.

*   **Step 4: Listen:**
    *   `listen(sockfd, BACKLOG)`: `BACKLOG` defines how many pending connections can queue up before the kernel starts rejecting them.

*   **Step 5: Accept Loop:**
    *   The server usually runs in a `while(1)` loop.
    *   `accept()` extracts the first connection request on the queue of pending connections.
    *   It fills `their_addr` with the client's information.
    *   **Logging:** You can use `inet_ntoa` (Network to ASCII) to print the client's IP address.
    *   **Cleanup:** After handling the request, `close(new_fd)` is called to free resources for that specific connection.

### 2.3 Client Implementation Details
The client side is generally simpler as it does not need to bind to a specific port or accept incoming links.

*   **Step 1: Create Socket:** Identical to the server (`socket()`).
*   **Step 2: Set Server Address:**
    *   The client must know the Server's IP and Port beforehand.
    *   If using a hostname, `gethostbyname` (deprecated) or `getaddrinfo` is used to resolve the IP.
*   **Step 3: Connect:**
    *   `connect()` uses the socket and the server's address struct to establish the link. If this fails (returns -1), the server is likely down or unreachable.

### 2.4 Common Coding Errors & Fixes
Several pitfalls exist when working with the C socket API.

*   **Visual Aid:** Common Coding Errors
```c
// ERROR: Using wrong type for size
int sin_size;
// FIX: Must use specific socket type
socklen_t sin_size; 

// ERROR: Using UDP type for TCP socket
socket(AF_INET, SOCK_DGRAM, 0); 
// FIX:
socket(AF_INET, SOCK_STREAM, 0);

// ERROR: Forgetting Byte Order conversion
my_addr.sin_port = MYPORT;
// FIX: Network Byte Order required
my_addr.sin_port = htons(MYPORT); 

// ERROR: accept() signature mismatch
accept(sockfd, &their_addr, sin_size);
// FIX: Must pass pointer to size
accept(sockfd, (struct sockaddr *)&their_addr, &sin_size);
```

*   **Size Variables:** `accept` requires a pointer to the size variable so it can update it. This variable must be of type **`socklen_t`**, not a standard `int`.
*   **Bind Size:** When binding, pass `sizeof(struct sockaddr_in)`.
*   **Forking/Child Processes:** When a server uses `fork()` to handle concurrent clients:
    *   **Child Process:** Inherits file descriptors. It should close the `sockfd` (listener) because it only cares about the active connection. It processes data and then exits.
    *   **Parent Process:** Should close `new_fd` immediately after forking, so it can go back to listening. It must also handle child cleanup (zombie processes) using `waitpid`.
*   **Sending Data:** Do not use `sendto()` on a TCP socket; that is for UDP. Use `send()` or `write()`.

***

## 3.0 Lecture Review: Packet Delay & Throughput
### 3.1 Packet Delay Components
The time it takes for a packet to travel from node A to node B is the sum of four specific delays at each node along the path.

*   **Visual Aid:** Nodal Delay Model
```text
        [Node A]
           |
   +-------v-------+
   | Processing    |  <-- d_proc: Check errors, route lookup
   +-------+-------+
           |
   +-------v-------+
   | Queueing      |  <-- d_queue: Waiting for turn
   +-------+-------+
           |
   +-------v-------+
   | Transmission  |  <-- d_trans: Pushing bits onto wire (L/R)
   +-------+-------+
           |
   =================  <-- d_prop: Physical travel time (d/s)
           |
        [Node B]
```

*   **Total Nodal Delay Formula:** $d_{nodal} = d_{proc} + d_{queue} + d_{trans} + d_{prop}$
    *   **$d_{proc}$ (Nodal Processing):** Time required to examine the packet header, check for bit-level errors, and determine the output link. Usually microseconds.
    *   **$d_{queue}$ (Queueing Delay):** Time waiting at the output link for transmission. This is the most variable component and depends entirely on network congestion.
    *   **$d_{trans}$ (Transmission Delay):** The time required to push all the packet's bits into the link.
        *   Formula: $L/R$ (Packet length in bits / Link bandwidth in bps).
    *   **$d_{prop}$ (Propagation Delay):** The time required for a bit to travel physically from the sender to the receiver.
        *   Formula: $d/s$ (Distance of physical link / Propagation speed of medium).

### 3.2 Extra Material: Queue Overflow
*   **Scenario:** What happens if the rate at which packets arrive (emission) is exactly the same as the rate they can be sent (transmission)?
*   **Random Walk Model:** We can model this as a random walk. If a queue behaves like a random walk, math dictates that with probability 1, the walk will eventually reach *any* fixed level $a$.
*   **Conclusion:** Even if the average arrival rate equals the service rate, natural fluctuations mean the queue size will eventually hit any finite limit. Therefore, **finite buffers will eventually overflow** (packet loss) if utilization remains at 100%.

### 3.3 Practice Calculations
*   **Propagation Delay:**
    *   *Calculation:* $2500 \text{ km} / (2.5 \times 10^8 \text{ m/s})$.
    *   $2.5 \times 10^6 \text{ meters} / 2.5 \times 10^8 \text{ m/s} = 0.01 \text{ seconds} = 10 \text{ ms}$.
    *   **Concept:** This delay does **not** depend on packet length ($L$) or transmission rate ($R$).

*   **Throughput:**
    *   **Definition:** The rate (bits/time) at which bits are actually transferred between sender and receiver.
    *   **Bottleneck Link:** In a path with multiple links (e.g., $R_1, R_2, R_3$), the end-to-end throughput is constrained by the slowest link.
    *   *Example:* If links are 500kbps, 2Mbps, and 1Mbps, the maximum throughput is **500kbps**.

*   **File Transfer Time:**
    *   Formula: File Size / Throughput.
    *   *Example:* 4 million bytes over a 500kbps bottleneck.
    *   Size in bits: $4 \times 10^6 \times 8 = 32 \times 10^6$ bits.
    *   Time: $32,000,000 / 500,000 = 64$ seconds.

### 3.4 Bottleneck Link Concept
*   **Visual Aid:** The Bottleneck Topology
```text
      Fast Pipe         Slow Pipe         Medium Pipe
    [===========] --- [=====] --- [========]
       2 Mbps         500 kbps       1 Mbps

    Result: Flow is limited to 500 kbps.
```
*   **Pipeline Effect:** When sending a large file broken into many packets, the transmission delays of the faster links effectively overlap with the slow link. The delays of the fast links are "hidden" by the pipelining process.
*   **Conclusion:** The throughput of the bottleneck link provides a very accurate approximation of the overall transmission time for large data transfers.

***

## 4.0 Applications & HTTP
### 4.1 Application Layer Models
*   **Client-Server:**
    *   There is an always-on host (Server) and intermittent hosts (Clients).
    *   Examples: Web (HTTP/TCP), FTP (TCP), Email (SMTP/TCP), DNS (UDP or TCP).
*   **Peer-to-Peer (P2P):**
    *   No always-on server; arbitrary end systems communicate directly. Highly scalable.
    *   Examples: BitTorrent, Tor (Onion Routing).
*   **Hybrid:**
    *   Combines elements of both.
    *   Examples: Skype (uses centralized servers for finding users, but P2P for the voice/video stream).

### 4.2 HTTP Overview
*   **Protocol:** HyperText Transfer Protocol. It is a **stateless** protocol running on top of **TCP**.
*   **Model:** It uses a **pull** model (client requests data, server sends it).
*   **Connection Types:**
    *   **Non-persistent (HTTP 1.0):** Opens TCP connection $\rightarrow$ Sends 1 object $\rightarrow$ Closes connection. Requires 2 RTT (Round Trip Time) per object (one for handshake, one for data).
    *   **Persistent (HTTP 1.1):** Opens TCP connection $\rightarrow$ Sends multiple objects over the same link $\rightarrow$ Closes eventually. Reduces overhead.
*   **Methods:** `GET` (retrieve), `POST` (submit data), `HEAD` (headers only), `PUT` (upload), `DELETE`.
*   **State Management:** Since HTTP is stateless, **Cookies** are used to maintain session state (e.g., shopping carts) across requests.
*   **Performance:** Web Caches (Proxy Servers) store copies of recent data to satisfy requests locally, reducing delay.

### 4.3 HTTP Message Formats
*   **Request Message:**
    *   **Visual Aid:**
    ```text
    [ Method (GET) | URL | Version (HTTP/1.1) ]  <-- Request Line
    [ Header Field Name: Value                ]  <-- Headers
    [ ...                                     ]
    [ CRLF (Empty Line)                       ]
    [ Body (optional, used in POST)           ]
    ```
*   **Response Message:**
    *   **Visual Aid:**
    ```text
    [ Version | Status Code (200) | Phrase (OK) ] <-- Status Line
    [ Header Field Name: Value                  ] <-- Headers
    [ ...                                       ]
    [ CRLF                                      ]
    [ Data (HTML, Image, etc.)                  ] <-- Entity Body
    ```

### 4.4 Practical Analysis
*   **Telnet:** A tool that allows you to open a raw TCP connection to a port (e.g., 80) and type HTTP commands manually to test server responses.
*   **Wireshark Analysis:**
    *   **`User-Agent`:** Identifies the browser. Modern browsers often include "Mozilla/5.0" for historical compatibility reasons (to avoid being blocked by old servers that used "User-Agent sniffing" to serve content only to Netscape/Mozilla).
    *   **`Keep-Alive`:** Indicates a persistent connection is desired.
    *   **`ETag`:** Used for cache validation.
    *   **`Date` vs `Last-Modified`:** `Date` is when the response was sent; `Last-Modified` is when the file on the server actually changed.

***

## 5.0 Project 1: HTTPS & Security
### 5.1 HTTP vs HTTPS
*   **HTTP:** Sends data in plaintext. Anyone on the network (WiFi sniffer, ISP, router admin) can read the data. Vulnerable to snooping and Man-in-the-Middle (MITM) attacks.
*   **HTTPS:** Wraps HTTP inside **TLS/SSL** (Transport Layer Security).

### 5.2 SSL/TLS Fundamentals
A secure channel must provide three core properties (CIA):
1.  **Confidentiality (Encryption):** Only the sender and receiver can read the data.
2.  **Authentication:** Proves the server is who it claims to be (preventing impersonation).
3.  **Integrity:** Ensures data was not altered in transit.

### 5.3 Certificates (PKI)
*   **Function:** A digital certificate binds a Public Key to an Identity (domain name).
*   **Chain of Trust:** Certificates are digitally signed by Certificate Authorities (CAs). Browsers trust the CAs, and CAs trust the website certificates.
*   **Project Context:** You will use **self-signed certificates**. Browsers will warn that they are "unsafe" because they aren't signed by a known CA, but they still provide encryption for localhost testing.

### 5.4 OpenSSL Library
You will use the OpenSSL C library to implement TLS.
*   **Key Structures:**
    *   `SSL_CTX`: Global context/configuration (loads the certificate and private key).
    *   `SSL`: A structure representing a specific active connection.
*   **New I/O Functions:**
    *   Instead of `recv()`, use `SSL_read()`.
    *   Instead of `send()`, use `SSL_write()`.
    *   These functions handle the encryption/decryption transparently.

*   **Server Implementation Flow:**
    1.  Initialize OpenSSL.
    2.  Create `SSL_CTX` and load certs.
    3.  Create a standard TCP `socket()`, `bind()`, `listen()`.
    4.  `accept()` a standard TCP connection.
    5.  Create an `SSL` object and attach it to the socket.
    6.  `SSL_accept()`: Performs the TLS handshake.
    7.  Use `SSL_read()`/`SSL_write()` for data.
    8.  `SSL_shutdown()`/`free()` to clean up.

### 5.5 SSL Termination (Reverse Proxy)
The project requires building a "Reverse Proxy" that handles security.

*   **Visual Aid:** Reverse Proxy Flow
```text
    [Browser]  ----(HTTPS/Encrypted)--->  [Your Proxy]  ----(HTTP/Plain)--->  [Backend/Video Server]
```
*   **Why?** This offloads CPU-intensive encryption from the backend servers and creates a centralized point for certificate management.
*   **Project Logic:**
    *   **Local Files:** If the request is for a local file, the Proxy decrypts the request, reads the file from disk, and sends it back encrypted via `SSL_write`.
    *   **Video Files (.ts, .m3u8):**
        1.  Proxy receives HTTPS request (via `SSL_read`).
        2.  Proxy opens a *new* plain TCP socket to the Video Server.
        3.  Proxy forwards the request (plain `send`).
        4.  Proxy receives video data (plain `recv`).
        5.  Proxy encrypts and forwards data to Browser (via `SSL_write`).

***

## 6.0 Homework 1 Q&A
*   **Topic:** Central Limit Theorem (CLT) (Problem 3).
*   **Visual Aid:** Normal Distribution
```text
           /   \
         /       \
       /           \
     _/             \_
```
*   **Concept:** The problem involves the **Lindeberg-Lévy CLT**. It states that if you have a sequence of i.i.d (independent and identically distributed) random variables, as the sample size ($n$) approaches infinity, the distribution of the sample mean converges to a **Normal Distribution** $\mathcal{N}(0, \sigma^2)$.
*   **Application:** This is used to model aggregate traffic behavior in networks (e.g., summing up bitrates from many users).

***

## 7.0 Key Terms
*   **Throughput:** The actual rate (bits/time unit) at which data is successfully transferred across a network path.
*   **Bottleneck Link:** The specific link in a network path with the lowest transmission capacity, which limits the maximum possible throughput for the entire path.
*   **Transmission Delay:** The time required to push all bits of a packet onto the wire ($L/R$).
*   **Propagation Delay:** The time it takes for a signal to physically travel through the medium from sender to receiver ($d/s$).
*   **Stateless Protocol:** A protocol (like HTTP) that treats each request as an independent transaction that is unrelated to any previous request.
*   **Persistent HTTP:** A mode of HTTP where the TCP connection remains open after a request is served, allowing subsequent requests to use the same connection to save overhead.
*   **SSL/TLS:** (Secure Sockets Layer / Transport Layer Security) Protocols designed to provide communications security over a computer network via encryption, authentication, and integrity.
*   **Reverse Proxy:** An intermediary server that sits between clients and backend servers. In Project 1, it handles the SSL encryption/decryption (termination) so the backend servers don't have to.
