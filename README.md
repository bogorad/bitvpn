### I. The Core Concept: A High-Stealth Tunneling System

The fundamental goal is to create a secure, private internet tunnel for a small, trusted group of users. Its primary design objective is not speed, but **stealth**. It is engineered to be virtually undetectable by Deep Packet Inspection (DPI) engines and other network surveillance tools that are designed to identify and block standard VPN protocols.

To achieve this, the system uses the ubiquitous **BitTorrent protocol as camouflage**. It does **not** use the public BitTorrent network to transfer data or improve speed. It merely mimics the behavior of a standard BitTorrent client at every stage, making its traffic blend in with the noise of global file-sharing activity.

---

### II. The Cast of Characters: System Roles

1.  **The Client Application:** Custom software run by each user on their device. It handles the entire process of finding the server, establishing the secret tunnel, and routing application traffic (e.g., from a web browser) through it.

2.  **The Exit Node Server:** A server you control (typically a Linux VPS) located in a jurisdiction of your choice. It runs the server-side software, which listens for connections, validates clients, and forwards their traffic to the public internet.

3.  **The Public BitTorrent DHT:** The massive, decentralized "phonebook" used by real BitTorrent clients. We leverage this public infrastructure as a resilient and untraceable mechanism for clients to learn the Exit Node's IP address.

---

### III. The Cryptographic Foundation: Keys and Secrets

The system's security is built on a foundation of five distinct cryptographic elements. You, the administrator, are responsible for generating and securely distributing the initial secrets.

#### A. Asymmetric Keypairs (For Authentication)

- **What:** A standard public/private keypair (e.g., ECDSA) is generated for the Exit Node and for each individual user.
- **Purpose:** **Authentication.** This proves identity. When a client connects, it uses its private key to sign a challenge, which the server verifies with the client's public key. This ensures that only authorized individuals can access the system.

#### B. The Permanent Secret Salt (The Master Secret)

- **What:** A single, long, high-entropy string of text (e.g., `a7d9...f4e2`). It is generated **once** and distributed securely to all users **once**.
- **Purpose:** This is the **permanent, static** root secret. It is never transmitted and never changes. It acts as the source material from which the system's daily dynamic secrets are derived.

#### C. The Daily `info_hash` (The Dynamic Public Meeting Point)

- **What:** A 20-byte code that is used to find the Exit Node on the public DHT. It **changes every day**.
- **Purpose:** **Discovery.** Both the client and server calculate this code independently using a standard formula:
  `info_hash = SHA1(Permanent_Secret_Salt + Current_UTC_Date_String)`
- The Exit Node announces itself to the DHT under this `info_hash`. Clients query the DHT for it. Because the `info_hash` is ephemeral and derived from a secret salt, it provides **location forward secrecy**.

#### D. The Daily Obfuscation Key (The Dynamic Internal Secret)

- **What:** A secret key used to encrypt the tunnel handshake itself. It also **changes every day**.
- **Purpose:** **Payload Protection.** To prevent DPI from recognizing the structure of the XTLS handshake, we encrypt it before wrapping it in a BitTorrent message. This key is derived independently:
  `obfuscation_key = SHA1(Permanent_Secret_Salt + Current_UTC_Date_String + "tunnel_key_v1")`
- This key is never transmitted; it's calculated by both sides to hide the handshake.

#### E. The Tunneling Protocol: XTLS (based on TLS 1.3)

- **What:** The actual secure tunnel that carries the user's internet traffic is built using XTLS, a modern, high-performance variant of TLS 1.3.
- **Purpose:** **Performance and Security.** This protocol is chosen for three key reasons:
  1.  **Speed:** It is extremely fast, as **modern CPUs include hardware acceleration** (e.g., AES-NI) for its underlying cryptographic operations, making encryption and decryption highly efficient.
  2.  **Modern Security:** It uses the latest, most secure cryptographic ciphers and has a faster, more secure handshake process than older protocols.
  3.  **Battle-Tested:** It is the same core technology that secures the vast majority of the modern web (HTTPS), meaning it is robust, well-understood, and heavily scrutinized.

---

### IV. The Complete Flow: A Step-by-Step Journey

This is the lifecycle of a connection, layering all our stealth strategies.

#### Phase 1: Dynamic Discovery

1.  A user starts the Client Application.
2.  The client calculates **today's `info_hash`** using the Permanent Salt and the current UTC date.
3.  It connects to the public BitTorrent DHT and asks for peers associated with that `info_hash`.
4.  The DHT network returns the IP address of your Exit Node.
5.  To handle the midnight UTC handover, if the client finds no peer, it automatically calculates **yesterday's `info_hash`** and queries again. The Exit Node announces on both hashes around midnight to ensure a smooth transition.

#### Phase 2: The Cover Story (Behavioral Mimicry)

1.  The client connects to the Exit Node's IP and performs a **perfectly standard BitTorrent handshake**.
2.  To establish a benign context, they engage in a **"bait data" exchange**.
    - The client sends an `interested` message; the server replies with `unchoke`.
    - The client requests a few pieces of a pre-shared, innocuous "bait file."
    - The server responds with the correct `piece` messages containing the bait data.
3.  **Result:** Any DPI engine has now classified this flow as a **legitimate BitTorrent file transfer**, defeating behavioral analysis.

#### Phase 3: The Secret Handshake (Tunnel Negotiation)

1.  With its cover established, the client begins the real handshake, disguised within the common **`ut_metadata` protocol**.
2.  It calculates **today's `obfuscation_key`**.
3.  It takes its XTLS `ClientHello` message and **encrypts it** with the `obfuscation_key`.
4.  It **wraps** this encrypted payload inside a `ut_metadata` message and sends it.
5.  The Exit Node receives the message, extracts the payload, decrypts it with the same `obfuscation_key`, and processes the `ClientHello`. It follows the identical encrypt-then-wrap process for its `ServerHello` response.
6.  After this exchange, a secure XTLS tunnel is established. This process defeats both **protocol anomaly detection** and **payload fingerprinting**.

#### Phase 4: The Secure Tunnel (Data Transfer)

1.  The XTLS tunnel is now active.
2.  All application traffic (e.g., an HTTPS request) is encrypted by the XTLS tunnel.
3.  The client wraps these encrypted XTLS packets inside standard BitTorrent **`piece` messages** and sends them to the Exit Node.
4.  The Exit Node unwraps the `piece` messages, decrypts the data via the XTLS tunnel, and forwards it to the public internet. This process looks like a normal, long-lived file transfer.

---

### V. Deployment Strategy

1.  **The "Genesis" Event:** As the administrator, you generate the Permanent Salt and all keypairs. You then securely distribute the following to each user:

    - Their unique private key.
    - The Exit Node's public key.
    - The shared Permanent Secret Salt.

2.  **The Exit Node Setup:** On a Linux VPS, you install the Exit Node software and a configuration file containing its private key and the Permanent Salt. You run the software as a service.

3.  **The Client Setup:** Your friends install the Client Application and a configuration file containing their secrets.

---

### VI. Client Requirements & User Experience

- **Software:** The user only needs the custom client application and their configuration file.
- **User Action:** The user runs the client. It does not act as a system-wide VPN. Instead, it creates a local **SOCKS5 proxy** (e.g., on `127.0.0.1:1080`).
- **The Experience:** The user must configure their web browser's network settings to use this local proxy. All traffic from the browser will then be automatically and transparently routed through the stealth tunnel, while other applications on their system use the internet normally.

---

### VII. Security & Limitations

- **Strengths:** Extremely high resistance to automated censorship and blocking. Resilient to server IP changes. Provides location forward secrecy.
- **Weaknesses:**
  - **Trust:** The administrator of the Exit Node has ultimate power and must be trusted completely.
  - **Centralization:** The Exit Node is a single point of failure for traffic.
  - **Not for Public Torrenting:** This system should not be used to download public torrents, as this would create traceable and risky network patterns.
  - **Complexity:** This is a sophisticated, custom system, not a simple consumer application.
