## 1.0 What is network security?

### 1.1 What is network security?
Network security is a broad field focused on protecting data and systems during transmission. To achieve a secure network, four primary pillars must be maintained:
*   **Confidentiality**: Only the sender and the intended receiver should be able to "understand" the message contents. This is typically achieved when the sender encrypts the message and the receiver decrypts it. Interceptors should only see unintelligible data.
*   **Authentication**: Both the sender and receiver need to confirm the identity of the other party. In a digital network, you cannot physically see who you are talking to, making cryptographic proof of identity essential.
*   **Message Integrity**: The sender and receiver want to ensure the message was not altered (either maliciously or accidentally) while in transit, or altered afterwards without detection.
*   **Access and Availability**: Security is not just about locking things down; services must remain accessible and available to legitimate users. Denial-of-service attacks specifically target this pillar.

***

### 1.2 Friends and enemies: Alice, Bob, Trudy
In the network security world, cryptographic scenarios are traditionally explained using a cast of characters. **Alice** and **Bob** represent two entities that want to communicate "securely," while **Trudy** represents a malicious intruder. Trudy has the capability to *intercept, delete, or add* messages to the communication channel.

```text
* Visual Aid: Secure Communication vs. Intruder *

  Alice                                                       Bob
  (Secure Sender)                                             (Secure Receiver)
       |                                                           ^
       v                  [ The Network Channel ]                  |
    [Data] =======> (Data & Control Messages in Transit) =======> [Data]
                                    |
                                    v
                                  Trudy
                       (Intercepts, Deletes, Adds)
```

While Alice and Bob sound like individuals, in real life, they often represent networked systems:
*   A Web browser and a Web server executing electronic transactions (e.g., online shopping).
*   An online banking client and its corresponding bank server.
*   `DNS` servers exchanging domain records.
*   `BGP` routers exchanging routing table updates.

***

### 1.3 There are bad people out there!
What can "bad people" (like Trudy) actually do on a network?
*   **Eavesdrop**: Secretly intercept and read messages traversing the network.
*   **Actively insert**: Inject new, forged messages into an ongoing connection.
*   **Impersonation**: Fake or "spoof" a source IP address (or any other field in a packet) to pretend to be someone else.
*   **Hijacking**: "Take over" an ongoing connection by forcefully removing the legitimate sender or receiver and inserting themselves in their place.
*   **Denial of service (DoS)**: Prevent a service from being used by legitimate users, typically by overloading the server's resources with bogus traffic.

***

## 2.0 Principles of cryptography

### 2.1 The language of cryptography
Cryptography uses specific mathematical notations to describe its processes:
*   `m`: The plaintext message (the readable original data).
*   `K_A(m)`: The ciphertext (the unreadable data), which has been encrypted using key `K_A`.
*   `m = K_B(K_A(m))`: The decryption process, showing that applying key `K_B` to the ciphertext restores the original plaintext message `m`.

```text
* Visual Aid: The Cryptographic Process *

                 Alice's Key (K_A)                  Bob's Key (K_B)
                       |                                  |
                       v                                  v
[Plaintext, m] -> [Encryption Algorithm] =========> [Decryption Algorithm] -> [Plaintext, m]
                                             |
                                        [Ciphertext]
                                             |
                                           Trudy
```

***

### 2.2 Breaking an encryption scheme
Intruders use various methods to break encryption depending on the information they have:
*   **Cipher-text only attack**: Trudy only has access to the ciphertext. She must use either *brute-force* (trying every possible key) or *statistical analysis* (looking for recurring patterns in the data) to crack it.
*   **Known-plaintext attack**: Trudy has access to a piece of ciphertext *and* its corresponding plaintext. She can use this pairing to reverse-engineer the key (e.g., determining letter pairings in a substitution cipher).
*   **Chosen-plaintext attack**: Trudy has the ability to trick the sender into encrypting a plaintext message of her choosing, allowing her to analyze the resulting ciphertext to deduce the key.

***

### 2.3 Symmetric key cryptography
In **symmetric key cryptography**, Bob and Alice share the *exact same key* (`K_S`). This single key is used for both the encryption and decryption processes.

```text
* Visual Aid: Symmetric Key Cryptography *

                Shared Key (K_s)                    Shared Key (K_s)
                       |                                  |
                       v                                  v
[Plaintext, m] -> [Encryption Algorithm] =========> [Decryption Algorithm] -> [Plaintext, m]
                                             |
                                     [Ciphertext K_s(m)]
```
The primary challenge here is logistics: how do Bob and Alice securely agree on this shared key if they have never met?

***

### 2.4 AES: Advanced Encryption Standard
The **Advanced Encryption Standard (AES)** is the widely accepted symmetric-key standard established by NIST in Nov 2001. 
*   It processes data in 128-bit blocks.
*   It uses keys that are 128, 192, or 256 bits long.
*   To understand its strength: a brute-force decryption that takes 1 second to crack the older DES standard would take **149 trillion years** to crack AES.

***

### 2.5 Public Key Cryptography
**Public key cryptography** solves the key-distribution problem of symmetric crypto using a *radically* different mathematical approach (like RSA or Diffie-Hellman). 
*   The sender and receiver do *not* share a secret key.
*   Every entity has two keys: a **public encryption key** that is known to *everyone*, and a **private decryption key** that is known *only* to the owner.

```text
* Visual Aid: Public Key Cryptography *

              Bob's Public Key (K_B+)             Bob's Private Key (K_B-)
                       |                                  |
                       v                                  v
[Plaintext, m] -> [Encryption Algorithm] =========> [Decryption Algorithm] -> [Plaintext, m]
                                             |
                                    [Ciphertext K_B+(m)]
```
Anyone can encrypt a message using Bob's public key, but *only* Bob can decrypt it using his private key.

***

### 2.6 RSA in practice: session keys
While RSA (a public key algorithm) is incredibly secure, the mathematics involved (heavy exponentiation) are computationally intensive and slow. AES is orders of magnitude faster.
*   In practice, systems use a hybrid approach.
*   Bob and Alice use slow public key cryptography merely to securely exchange a symmetric **session key** (`K_S`).
*   Once both parties have `K_S`, they switch to symmetric key cryptography (like AES) for the remainder of the high-speed data transfer.

***

## 3.0 Authentication, message integrity

### 3.1 Authentication
**Authentication** is the process where Bob wants Alice to "prove" her identity to him over a network where he cannot physically "see" her.
*   **Protocol ap1.0**: Alice simply sends a message saying "I am Alice."
*   **Failure scenario**: Trudy can easily generate a message saying "I am Alice." There is no proof.

```text
* Visual Aid: Protocol ap1.0 Failure *

 Alice -----> "I am Alice" -----> Bob

 Trudy -----> "I am Alice" -----> Bob   (Bob cannot tell the difference)
```

***

### 3.2 Authentication: another try
*   **Protocol ap2.0**: Alice says "I am Alice" inside an IP packet that contains her legitimate source IP address.
*   **Failure scenario**: Trudy can perform **IP spoofing**, creating a forged packet that artificially places Alice's IP address in the source field.

```text
* Visual Aid: Protocol ap2.0 Failure *

 Trudy -----> [ Src IP: Alice's IP | "I am Alice" ] -----> Bob
 (Trudy successfully spoofs Alice's address)
```

***

### 3.3 Authentication: a third try
*   **Protocol ap3.0**: Alice sends her secret password over the network to prove her identity.
*   **Failure scenario**: A **playback attack**. Trudy doesn't need to know the password; she simply eavesdrops, records Alice's packet, and plays it back to Bob later to gain access.

```text
* Visual Aid: Protocol ap3.0 Playback Attack *

 Alice -----> [ IP: Alice | Password: "XYZ" ] -----> Bob
      \
       \---> (Trudy records this packet)

 [Later...]
 Trudy -----> [ IP: Alice | Password: "XYZ" ] -----> Bob 
 (Bob grants access thinking it is Alice)
```

***

### 3.4 Authentication: a modified third try
*   **Protocol ap3.0 (Modified)**: Realizing plaintext passwords are bad, Alice sends her *encrypted* secret password.
*   **Failure scenario**: A **playback attack still works**. Trudy simply records the encrypted string and replays it. Bob receives the valid encrypted string and authenticates Trudy.

```text
* Visual Aid: Modified ap3.0 Playback Attack *

 Alice -----> [ IP: Alice | Encrypted Password ] -----> Bob
      \
       \---> (Trudy records encrypted packet)

 [Later...]
 Trudy -----> [ IP: Alice | Encrypted Password ] -----> Bob 
 (Bob grants access because the encrypted string is valid)
```

***

### 3.5 Authentication: a fourth try
*   **Goal**: To defeat the playback attack, we must make sure every authentication attempt is mathematically unique.
*   **Solution**: Introduce a **nonce** (`R`), which is a random number used only *once-in-a-lifetime*.
*   **Protocol ap4.0**: Bob challenges Alice by sending her a nonce `R`. Alice must use a shared secret key to encrypt `R` and send it back. Because Trudy doesn't know the key, she cannot encrypt a new nonce.

```text
* Visual Aid: Protocol ap4.0 Nonce Exchange *

 Alice                                                 Bob
   <------------------ [ Nonce: R ] --------------------
   --- [ Encrypt(R) with Shared Key K_A-B ] ----------->
```

***

### 3.6 Authentication: ap5.0 (and its flaw)
*   **Protocol ap5.0**: Can we use a nonce with public key cryptography? Yes, Bob sends nonce `R`, and Alice encrypts it with her private key (`K_A-`). Bob decrypts it with Alice's public key (`K_A+`).
*   **Failure scenario**: The **man in the middle attack**. Trudy intercepts the initial public key exchange. She gives Bob her own public key, pretending it's Alice's. She gives Alice her own public key, pretending it's Bob's.
*   **Reason**: This happens because the protocol lacks a "root of trust." Bob has no way to verify that the public key he received actually belongs to Alice.

```text
* Visual Aid: Man in the Middle Attack *

Alice                      Trudy (MITM)                       Bob
      -- Send Pub Key -->  (Intercepts)
                           -- Sends Trudy's Pub Key to Bob --> (Bob thinks it's Alice's)
      <-- Send Pub Key --  (Intercepts)
<-- Sends Trudy's Pub Key-
```

***

### 3.7 Digital signatures
A digital signature is a cryptographic technique analogous to a hand-written signature, but much stronger.
*   The sender (owner/creator) digitally signs a document by encrypting it with their **private key**.
*   **Verifiable, nonforgeable**: The recipient can prove that *only* the specific sender could have signed it, because it successfully decrypts with the sender's public key.
*   **Non-repudiation**: Because only Bob has Bob's private key, Alice can take the signed document to a court of law to prove Bob signed it; Bob cannot deny it.

```text
* Visual Aid: Creating a Digital Signature *

 [Message, m] -----> [ Encrypt with Bob's Private Key, K_B- ] -----> [Signed Message, K_B-(m)]
```

***

### 3.8 Message digests
Public-key encryption is computationally expensive, making it highly inefficient to encrypt (sign) long messages. 
*   **Goal**: Create a fixed-length, easy-to-compute digital "fingerprint" of the message.
*   We apply a Hash function `H` to message `m` to get a digest `H(m)`.
*   **Hash function properties**: It is a many-to-1 function, produces a fixed-size digest, and it is computationally infeasible to invert (you cannot guess the message from the hash).

```text
* Visual Aid: Hash Function Process *

 [Large Message, m] -----> [ Hash Function, H ] -----> [Fixed Digest, H(m)]
```

***

### 3.9 Digital signature = signed message digest
Instead of signing the whole message, we sign the hash.
*   **Sender (Bob)**: Hashes the large message, encrypts *only the hash* with his private key, and sends both the plaintext message and the signed hash.
*   **Receiver (Alice)**: Decrypts the signed hash using Bob's public key. She then runs her own hash on the plaintext message. If her hash matches the decrypted hash, the message is authentic and unaltered.

```text
* Visual Aid: Signing and Verifying a Digest *

SENDER (BOB):
[Message] -> (Hash) -> [H(m)] -> (Encrypt with Private Key K_B-) -> [Signed Hash]
   |                                                                       |
   +---------------------------------> (Transmitted Together) <------------+

RECEIVER (ALICE):
[Received Message] -> (Hash) -> [Local H(m)] 
                                      |
                                      v
                             [ Compare for Equality ]
                                      ^
                                      |
[Signed Hash] -> (Decrypt with Public Key K_B+) -> [Decrypted H(m)]
```

***

### 3.10 Hash function algorithms
Common algorithms used to generate these digests include:
*   **MD5**: Creates a 128-bit message digest in a 4-step process (widely used historically, though vulnerable to collision attacks now).
*   **SHA-1**: Creates a 160-bit message digest and is a US standard. More secure than MD5, though newer standards like SHA-256 are modern best practices.

***

### 3.11 Authentication: ap5.0 – let’s fix it!!
Recall the problem with ap5.0: the Man-in-the-Middle attack works because Bob and Alice have no way to verify that the public keys they receive actually belong to each other. They lack a root of trust.

```text
* Visual Aid: The Flaw *
(Duplicate of MITM problem - Trudy intercepts public keys because there is no trusted third party to verify them.)
```

***

### 3.12 Public key Certification Authorities (CA)
To fix the lack of trust, networks use a **Certification Authority (CA)**.
*   A CA is a trusted third party that binds a public key to a specific entity (like a person, website, or router).
*   An entity provides "proof of identity" to the CA. The CA then creates a **certificate**, which contains the entity's public key, digitally signed by the *CA's private key*.
*   When Alice wants Bob's public key, she gets Bob's certificate and verifies it using the *CA's public key* (which is pre-installed on her machine, acting as the root of trust).

```text
* Visual Aid: Certification Authority Process *

1. CA signs Bob's Info:
[Bob's Identity + Bob's Public Key] -> (Encrypt with CA Private Key) -> [Bob's Certificate]

2. Alice verifies:
[Bob's Certificate] -> (Decrypt with CA Public Key) -> [Verified Bob's Public Key]
```

***

## 4.0 Securing TCP connections: TLS

### 4.1 Transport-layer security (TLS)
**TLS** is a widely deployed protocol that sits above the transport layer (most notably used by HTTPS on port 443).
*   It provides a complete security package:
    *   **Confidentiality**: via symmetric encryption.
    *   **Integrity**: via cryptographic hashing (MAC).
    *   **Authentication**: via public key cryptography and CAs.
*   **History**: It replaces the deprecated Secure Sockets Layer (SSL). TLS 1.3 is the current standard.
*   It provides a simple API that *any* application can use to secure a data stream.

```text
* Visual Aid: Protocol Stack *

   +--------------------------+
   |       Application        |
   | (e.g. HTTP/2, HTTP 1.0)  |
   +--------------------------+
   |          TLS             |
   +--------------------------+
   |          TCP             |
   +--------------------------+
   |          IP              |
   +--------------------------+
```

***

### 4.2 Transport-layer security: what’s needed?
To understand TLS, we can build a simplified toy version called `t-tls`. A full TLS protocol requires four phases:
1.  **Handshake**: Use certificates to authenticate and safely exchange a shared secret.
2.  **Key Derivation**: Expand the shared secret into multiple specific keys.
3.  **Data Transfer**: Stream data securely as a series of encrypted records.
4.  **Connection Closure**: Securely terminate the connection to prevent attackers from tampering with the stream ending.

***

### 4.3 t-tls: initial handshake
In our toy protocol:
*   Bob establishes a standard TCP connection with Alice.
*   Bob verifies Alice's identity using her CA-signed certificate.
*   Bob generates and sends Alice a Master Secret (MS), securely encrypted with her public key.
*   **Potential Issue**: This process is chatty. It takes 3 Round Trip Times (RTT) before the client can start receiving actual data (TCP handshake + TLS hello + Certificate/Secret exchange).

```text
* Visual Aid: Initial Handshake *

Client (Bob)                                     Server (Alice)
   | ------ TCP SYN ----------------------------------> |
   | <----- TCP SYNACK -------------------------------- |
   | ------ TCP ACK & t-tls hello --------------------> |
   | <----- Public Key Certificate -------------------- |
   | ------ Encrypted Master Secret: K_A+(MS) --------> |
```

***

### 4.4 t-tls: cryptographic keys
It is a bad cryptographic practice to use the exact same key for different functions (like encryption and message integrity) or for different directions of traffic.
*   TLS uses a Key Derivation Function (KDF) to turn the Master Secret into four distinct keys:
    *   `K_c`: Encryption key (Client to Server)
    *   `M_c`: MAC (Integrity) key (Client to Server)
    *   `K_s`: Encryption key (Server to Client)
    *   `M_s`: MAC (Integrity) key (Server to Client)

***

### 4.5 t-tls: encrypting data
TCP is a *byte stream*. If we just encrypted the stream and put a MAC at the very end, we wouldn't know if data was altered until the entire connection closed.
*   **Solution**: Break the stream into discrete "records."
*   Each record carries its own MAC, so the receiver can act securely on each chunk as it arrives.
*   **Attacks**: An attacker could *re-order* TCP segments or *replay* them.
*   **Fix**: TLS uses sequence numbers incorporated into the MAC, and nonces, to ensure data is in order and fresh.

```text
* Visual Aid: t-tls Data Record *

  +-------------------------------------------------+
  |  Encrypted with K_c:                            |
  |  [ Length |       Data       |       MAC      ] |
  +-------------------------------------------------+
```

***

### 4.6 t-tls: connection close
Attackers could execute a **truncation attack**: forging a TCP connection close segment so the receiver thinks the file finished downloading, when in reality, the end was cut off.
*   **Solution**: Introduce a `type` field into the record (e.g., type `0` for data, type `1` for close).
*   The MAC is computed over the data, the type, and the sequence number, meaning a forged closure cannot be authenticated.

```text
* Visual Aid: Updated t-tls Record Format *

  +---------------------------------------------------------+
  |  Encrypted with K_c:                                    |
  |  [ Length |   Type   |       Data       |       MAC   ] |
  +---------------------------------------------------------+
```

***

## 5.0 Network layer security: IPsec

### 5.1 IP Sec
While TLS operates at the transport layer, **IPsec** operates lower down, providing datagram-level encryption, authentication, and integrity. This protects both user data and network control traffic (like BGP and DNS).
*   **Transport mode**: Only the *payload* of the IP datagram is encrypted. The IP header remains plaintext.
*   **Tunnel mode**: The *entire* original datagram (header and payload) is encrypted and then encapsulated into a brand new IP datagram with a new header.

```text
* Visual Aid: Transport vs Tunnel Mode *

Transport Mode Payload:  [ IP Header (Plain) | Encrypted Payload ]
Tunnel Mode Payload:     [ New IP Header (Plain) | Encrypted [ Old IP Header | Payload ] ]
```

***

### 5.2 Two IPsec protocols
IPsec primarily utilizes two different protocols to achieve security:
*   **Authentication Header (AH)**: Provides source authentication and data integrity, but *no confidentiality* (the data is not encrypted).
*   **Encapsulation Security Protocol (ESP)**: Provides authentication, data integrity, *and confidentiality*. Because it encrypts the data, ESP is far more widely used than AH.

***

### 5.3 Security associations (SAs)
IP is natively connectionless, but IPsec requires stateful knowledge (keys, algorithms). Before data is sent, a directional **Security Association (SA)** is established.
*   A router (R1) stores SA state information, which includes:
    *   A 32-bit Security Parameter Index (SPI) to identify the connection.
    *   Origin and destination interfaces.
    *   Encryption type/keys and Integrity type/keys.

```text
* Visual Aid: Security Association *

     Router R1 (200.168.1.100) <===== [ Security Association (SA) State ] =====> Router R2 (193.68.2.23)
```

***

### 5.4 IPsec datagram
In Tunnel Mode using ESP, the IPsec datagram contains several components to guarantee security:
*   **ESP Trailer**: Provides padding necessary for block ciphers like AES.
*   **ESP Header**: Contains the SPI (so the receiver knows which SA to use) and a sequence number (to thwart replay attacks).
*   **ESP Auth Field**: A MAC created with the shared secret key to verify integrity.

```text
* Visual Aid: IPsec Datagram (Tunnel Mode with ESP) *

                           <-------- Encrypted -------->
                           <-------------- Authenticated -------------->
[ New IP Header | ESP Header | Original IP Header | Payload | ESP Trailer ] [ ESP Auth ]
                   (SPI, Seq)                                    (Padding)      (MAC)
```

***

### 5.5 ESP tunnel mode: actions
When Router 1 (R1) processes a packet in ESP tunnel mode, it performs the following steps:
1.  Appends an ESP trailer to the original datagram.
2.  Encrypts this combined result.
3.  Appends the ESP header to the front of the encrypted chunk.
4.  Creates an authentication MAC over the whole sequence.
5.  Creates a completely new IP header to route the encapsulated packet to the tunnel endpoint.

```text
* Visual Aid: Encapsulation Steps at R1 *

 Original:                     [ Orig IP Hdr | Payload ]
 1. Add Trailer:               [ Orig IP Hdr | Payload | Trailer ]
 2. Encrypt:                   *ENCRYPTED DATA*
 3. Add ESP Hdr:               [ ESP Hdr | *ENCRYPTED DATA* ]
 4. Add MAC:                   [ ESP Hdr | *ENCRYPTED DATA* ] + [ Auth MAC ]
 5. Add New IP Hdr:            [ New IP Hdr | ESP Hdr | *ENCRYPTED DATA* | Auth MAC ]
```

***

### 5.6 IPsec sequence numbers
To prevent replay attacks at the network layer:
*   The sender initializes a sequence number counter to 0 and increments it for every datagram sent on that SA.
*   The destination checks for duplicate sequence numbers.
*   Since keeping track of every single packet ever received is impossible, the receiver uses a "sliding window" to check for recent duplicates.

***

### 5.7 IPsec security databases
A router manages IPsec operations using two key databases:
*   **Security Policy Database (SPD)**: Determines *what* to do. It looks at the source/destination IPs and protocols to decide if a datagram *should* be protected by IPsec.
*   **Security Assoc. Database (SAD)**: Determines *how* to do it. If the SPD says IPsec is required, the router looks in the SAD to find the corresponding SA state (the actual keys and algorithms to apply).

***

### 5.8 Summary: IPsec services
By deploying IPsec (specifically ESP with sequence numbers), a network protects against:
*   Eavesdroppers seeing original contents (via encryption).
*   Attackers flipping bits in transit (via MAC/Integrity).
*   Attackers masquerading as trusted routers (via Authentication).
*   Attackers executing playback attacks (via Sequence numbers).

***

## 6.0 Operational security: firewalls and IDS

### 6.1 Firewalls
A **firewall** is a security device or software that isolates an organization's internal, trusted network from the untrusted, larger Internet. It allows specific packets to pass while blocking others.
*   **Why use them**: To prevent DoS attacks (like SYN flooding), prevent unauthorized modification of internal data, and ensure only authenticated users access the network.
*   **Three types**: Stateless packet filters, stateful packet filters, and application gateways.

```text
* Visual Aid: Firewall Concept *

  [ Trusted Internal Network ] <======> [ ||||FIREWALL|||| ] <======> [ Untrusted Public Internet ]
          "Good Guys"                                                    "Bad Guys"
```

***

### 6.2 Stateless packet filtering
Stateless filters evaluate traffic strictly on a *packet-by-packet* basis, with no memory of past packets.
*   Rules are based on checking: Source/destination IPs, TCP/UDP ports, ICMP types, and TCP bit flags (SYN, ACK).

```text
* Visual Aid: Router as a Firewall *

  (Internet) ---> [ Router/Firewall ] ---> (Internal Network)
                      |
        "Does this specific packet match an ALLOW rule?"
```

**Example Policy Table:**
| Policy | Firewall Setting |
| :--- | :--- |
| No outside web access | Drop all outgoing packets to any IP, port 80 |
| Prevent smurf DoS attacks | Drop all ICMP packets going to broadcast IPs |

***

### 6.3 Stateful packet filtering
Stateless filters are "heavy-handed" and can be tricked (e.g., they might admit a packet with an `ACK` bit set even if no connection was ever initiated).
*   A **stateful packet filter** solves this by tracking the status of every ongoing TCP connection.
*   It monitors `SYN` (setup) and `FIN` (teardown) packets to build a connection state table. If a packet arrives claiming to belong to a connection that isn't in the state table, it is dropped.
*   It also automatically times out inactive connections.

```text
* Visual Aid: Stateful ACL Addition *

| Action | Src IP | Dest IP | Protocol | Src Port | Dest Port | Flag | Check Connection? |
|--------|--------|---------|----------|----------|-----------|------|-------------------|
| Allow  | In     | Out     | TCP      | >1023    | 80        | ANY  |                   |
| Allow  | Out    | In      | TCP      | 80       | >1023     | ACK  |       [ X ]       | 
  *(The 'X' means the firewall checks its internal state table before allowing)*
```

***

### 6.4 Limitations of firewalls, gateways
Firewalls are not a silver bullet:
*   **IP Spoofing**: Because firewalls rely heavily on reading IP headers, a router cannot definitively know if a packet *really* originated from the IP address claimed in the source field.
*   **Usability Tradeoff**: There is always a friction tradeoff between maintaining a high degree of communication capability and a high level of security.

***

### 6.5 Intrusion detection systems
Packet filtering (firewalls) only looks at TCP/IP headers, not the actual data payload. Furthermore, they don't correlate activities across multiple different connections.
*   **IDS (Intrusion Detection System)**: Monitors the network at a deeper level.
*   **Deep packet inspection**: The IDS looks at the actual contents of the payload, scanning for known virus signatures or attack strings.
*   **Correlation**: It examines patterns across multiple packets to detect activities like port scanning, network mapping, or distributed DoS attacks.

***

### 6.6 Network Security (summary)
To achieve true network security, a layered approach is required:
*   **Basic Techniques**: Cryptography (Symmetric and Public key), Message Integrity (Hashing/MACs), and Endpoint Authentication (Nonces, Certificates).
*   **Protocol Implementations**: These techniques are applied across various layers: Application (Secure Email), Transport (TLS), Network (IPsec), and Link (802.11 Wi-Fi, 4G/5G).
*   **Operational Security**: The physical and logical boundaries are monitored and protected by firewalls and IDSs.

***

### **Key Terms for Definition:**
*   **Confidentiality**: A security principle ensuring that only the sender and intended receiver can read and understand the contents of a message.
*   **Authentication**: The cryptographic process of definitively verifying the identity of a communicating party over a network.
*   **Message Integrity**: A security guarantee that a message has not been altered, tampered with, or corrupted during transit.
*   **Access and Availability**: The principle ensuring that network services remain operational and accessible to authorized users (thwarting DoS attacks).
*   **Nonce**: A random number used only "once-in-a-lifetime" in authentication protocols to prevent playback attacks.
*   **Playback Attack**: A network attack where a malicious user records a legitimate encrypted packet or password and replays it later to gain unauthorized access.
*   **Man in the middle attack**: An attack where an intruder secretly intercepts and relays communications between two parties, substituting their own public keys to decrypt and alter the traffic.
*   **Verifiable, nonforgeable (signatures)**: A property of digital signatures ensuring that the recipient can mathematically prove that only the stated owner of the private key could have created the signature.
*   **Non-repudiation**: The cryptographic assurance that the creator of a signed message cannot later deny having sent or signed it.
*   **Certification Authority (CA)**: A trusted third-party entity that verifies identities and issues digital certificates binding an identity to a public key.
*   **Security Association (SA)**: The connection-oriented, directional state data (including keys and SPI) maintained by routers before sending IPsec protected traffic.
*   **Firewall**: A network security system that monitors and controls incoming and outgoing traffic based on predetermined security rules, isolating an internal network from the Internet.
*   **IP Spoofing**: The creation of IP packets with a false source IP address, intended to impersonate another computing system.
*   **Intrusion Detection System (IDS)**: A security application that monitors network traffic for suspicious activity, policy violations, and known attack signatures.
*   **Deep Packet Inspection**: A form of network filtering (often used by IDS) that examines the actual data part (payload) of a packet rather than just the header information.
