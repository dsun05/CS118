## 1.0 Introduction

### 1.1 CS118 Discussion 1A, Week 6
Welcome to Week 6, Discussion 1A for CS118. This session is led by your teaching assistant, Ziyue Dang, and focuses heavily on the practical implementation details required for your upcoming project assignments.

***

### 1.2 Today's Agenda
In this session, we will be covering two primary areas:
*   **Project 2 topics:** We will dive deep into the specific cryptographic protocols, state machines, and data structures you need to implement for Project 2.
*   **Office hours format:** The remainder of the discussion will operate similarly to office hours, allowing for specific troubleshooting and questions regarding your individual project codebases.

## 2.0 Security Goals

### 2.1 The CIA Triad
Before diving into the implementation details, it is essential to understand the fundamental objectives of network security, commonly referred to as the **CIA Triad**. Your project protocol is designed to achieve these three specific goals:

*   **Authenticity:** This answers the question, *"Are you really talking to who you think you're talking to?"* In network communications, malicious actors can easily spoof IP addresses. Authenticity ensures that the server you are communicating with is genuinely the entity it claims to be, typically verified via digital certificates.
*   **Privacy (Confidentiality):** This answers the question, *"Can anyone else read our messages?"* When data traverses the public internet, it passes through dozens of intermediary routers. **Privacy** ensures that even if a third party intercepts the packets, they cannot read the underlying plaintext. This is achieved through robust encryption.
*   **Integrity:** This answers the question, *"Has anyone tampered with our messages?"* Even if a message is encrypted, an attacker might try to flip bits in the ciphertext to corrupt the data or alter its meaning. **Integrity** checks ensure that if a single byte is modified in transit, the receiver will detect it and reject the payload.

## 3.0 TLV Encoding

### 3.1 The Data Format & Nested TLVs
To transmit data over the network in a structured way, our protocol uses **TLV Encoding** (Type-Length-Value). This is a highly extensible data framing technique widely used in networking.

```text
+---------------+---------------------+-----------------------------------+
| Type (1 byte) | Length (1-3 bytes)  | Value (Length bytes)              |
+---------------+---------------------+-----------------------------------+
| e.g., 0x82    | e.g., 0x20          | e.g., [32 random bytes...]        |
| (NONCE)       | (32 bytes decimal)  |                                   |
+---------------+---------------------+-----------------------------------+
```

*   **Structure Examples:** As seen in the diagram above, a **Nonce** is represented by a specific type identifier (`0x82`). Its length is `0x20` (32 bytes in decimal), followed directly by the 32 bytes of random data.
*   **Length Formatting Rules:** To optimize bandwidth, the protocol uses a variable-length field for the "Length" component:
    *   If the payload length is ≤ 252 bytes: The length is represented by exactly 1 byte.
    *   If the payload length is > 252 bytes: The length field starts with a special marker `0xFD`, followed by a 2-byte big-endian representation of the actual length.
    *   **Purpose:** This design optimizes space for small messages (which are frequent) while still supporting large messages (like file transfers or large certificates) when needed.
*   **Nested TLVs:** One of the most powerful features of TLV encoding is nesting. Certain types act as "containers" (`CLIENT_HELLO`, `SERVER_HELLO`, `CERTIFICATE`, `DATA`). These containers hold other TLV structures inside their "Value" payload.
    *   *Example:* The `CLIENT_HELLO` TLV's value is actually a sequence of three smaller TLVs: `VERSION_TAG`, `NONCE`, and `PUBLIC_KEY`.
*   **Implementation Notes (Important!):** 
    *   Do **NOT** attempt to write the TLV parsing logic manually. The course staff has provided `serialize_tlv()` and `deserialize_tlv()` functions. Rely on these to prevent subtle memory bugs.
    *   You must, however, thoroughly understand how the structure nests together to construct the right buffers and to debug effectively.
    *   Make heavy use of the `print_tlv_bytes()` function. Printing out the hex representation of your network buffers is the fastest way to verify that your nested TLVs are formatted correctly before dispatching them over a socket.

## 4.0 Protocol Steps & States

### 4.1 Step 1: Client Hello
The connection begins with the client initiating the handshake. 
*   **State:** The client program begins in the `CLIENT_CLIENT_HELLO_SEND` state.

```text
CLIENT_HELLO {
    VERSION: 0x01
    NONCE: 32 random bytes (client_nonce)
    PUBLIC_KEY: Client's ephemeral ECDH public key
}
```

*   **Actions:** 
    *   First, generate a mathematically random 32-byte value using `generate_nonce()`. This **Nonce** is strictly single-use and will protect the session against replay attacks.
    *   Next, generate an **ephemeral** keypair. This is a temporary cryptographic key used *only* for this specific session. You will call `generate_private_key()` followed by `derive_public_key()`.
    *   Finally, package the protocol version, the nonce, and the public key into nested TLVs inside the `CLIENT_HELLO` container and send it to the server.

***

### 4.2 Client States Overview
The client operates as a state machine. Understanding the transition of states is crucial for organizing your code:
*   `C_CLIENT_HELLO_SEND`: The initial state where the client constructs and dispatches the `CLIENT_HELLO` message.
*   `C_SERVER_HELLO_AWAIT`: The client blocks and waits for the server's response. Upon receiving the `SERVER_HELLO`:
    *   It verifies the Handshake Signature and the Certificate.
    *   If verification passes, it derives the shared secret.
*   `DATA_STATES`: Once the handshake is complete, the client enters the data transmission loop, securely sending and receiving encrypted data.
*   *Related functions highlighted:* To manage these states, you will utilize provided helper functions such as `generate_nonce()`, `generate_private_key()`, `derive_public_key()`, `load_ca_public_key()`, `verify()`, etc.

***

### 4.3 Server States Overview
Similarly, the server follows a strict state progression:
*   `S_CLIENT_HELLO_AWAIT`: The server sits idle, listening for an incoming `CLIENT_HELLO` message.
*   `S_SERVER_HELLO_SEND`: Once the client's message is received, the server transitions here to:
    *   Construct and send the `SERVER_HELLO`.
    *   Derive the shared cryptographic secret on its end.
*   `DATA_STATES`: Finally, the server enters the synchronous loop of sending and receiving protected data.

***

### 4.4 Step 2: Server Hello
When the server receives the `CLIENT_HELLO`, it must respond appropriately to establish trust and cryptographic synchronization.
*   **State:** The server is in the `SERVER_SERVER_HELLO_SEND` state.

```text
SERVER_HELLO {
    NONCE: 32 random bytes (server_nonce)
    CERTIFICATE: Full certificate (from server_cert.bin)
    PUBLIC_KEY: Server's ephemeral ECDH public key
    HANDSHAKE_SIGNATURE: Signs entire handshake transcript
}
```

*   **Actions:**
    *   The server parses the incoming `CLIENT_HELLO` TLV.
    *   It extracts and loads the peer's (client's) public key.
    *   The server generates its own **ephemeral keypair** specifically for this session's key exchange.
    *   **Critical:** The server must mathematically sign the entire handshake transcript to prove its identity. It does this using its long-term **IDENTITY key** (loaded from `server_key.bin`).
*   **Key switching dance (important!):**
    *   A common pitfall is misunderstanding that the server utilizes **TWO** distinct private keys during this step:
        1.  **Ephemeral key:** Generated on the fly, used purely for the **ECDH** key exchange. It is temporary and provides forward secrecy.
        2.  **Identity key:** The permanent, long-term private key associated with the server's certificate. It is used *only* for signing the handshake to prove identity.
    *   To implement this, you must temporarily save your ephemeral key, load the identity key into the cryptographic engine to compute the signature, and then immediately restore the ephemeral key for the remainder of the session.
    *   You will manage this delicate process using the `get_private_key()` and `set_private_key()` functions.

***

### 4.5 Step 3: Client Verification
Once the client receives the `SERVER_HELLO`, it must rigorously validate the server's claims before proceeding.
*   **State:** The client transitions out of `CLIENT_SERVER_HELLO_AWAIT`.
*   **Verification Sequence:** Failure at any of these steps mandates an immediate connection termination.
    *   **3a. Certificate Signature Validation:** The client verifies the certificate against the Certificate Authority (CA) public key. If invalid, the client triggers `exit(1)` ("Bad Certificate"). This establishes the chain of trust—the CA officially vouches for this server.
    *   **3b. Certificate Lifetime Check:** The client checks the timestamps on the certificate. If it is expired or its "valid from" date is in the future, trigger `exit(1)` ("Bad Certificate").
    *   **3c. Hostname Verification:** The client checks if the domain name on the certificate matches the server it intended to connect to. If there is a mismatch, trigger `exit(2)` ("Bad Identity"). This prevents an attacker from stealing a valid certificate for `example.com` and presenting it while hosting `yourbank.com`.
    *   **3d. Handshake Signature Verification:** The client verifies the cryptographic signature attached to the `SERVER_HELLO`. If invalid, trigger `exit(3)` ("Bad Handshake Signature"). This is the definitive proof that the server actually possesses the private key mathematically linked to the provided certificate.

***

### 4.6 Verification Success 
After the client successfully navigates the verification gauntlet, the secure channel can be finalized.
*   The client must load the server's **EPHEMERAL** public key (ensure you do not accidentally load the long-term identity key found in the certificate).
*   Execute `derive_secret()` to compute the master shared secret.
*   Derive the specific session keys (for encryption and MAC) by calling `derive_keys(salt, 64)`.
    *   *Note:* The *Salt* parameter used in this derivation adds crucial entropy and must be constructed by concatenating the nonces: `client_nonce` || `server_nonce`.

## 5.0 Cryptographic Primitives Deep Dive

### 5.1 Elliptic Curve Diffie-Hellman (ECDH)
*   **Problem:** How do two parties establish a shared, secret value over a channel where an attacker is observing every packet?
*   **Mechanism:**
    *   ECDH relies on the mathematical properties of elliptic curves over finite fields. If Alice has private key `a` and computes public key `A = a·G` (where G is a standard base point), and Bob does the same (`B = b·G`), they can safely exchange `A` and `B`.
    *   **Shared secret computation:** Alice computes `a·B`. Bob computes `b·A`. Because of commutative properties, `a·(b·G) = b·(a·G) = a·b·G`. They arrive at the exact same shared secret.
    *   **Security:** An eavesdropper sees `A` and `B`, but cannot deduce the secret because deriving `a` or `b` from the public keys involves solving the Elliptic Curve Discrete Logarithm Problem (**ECDLP**), which is computationally infeasible for modern computers.

***

### 5.2 HKDF (HMAC-based Key Derivation Function)
*   **Problem:** ECDH gives us exactly ONE 32-byte shared secret. However, best security practices dictate we use different keys for different operations—we need TWO keys: an `enc_key` (for encryption) and a `mac_key` (for integrity).
*   **Solution:** We pass the raw shared secret through **HKDF-SHA256**.
    *   **Input:** The raw Secret (from ECDH), a Salt (to add randomness, preventing pre-computation attacks), and Info (a string like "enc" or "mac" to provide domain separation, ensuring the two derived keys are totally different).
    *   **Output:** HKDF securely expands the material into two independent 32-byte keys suitable for AES-256 and HMAC.

***

### 5.3 Digital Signatures (ECDSA)
*   **Problem:** How do we definitively prove the origin of a message and ensure it hasn't been spoofed?
*   **Asymmetric Property:** We use Elliptic Curve Digital Signature Algorithm (**ECDSA**). A private key is used to mathematically **SIGN** a message, while the corresponding public key can **VERIFY** that signature. Signatures cannot be forged without the private key.
*   **Two protocol signatures:** Your project utilizes this concept in two distinct places:
    1.  **Certificate signature:** Generated by the CA to sign the server's identity. You verify this using the CA's well-known public key.
    2.  **Handshake signature:** Generated by the Server to prove possession of the identity key. You verify this using the server's public key (extracted from the certificate).

***

### 5.4 AES-256-CBC Encryption
*   **Problem:** We need **Confidentiality**; we must obscure our application data.
*   **AES-256:** This is the Advanced Encryption Standard, using a 256-bit (32-byte) symmetric key (the same key encrypts and decrypts). It is the global industry standard.
*   **CBC Mode (Cipher Block Chaining):**
    *   AES natively encrypts data in fixed 16-byte blocks. CBC mode chains these blocks together (the ciphertext of block 1 is XORed with the plaintext of block 2, etc.).
    *   This chaining requires a 16-byte **Initialization Vector (IV)** to start the process.
    *   The **IV** must be mathematically random and entirely unique for every single message.
    *   **Why?** Without a unique IV, sending the exact same plaintext message twice would yield the exact same ciphertext. An attacker could observe this and deduce patterns in the communication. The IV prevents identical plaintexts from yielding identical ciphertexts.

***

### 5.5 HMAC-SHA256 (Message Authentication Code)
*   **Problem:** We need **Integrity**; we must prevent an attacker from blindly flipping bits in our ciphertext to cause chaotic behavior upon decryption.
*   **Mechanism:** We use a cryptographic hash function (SHA-256) deeply intertwined with a secret key (the `mac_key`).
*   **Output:** This algorithm produces a 32-byte "tag" or "digest" that acts as an unforgeable cryptographic checksum. Only someone possessing the `mac_key` can generate a valid tag.

## 6.0 Data Transmission

### 6.1 Sending Data
Once the handshake is verified, application data is structured into a `DATA` TLV.

```text
DATA {
    IV: 16 bytes
    MAC: 32 bytes
    CIPHERTEXT: variable length
}
```

*   **Process Pipeline:** When your program needs to send a message, it follows this strict pipeline:
    1.  **Read plaintext:** Retrieve input data (e.g., from `stdin` via `input_io()`).
    2.  **Encrypt:** Call `encrypt_data(iv, ciphertext, plaintext, len)`. This function generates the random IV and computes the ciphertext.
    3.  **Serialize TLVs:** Package the IV and Ciphertext into local TLV buffers.
    4.  **Compute MAC:** Calculate the HMAC over the serialized buffer to ensure everything is tamper-proof.
    5.  **Build final DATA TLV:** Combine the IV, MAC, and Ciphertext components into the final parent `DATA` container.
    6.  **Send over network:** Dispatch the buffer through the socket.

***

### 6.2 Receiving Data
Receiving data requires extreme care, as malicious actors can send carefully crafted, malformed packets to exploit cryptographic engines.
*   **Critical Rule:** You must verify the MAC **BEFORE** you attempt to decrypt the data!
*   **Process Pipeline:**
    1.  **Parse DATA TLV:** Extract the IV, MAC, and Ciphertext sub-TLVs.
    2.  **Re-serialize locally:** Rebuild the IV and Ciphertext TLVs in memory to match exactly how the sender would have structured them before hashing.
    3.  **Compute HMAC:** Use your local `mac_key` to compute what the MAC *should* be based on the incoming data.
    4.  **Compare MACs:** Compare the received MAC against your locally computed MAC. This must be a **constant-time comparison** to prevent timing attacks.
    5.  **If invalid -> Exit immediately:** Trigger `exit(5)`. Do not process the data further.
    6.  **If valid -> Decrypt:** Call `decrypt_cipher()` and output the resulting plaintext.
*   **Why verify MAC first?**
    *   **Security:** Decrypting unauthenticated ciphertext opens you up to devastating vulnerabilities, most notably the **Padding Oracle Attack**, where an attacker uses cryptographic padding errors as a side-channel to gradually deduce the plaintext.
    *   **Robustness:** Cryptographic decryption is computationally expensive. Verifying the MAC first acts as a highly efficient filter, immediately rejecting garbage data or tampering before wasting CPU cycles.

## 7.0 Security Analysis

### 7.1 Protocol Coverage
Understanding the boundaries of your security protocol is just as important as implementing it.
*   **Attacks Prevented:**
    *   **Passive eavesdropping:** Thwarted by AES-256 encryption. Attackers cannot read the payload.
    *   **Message tampering:** Thwarted by HMAC verification. Any flipped bit results in a rejected packet.
    *   **Server impersonation:** Thwarted by Certificate chain validation. An attacker cannot fake being the trusted server.
    *   **Replay attacks:** Thwarted by the use of fresh, randomized nonces in every handshake. An old intercepted handshake will be rejected because the nonces won't match.
    *   **Man-in-the-middle (MITM):** Thwarted by strict Hostname + Signature validation. An attacker cannot intercept and seamlessly relay traffic without triggering certificate errors.
*   **Out of Scope (Not Prevented):**
    *   **Client authentication:** This protocol only requires the server to prove its identity to the client. The server does not verify who the client is.
    *   **Denial of Service (DoS):** An attacker can still spam the server with junk connections, consuming CPU and memory.
    *   **Traffic analysis:** Even though the data is encrypted, an eavesdropper can still observe packet sizes and timing, potentially inferring patterns (e.g., distinguishing between a VoIP call and a file download).
    *   **Entire connection replays:** Since there is no session resumption or ticketing system, an attacker could theoretically replay raw network captures, though the protocol's state machine will quickly reject them as malformed handshakes.

---

### **Key Terms for Formal Definition**
*   **Authenticity, Privacy, Integrity:** The three pillars of the CIA Triad in security, representing verification of identity, confidentiality of data, and protection against tampering, respectively.
*   **TLV Encoding:** Type-Length-Value data formatting structure, allowing flexible, nestable protocol design.
*   **Nonce:** A "Number used ONCE". A random, unique value generated per session to prevent replay attacks.
*   **Ephemeral Key:** A temporary cryptographic key pair generated dynamically and used only for a single session or key exchange (provides forward secrecy).
*   **Identity Key:** A long-term cryptographic private key firmly associated with a server's identity, primarily used for generating digital signatures.
*   **ECDH (Elliptic Curve Diffie-Hellman):** A cryptographic protocol allowing two parties to establish a shared secret over an insecure channel by utilizing the mathematics of elliptic curves.
*   **HKDF:** HMAC-based Key Derivation Function. An algorithm used to expand a single master secret into multiple independent keys (e.g., splitting a shared secret into separate encryption and MAC keys).
*   **ECDSA:** Elliptic Curve Digital Signature Algorithm. An asymmetric cryptographic algorithm used to generate unforgeable digital signatures for authentication.
*   **AES-256-CBC:** Advanced Encryption Standard utilizing a 256-bit key operating in Cipher Block Chaining mode for symmetric encryption.
*   **Initialization Vector (IV):** A randomly generated block of data required to initialize CBC encryption. It guarantees that identical plaintexts will encrypt to entirely different ciphertexts.
*   **HMAC:** Hash-based Message Authentication Code. A technique mapping a cryptographic hash (like SHA-256) with a secret key to verify data integrity and authenticity.
*   **Padding Oracle Attack:** A severe cryptographic vulnerability where an attacker exploits the side-channel behavior of a system (specifically, how it reacts to valid vs. invalid padding during decryption) to decrypt ciphertext without knowing the key. Avoided by always verifying the MAC before attempting decryption.
