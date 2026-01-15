# Lecture Outline: CS118 Discussion 1A, Week 1

## 1.0 Logistics & Course Overview

### 1.1 Instructor Information
*   **Instructor:** Ziyue Dang (3rd Year CS PhD Student @ UCLA).
*   **Office Hours:** Mondays 9:00 am – 11:00 am, Engineering VI 392.
*   **Email Policies:**
    *   **Address:** `ziyue.dang@cs.ucla.edu`
    *   **Subject Line:** Must include `[CS118]` to prevent the email from being flagged as spam.
    *   **Content:** Always include your full Name and UID.
    *   **Response Time:** Allow 24 hours for a reply. If no response is received within that window, feel free to follow up.

### 1.2 Grading Breakdown
*   **Homework:** 18%
*   **Programming Projects:** 25%
    *   **Project 1 (10%):** Implementation of a simple web server to establish familiarity with network programming.
    *   **Project 2 (15%):** Implementation of TLS (Transport Layer Security). *Subject to change.*
    *   **Bonus Credit:** Up to 10% extra work is available within the projects (e.g., Project 1 offers 1.0 bonus points; Project 2 offers 1.5).
    *   **Test Environment:** Projects will be graded in an **Ubuntu virtual machine**.
*   **Exams:** 54%
    *   **Midterm:** 27% (Covers Parts 1, 2, and 3).
    *   **Final:** 27% (Cumulative).
*   **Participation:** 3%
    *   Measured via random sign-in sheets distributed during discussions throughout the quarter.

### 1.3 Homework Submission Guidelines
*   **Platform:** All assignments must be submitted via **Gradescope**.
*   **Deadlines:**
    *   **Hard Deadline:** Fridays at 6:00 pm.
    *   **Policy:** The system accepts multiple resubmissions up until the deadline. No submissions are accepted after the deadline.
*   **Format:**
    *   Answers must be written in the dedicated box or on a separate sheet.
    *   **Alignment:** You must align your uploaded answers with the specific questions in the Gradescope interface.
    *   **Scanning:** If handwriting or drawing diagrams, scan the paper to a **high-quality black and white PDF** (e.g., using a smartphone scanner app). Low-quality or inaccessible uploads will result in low scores.

### 1.4 Course Syllabus (Top-Down Approach)
The course follows a "Top-Down" approach, starting from the layer closest to the user (Application) and moving down to the physical transmission.

*   **Part 1:** Introduction (Chapter 1).
*   **Part 2:** Application Layer (Chapter 2).
    *   *Focus:* Introduction to socket programming.
*   **Part 3:** Transport Layer (Chapter 3).
    *   *Key Date:* **Project 1 due October 27 (Friday)**.
    *   *Key Date:* **Midterm Exam November 7 (Tuesday)**.
*   **Part 4:** Network Layer (Chapters 4 and 5).
*   **Part 5:** Link Layer, LANs (Chapter 6).
*   **Part 6:** Wireless and Mobile Networks (Chapter 7).
*   **Part 8:** Network Security (Chapter 8).
    *   *Key Date:* **Project 2 due December 8 (Friday)**.
    *   *Key Date:* **Final Exam December 14 (Thursday)**.

***

## 2.0 Network Programming Models

### 2.1 Core Concepts
*   **Why Networks?** Networking is arguably the most impactful area of computer science regarding daily life. As noted by Keith Winstein, networking and communications have produced the tangible changes the average person interacts with most over the last two decades.
*   **Visual Aid: Global Connectivity**
    ```text
          [Cloud/Server Cluster]
                 /   \
                /     \
        [Laptop]-------[Smartphone]
           |              |
        [Router]-------[IoT Device]
    ```

### 2.2 The Client-Server Model
This is the standard model for network programming, characterized by **asymmetric communication**.

*   **Client:**
    *   **Role:** The active initiator of communication.
    *   **Behavior:** Requests data and waits for a response.
    *   **Multiplicity:** Multiple clients can connect to a single server simultaneously.
*   **Server (Daemon):**
    *   **Role:** A passive entity (often a background process or "daemon").
    *   **Behavior:** It listens on a specific IP and Port, waits for incoming connections, processes requests, and sends replies.
*   **Architecture:**
    *   **Centralized:** All communication flows through the server; clients do not typically communicate directly with one another in this model.
*   **Relationships:**
    *   Roles are logical, not physical. A machine acting as a server for one process can act as a client for another (e.g., a web server acting as a database client).
*   **Server Service Models:**
    *   **Concurrent:** The server processes multiple client requests simultaneously (e.g., using threads or `fork()`).
    *   **Sequential:** The server processes requests one at a time (FIFO).
    *   **Hybrid:** The server accepts multiple connections but generates responses sequentially.

### 2.3 Layer Context
Network programming takes place at the boundary between the Application Layer and the Transport Layer.

*   **Application Layer:** Contains the logic for "Clients" and "Servers" (e.g., Web Browser, Web Server).
*   **Transport Layer:** Provides the underlying communication tunnels (TCP/UDP). The application accesses these via the **Socket API**.

*   **Visual Aid: The Layer Stack**
    ```text
    +---------------------+              +---------------------+
    |      Client App     |              |      Server App     |
    +---------------------+              +---------------------+
              ^                                     ^
              | Socket API (Interface)              | Socket API
              v                                     v
    +---------------------+              +---------------------+
    |   Transport (TCP)   |<------------>|   Transport (TCP)   |
    +---------------------+              +---------------------+
    |       Network       |              |       Network       |
    +---------------------+              +---------------------+
    |      Data Link      |              |      Data Link      |
    +---------------------+              +---------------------+
    |       Physical      |<------------>|       Physical      |
    +---------------------+              +---------------------+
    ```

### 2.4 Transport Layer Protocols
Applications generally choose between two standard protocols provided by the Transport Layer:

*   **TCP (Transmission Control Protocol):**
    *   **Connection-oriented:** Requires a handshake (connection setup) before data exchange.
    *   **Reliable Data Transfer:** Guarantees delivery; data is acknowledged.
    *   **No Duplicates:** The protocol handles retransmission and deduplication.
    *   **Ordered Transfer:** Data arrives at the application in the exact order it was sent (FIFO).
    *   **Flow & Congestion Control:** Regulates data flow to prevent overwhelming the receiver or the network.
    *   **Analogy:** A phone call (connection established, reliable conversation).
*   **UDP (User Data Protocol):**
    *   **Connectionless:** No handshake; simply send data to an address.
    *   **Datagrams:** Data is sent in discrete packets (variable length) rather than a stream.
    *   **Unreliable:** No guarantee of delivery, ordering, or deduplication.
    *   **No Flow Control:** Sends as fast as possible.
    *   **Analogy:** Sending a physical letter (fire and forget, might get lost).

***

## 3.0 Socket Programming APIs

### 3.1 Introduction to Sockets
*   **Definition:** A socket is an endpoint for inter-process communication (IPC) across a computer network.
*   **Identification:** A socket is uniquely identified by a tuple: `<IP Address : Port Number>`.
*   **Function:** It acts as the "door" or "tunnel" builder between the Application Layer and the Transport/Network layers.
*   **Visual Aid: Socket Interface**
    ```text
    [ Application Layer ]
             |
    ---------| <--- Socket API (BSD Sockets)
             |
    [   TCP / UDP      ]
             |
    [       IP         ]
             |
    [ Network Device   ]
    ```

### 3.2 Port Numbers
Ports allow the OS to distinguish between different network services running on the same IP address. (Reference: RFC 1700).

*   **Common Ports:**
    *   **HTTP:** 80
    *   **HTTPS:** 443
*   **Ranges:**
    *   **1 - 512:** Standard services (requires super-user/root privileges).
    *   **513 - 1023:** Registered and controlled (requires super-user/root privileges).
    *   **1024 - 49151:** Registered services / Ephemeral ports (accessible by standard users).
    *   **49152 - 65535:** Private / Ephemeral ports.

### 3.3 TCP Socket Workflow (Diagram Evolution)
The lifecycle of a TCP connection differs for the client and the server.

*   **Visual Aid: Client-Server Interaction Flowchart**
    ```text
          TCP CLIENT                                 TCP SERVER
          ----------                                 ----------
              |                                          |
        1. socket()                                1. socket()
              |                                          |
              |                                    2. bind()
              |                                          |
              |                                    3. listen()
              |                                          |
              |                                    4. accept()
              |                                    (Blocks & waits)
              |                                          |
        2. connect() ----------------------------------->|
         (Handshake)                                     |
              |                                    (Unblocks)
              |                                          |
        3. write()                                 5. read()
        (Request Data) --------------------------------->|
              |                                   (Process Request)
              |                                          |
              | <---------------------------------- 6. write()
        4. read()                                 (Send Reply)
        (Receive Reply)                                  |
              |                                          |
        5. close()                                 7. read()
              | <----------------------------------- (Returns 0/EOF)
              |                                          |
              |                                    8. close()
    ```
    *   **Note:** The server's `accept()` call blocks program execution until a client initiates a connection via `connect()`.

***

## 4.0 System Calls & Data Structures

### 4.1 Essential Structs
In C network programming, specific structures are used to handle addressing.

*   **`int sockfd`:** The socket descriptor. It is simply an integer (file descriptor) referencing the socket.
*   **`struct sockaddr`:** A generic structure for socket address information.
    ```c
    struct sockaddr {
        unsigned short sa_family; // Address family (e.g., AF_INET)
        char sa_data[14];         // 14 bytes of protocol address
    };
    ```
*   **`struct sockaddr_in`:** The IPv4-specific implementation of the address structure.
    ```c
    struct sockaddr_in {
        short sin_family;             // e.g., AF_INET
        unsigned short sin_port;      // Port number (Network Byte Order)
        struct in_addr sin_addr;      // Internet Address
        unsigned char sin_zero[8];    // Padding to match sockaddr size
    };
    ```
    *   **Crucial Note:** You must cast `sockaddr_in*` to `sockaddr*` when passing it to system calls like `bind` or `connect`.

### 4.2 Core System Calls

1.  **`socket(int domain, int type, int protocol)`**
    *   Creates a generic socket.
    *   **Params:** `PF_INET` (IPv4), `SOCK_STREAM` (TCP) or `SOCK_DGRAM` (UDP).
    *   **Returns:** Socket descriptor (int) or -1 on failure.

2.  **`bind(int sockfd, struct sockaddr* myaddr, int addrlen)`**
    *   Associates a socket with a specific local IP address and Port.
    *   Used primarily by the **Server** to claim a specific port (e.g., 80).

3.  **`listen(int sockfd, int backlog)`**
    *   Converts the socket into a passive state, ready to accept incoming connection requests.
    *   **`backlog`:** Defines the maximum size of the queue for pending connections (clients trying to connect while the server is busy).

4.  **`accept(int sockfd, struct sockaddr* client_addr, int* addrlen)`**
    *   Extracts the first connection request on the queue.
    *   **Returns:** A **new** file descriptor specifically for this client connection. The original `sockfd` remains listening for *other* new connections.
    *   **Blocking:** By default, this call sleeps until a client connects.
    *   **Non-Blocking:** If configured, returns -1 if no connection is pending.

5.  **`connect(int sockfd, struct sockaddr* server_addr, int addrlen)`**
    *   Used by the **Client** to initiate the 3-way handshake with the server.

6.  **`write(int sockfd, char* buf, size_t nbytes)`**
    *   Sends data over the TCP stream.

7.  **`read(int sockfd, char* buf, size_t nbytes)`**
    *   Reads data from the TCP stream.
    *   **Returns:** Number of bytes read. **Returns 0** if the peer has closed the connection.

8.  **`close(int sockfd)`**
    *   Terminates the connection and invalidates the file descriptor.

***

## 5.0 Data Representation & Utilities

### 5.1 Byte Ordering (Endianness)
Different computer architectures store multi-byte integers differently.
*   **Little Endian:** The Least Significant Byte (LSB) is stored at the lowest memory address (e.g., x86 architectures).
*   **Big Endian:** The Most Significant Byte (MSB) is stored at the lowest memory address.
*   **Network Byte Order:** The Internet standard is **Big Endian**.

*   **Visual Aid: Memory Representation (0xAABB)**
    ```text
    Value: 0xAABB

    Address:   |  0x1000  |  0x1001  |
               +----------+----------+
    Big Endian:|   0xAA   |   0xBB   |  <-- Network Standard
               +----------+----------+
    Little End:|   0xBB   |   0xAA   |  <-- Typical Host (x86)
               +----------+----------+
    ```

*   **Conversion Functions:**
    *   `htons()` / `htonl()`: **H**ost **to** **N**etwork (short/long). Call before **sending/binding**.
    *   `ntohs()` / `ntohl()`: **N**etwork **to** **H**ost (short/long). Call after **reading**.

### 5.2 Address Utility Functions
*   **`struct hostent* gethostbyname(const char* name)`:** Performs DNS lookup (e.g., turns "google.com" into an IP struct).
    *   Returns a `struct hostent` containing the canonical name, aliases, and address list.
*   **`inet_ntoa(struct in_addr in)`:** Converts a binary IP struct into a human-readable dotted-decimal string (e.g., "192.168.1.1").
*   **`inet_addr(const char* cp)`:** Converts a dotted-decimal string into a Network Byte Order binary value (returns `in_addr_t`).
*   **`gethostname()`:** Retrieves the name of the local computer.

***

## 6.0 Implementation Examples

### 6.1 Writing a Server (C Code)
The server implementation generally follows the `socket` -> `bind` -> `listen` -> `accept` loop pattern.

*   **Visual Aid: Server Header & Setup**
    ```c
    #include <stdio.h>
    #include <string.h>
    #include <sys/socket.h>
    #include <netinet/in.h>

    #define MYPORT 5000  // Choose a port > 1024
    #define BACKLOG 10

    int main() {
        int sockfd, new_fd;
        struct sockaddr_in my_addr;
        struct sockaddr_in their_addr; // Client's info
        int sin_size;

        // 1. Create Socket
        sockfd = socket(PF_INET, SOCK_STREAM, 0);

        // 2. Setup Address Struct
        my_addr.sin_family = AF_INET;
        my_addr.sin_port = htons(MYPORT);     // Host to Network Short
        my_addr.sin_addr.s_addr = htonl(INADDR_ANY); // Bind to all local IPs
        memset(my_addr.sin_zero, '\0', 8);    // Zero padding

        // 3. Bind
        bind(sockfd, (struct sockaddr *)&my_addr, sizeof(struct sockaddr));

        // 4. Listen
        listen(sockfd, BACKLOG);

        // 5. Accept Loop
        while(1) {
            sin_size = sizeof(struct sockaddr_in);
            // Blocks here until client connects
            new_fd = accept(sockfd, (struct sockaddr *)&their_addr, &sin_size);

            printf("server: got connection from %s\n",
                   inet_ntoa(their_addr.sin_addr));

            // Handle connection (read/write)...

            close(new_fd); // Close client connection, keep listening
        }
    }
    ```

### 6.2 Writing a Client (C Code)
The client requires the server's IP (or hostname) to initiate a connection.

*   **Visual Aid: Client Implementation**
    ```c
    int main() {
        int sockfd;
        struct sockaddr_in their_addr;
        struct hostent *he;

        // Resolve Hostname
        he = gethostbyname("server.example.com");

        // 1. Create Socket
        sockfd = socket(PF_INET, SOCK_STREAM, 0);

        // 2. Setup Server Info
        their_addr.sin_family = AF_INET;
        their_addr.sin_port = htons(5000);
        their_addr.sin_addr = *((struct in_addr *)he->h_addr);
        memset(their_addr.sin_zero, '\0', 8);

        // 3. Connect
        connect(sockfd, (struct sockaddr *)&their_addr, sizeof(struct sockaddr));

        // ... read/write operations ...

        close(sockfd);
        return 0;
    }
    ```

***

## 7.0 Conclusion & Resources

### 7.1 Summary
*   We reviewed the **Client-Server model** and its asymmetric nature.
*   We compared **TCP** (reliable, connection-oriented) vs **UDP** (unreliable, packet-based).
*   We detailed the **Socket API** lifecycle: `socket`, `bind`, `listen`, `accept` (Server) vs `connect` (Client).
*   We highlighted the importance of **Network Byte Order** (Big Endian) conversions.

### 7.2 Further Reading
*   *UNIX Network Programming*, Vol. 1 by Stevens, Fenner, & Rudoff (The standard reference).
*   *Beej’s Guide to Network Programming* (Highly recommended, accessible online resource).
*   `man` pages (e.g., `man socket`, `man bind`) in the Linux terminal.

***

### **Key Terms**
*   **Socket:** An endpoint for inter-process communication across a network identified by an IP address and port.
*   **Client-Server Model:** A centralized architecture where clients initiate requests and servers process them.
*   **TCP (Transmission Control Protocol):** A reliable, connection-oriented, full-duplex protocol providing ordered data delivery and flow control.
*   **UDP (User Datagram Protocol):** A simple, connectionless protocol with no guarantees on reliability or ordering.
*   **Datagram:** The basic unit of data transfer in UDP.
*   **Byte Stream:** The continuous flow of data transmission used in TCP.
*   **Network Byte Order:** The standard byte order for networks (Big Endian).
*   **Little Endian:** Least significant byte is stored at the lowest memory address.
*   **Big Endian:** Most significant byte is stored at the lowest memory address.
*   **Syscalls:** System calls used for network programming (socket, bind, listen, accept, connect, read, write, close).
*   **struct sockaddr_in:** A C structure used to represent an IPv4 internet address and port.
