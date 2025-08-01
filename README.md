
### I. The Core Concept: A Stealth VPN

The fundamental goal is to create a secure, private internet tunnel for a small, trusted group of people (e.g., you and your friends). However, instead of using a standard VPN protocol like OpenVPN or WireGuard, which can be easily identified and blocked by internet service providers or firewalls, this system is designed for maximum stealth.

Its core principle is **camouflage**. The entire system—from finding the server to transferring data—is meticulously designed to be indistinguishable from mundane, high-volume **BitTorrent traffic**. It does **not** use the BitTorrent network for bandwidth or speed; it only uses its protocol as a disguise to hide in plain sight.

---

### II. The Cast of Characters: System Roles

There are three main components in this ecosystem:

1.  **The Client Application:** This is the custom software you and your friends run on your computers or devices. Its job is to find the Exit Node, establish the secret tunnel, and route your browser's traffic through it.

2.  **The Exit Node Server:** This is a server you control, typically a cheap Virtual Private Server (VPS) running Linux in a country of your choice. It runs the server-side software, which listens for connections from authorized clients, unwraps the disguised traffic, and forwards it to the public internet.

3.  **The Public BitTorrent DHT:** This is the massive, decentralized "phonebook" used by real BitTorrent clients to find each other without a central tracker. We use this public infrastructure as a secure and resilient way for your clients to discover the Exit Node's IP address, even if it changes frequently. We never ask it to transfer our data.

---

### III. The Secret Sauce: Keys & Daily Codes

The system's security and stealth rely on two sets of secrets that you, the administrator, create and distribute once.

1.  **Asymmetric Keypairs (For Authentication):**
    *   Using standard cryptography (e.g., ECDSA), you generate a unique keypair (a public key and a private key) for each person, plus one for the Exit Node.
    *   **Private Key:** Kept absolutely secret by each person. It's their proof of identity.
    *   **Public Key:** Shared openly within the trusted group. It's used to verify identities.
    *   **Purpose:** This ensures that **only authorized users** can connect to your Exit Node.

2.  **The Secret Salt & Daily `info_hash` (For Discovery):**
    *   **The Salt:** You generate a single, long, random string of text (e.g., `a7d9...f4e2`). This salt is distributed once to all users.
    *   **The Daily `info_hash`:** This is a 20-byte code that changes every day. Both the client and server calculate it using the formula: `info_hash = SHA1(The_Secret_Salt + Current_UTC_Date_String)`.
    *   **Purpose:** This code is the "secret meeting spot" on the public DHT. By using a code that changes daily, the system gains **forward secrecy**; even if the salt is compromised in the future, an adversary cannot find the IP addresses your server used in the past.

---

### IV. The Complete Flow: From Startup to Browsing

This is the step-by-step lifecycle of a connection, designed for maximum stealth.

#### Phase 1: Dynamic Discovery (Finding the Meeting Spot)
1.  A user starts the Client Application.
2.  The client calculates **today's `info_hash`** using the shared salt and the current UTC date.
3.  It connects to the public BitTorrent DHT and asks, "Which peers have the content for this `info_hash`?"
4.  The DHT network responds with the IP address of your Exit Node, which is constantly announcing itself to the DHT using the same daily `info_hash`.
5.  If the client fails (e.g., around the UTC midnight transition), it automatically calculates **yesterday's `info_hash`** and tries again, ensuring a seamless handover.

#### Phase 2: The Cover Story (Establishing a Benign Context)
1.  The client connects to the Exit Node's IP address and performs a **perfectly standard BitTorrent handshake**.
2.  To defeat behavioral analysis, the client and server now engage in a **"bait data" exchange**.
    *   The client sends an `interested` message. The server replies with `unchoke`.
    *   The client requests a few random pieces of a pre-shared, innocuous "bait file" (e.g., a public domain text file).
    *   The server dutifully sends back the correct `piece` messages containing the bait data.
3.  **Result:** To any DPI system watching, this connection is now firmly classified as a **benign BitTorrent file transfer session**.

#### Phase 3: The Secret Handshake (Building the Tunnel)
1.  With the "cover story" established, the client initiates the real handshake, hiding it inside the **`ut_metadata` protocol** (a common BitTorrent extension).
2.  It constructs its XTLS `ClientHello` message (the start of a modern TLS 1.3 handshake).
3.  It **obfuscates** this message by encrypting it with a key derived from the daily salt. This erases any recognizable cryptographic structure.
4.  It wraps this encrypted, obfuscated payload inside a `ut_metadata` request message and sends it.
5.  The server receives it, unwraps it, decrypts it, and responds by following the same process for its `ServerHello`.
6.  After a few back-and-forths, a secure, mutually authenticated XTLS tunnel is established. The DPI engine saw nothing but a plausible metadata exchange with a random-looking payload.

#### Phase 4: The Secure Tunnel (Using the Connection)
1.  The XTLS tunnel is now active.
2.  When you browse a website, your client application takes the encrypted data packets from the XTLS tunnel and wraps them inside standard BitTorrent **`piece` messages**.
3.  It sends these `piece` messages to the Exit Node. The Exit Node unwraps them, decrypts the traffic via XTLS, and sends it to the destination website.
4.  Replies come back through the same process. To the outside world, this just looks like a long, slow file transfer, which is perfectly normal for BitTorrent.

---

### V. Deployment Strategy: Bringing it to Life

1.  **The "Genesis" Event:**
    *   As the administrator, you generate the secret salt and all the keypairs for your friends and the server.
    *   You must distribute these secrets securely (e.g., using a tool like Signal, PGP-encrypted email, or in person). Each user gets their own private key, the server's public key, and the shared secret salt.

2.  **The Exit Node Setup:**
    *   Rent a cheap Linux VPS from any provider.
    *   Install the custom Exit Node software (a single binary file).
    *   Create a configuration file containing its private key and the secret salt.
    *   Run the software. It will automatically start listening for connections and announcing itself to the DHT.

3.  **The Client Setup:**
    *   You provide your friends with the custom Client Application (a single executable file).
    *   They place it in a folder with a configuration file containing their private key, the server's public key, and the secret salt.
    *   They run the application.

---

### VI. Client Requirements & User Experience

*   **Software:** The user needs the custom client application and a configuration file. No other special software is required.
*   **User Action:** The user simply runs the client. It will open a local **SOCKS5 proxy** on their machine (e.g., on port `1080`).
*   **The Experience:** This is **not** a system-wide VPN. The user must configure their web browser (or other applications) to use the local SOCKS5 proxy (`127.0.0.1:1080`). Once configured, all traffic from that application is automatically and transparently routed through the stealth tunnel. This provides excellent application-level control.

---

### VII. Security & Limitations

*   **Strengths:** Highly resilient to IP changes, resistant to DPI and behavioral analysis, and provides forward secrecy for the server's location.
*   **Weaknesses:**
    *   **Trust:** You must trust the administrator of the Exit Node completely, as they can see all unencrypted traffic.
    *   **Centralization:** While discovery is decentralized, the Exit Node is a single point of failure and a potential bottleneck.
    *   **Not for Torrenting:** Ironically, you should not use this system to download actual public torrents, as that would mix your private, disguised traffic with real, traceable BitTorrent activity.
    *   **Complexity:** This is a sophisticated system requiring custom software, not a simple one-click app.
