# Project BitVPN: A High-Stealth Obfuscated Tunneling System

## Overview

Project BitVPN is a personal, high-stealth VPN alternative designed to evade sophisticated network surveillance and Deep Packet Inspection (DPI). It achieves this by meticulously camouflaging its traffic to be indistinguishable from mundane BitTorrent activity.

This system is not a tool for using the BitTorrent network. Instead, it leverages the BitTorrent protocol as a disguise, creating a secure, resilient, and difficult-to-block tunnel for a small, trusted group of users. The architecture relies on a decentralized and authenticated discovery mechanism. Each exit node uses a unique, time-rotated signing key to publish its encrypted contact information as a BEP-44 mutable item in the public DHT. This provides strong guarantees of origin and freshness, preventing discovery pollution while keeping the contact details confidential.

## Core Concepts

- **BitTorrent Camouflage:** The entire lifecycle of a connection, from server discovery to data transfer, is wrapped in legitimate-looking BitTorrent protocol messages.
- **Authenticated Discovery via BEP-44:** The system uses BEP-44 mutable items for discovery. Each exit node has a cryptographic identity on the DHT, allowing it to publish signed updates. This prevents unauthorized modification, replay attacks, and spam, ensuring clients can reliably find the freshest, authentic contact record.
- **Payload Confidentiality via AEAD:** While BEP-44 authenticates the _publisher_, the _content_ of the contact record (the node's IP address) remains confidential, protected by strong AEAD (Authenticated Encryption with Associated Data) encryption.
- **Constant-Rate Cover Traffic:** To defeat traffic correlation and timing analysis, the system eliminates revealing periods of silence. During user inactivity, the client and server automatically generate and exchange encrypted "noise" packets disguised as standard `piece` messages. This smooths the traffic flow from a bursty, interactive pattern into a continuous stream, more closely mimicking a large file transfer and frustrating behavioral fingerprinting.
- **Multi-Node Resilience & Performance:** The system is designed to run with multiple exit nodes. Clients discover all available nodes, select the fastest (lowest latency), and perform instant failover if a node becomes unreachable.

## System Architecture

The system consists of three main components:

1.  **The Client Application:** A smart client run by each user. It handles the full, layered discovery process: deriving the expected public keys for the current epoch, fetching the corresponding mutable items from the DHT, verifying their signatures and sequence numbers, and only then decrypting the confidential payload. It then performs latency testing, node selection, and all protocol camouflage logic, exposing a simple SOCKS5 proxy.
2.  **The Exit Node Servers:** A fleet of servers you control. Each server runs the BitVPN software, independently deriving its own signing key for the current epoch. It periodically publishes its encrypted contact record as a signed, sequence-numbered update to its designated mutable item slot in the DHT.
3.  **The Public BitTorrent DHT:** The global, decentralized BitTorrent "phonebook." We use its BEP-44 mutable-item storage feature as a robust, censorship-resistant, and authenticated database. DHT nodes themselves help reject invalid or stale updates, reducing the burden on our clients.

## The Cryptographic Foundation

The system's security is built on a layered cryptographic schedule, deriving multiple keys from a single master secret for distinct, independent purposes.

1.  **The Permanent Secret Salt (The Master Secret):** A single, long, random string of text generated once by the administrator. This **permanent, static** secret is the root from which all dynamic, time-based secrets are derived.

2.  **The Epochal Key Schedule (Dynamic Secrets):** All time-limited keys are derived using HKDF-SHA256 from the Permanent Salt and the current time epoch (e.g., `YYYY-MM-DD-HH`). Using HKDF with distinct "info" labels ensures that each key is cryptographically independent.

    - **DHT Identity Keypair (`pk`/`sk`):** `sk = HKDF(salt, info="bitvpn_dht_id_v1", epoch || node_id)`. This Ed25519 keypair is used to sign and authenticate BEP-44 mutable item updates. The public key (`pk`) serves as the "address" for the item in the DHT.
    - **AEAD Payload Key (`K_announce`):** `K_announce = HKDF(salt, info="bitvpn_announce_v1", epoch)`. An AEAD key (e.g., for XChaCha20-Poly1305) used to encrypt the contact record itself, ensuring its confidentiality.
    - **Tunnel Obfuscation Key (`obfuscation_key`):** `obfuscation_key = HKDF(salt, info="bitvpn_tunnel_v1", epoch)`. Used to encrypt the inner XTLS tunnel handshake.

3.  **The Tunneling Protocol (XTLS):** The secure tunnel is built using XTLS (based on TLS 1.3), chosen for its speed, modern security features, and battle-tested robustness.

## Flow of a Connection: A Step-by-Step Journey

This is the lifecycle of a connection, layering all stealth and resilience strategies.

#### Phase 1: Discovery & Node Selection (Authenticated Fetch)

1.  A user starts the Client Application.
2.  The client calculates **this epoch's keys**. For each known `node_id`, it derives the expected Ed25519 public key (`pk`) that the node should be using.
3.  It performs a DHT `GET` request for the mutable item whose address is the derived `pk`.
4.  The DHT returns a response containing the value (`v`), sequence number (`seq`), and signature (`sig`).
5.  **The client first verifies the BEP-44 envelope.** It checks that the `sig` is a valid signature over the `seq` and `v` for the given `pk`. It also checks that the `seq` is fresh (e.g., greater than the last one seen). If either check fails, the record is discarded immediately as invalid, stale, or forged. This step requires minimal CPU and protects against pollution.
6.  **Only after the envelope is verified** does the client proceed to decrypt the payload. It uses `K_announce` to AEAD-decrypt the value (`v`). The Associated Data for the AEAD includes the `pk` and `seq` to cryptographically bind the confidential payload to its authenticated envelope.
7.  The client aggregates the IP addresses from all valid, verified, and decrypted records into a list of available Exit Nodes.
8.  It then performs a quick **latency test** on every IP in the list and sorts the nodes from fastest to slowest.

#### Phase 2: Connection & The Cover Story

1.  The client attempts to connect to the **fastest available node** from its sorted list.
2.  Upon successful TCP connection, it performs a **perfectly standard BitTorrent handshake**.
3.  To establish a benign context, it engages in a **"bait data" exchange**, requesting and receiving a few real `piece` messages from a pre-shared bait file.
4.  **Failover:** If the connection fails, the client automatically discards that node and repeats the process with the next-fastest node.

#### Phase 3: The Secret Handshake

1.  With its cover story established, the client begins the real handshake, disguised within the common **`ut_metadata` protocol**.
2.  It calculates **this epoch's `obfuscation_key`**.
3.  It takes its XTLS `ClientHello` message, **encrypts it** with the `obfuscation_key`, and **wraps** it inside a `ut_metadata` message to send.
4.  The Exit Node follows the identical encrypt-then-wrap process for its responses.
5.  A secure XTLS tunnel is established.

#### Phase 4: The Secure Tunnel & Active Camouflage

1.  The XTLS tunnel is now active.
2.  All application traffic is encrypted by the tunnel and wrapped inside standard BitTorrent **`piece` messages** for transfer.
3.  **Active Camouflage via Cover Traffic:** To mask the natural rhythm of user activity, the tunnel actively conceals periods of idleness.
    - If no real data has been sent for a short, randomized period, both the client and the server will independently generate a **cover traffic packet**.
    - This packet, marked internally with a discard signal, is encrypted and wrapped in a `piece` message just like real data.
    - The receiving end decrypts the packet, recognizes the discard signal, and silently drops it.
    - From an external observer's perspective, the flow of `piece` messages is continuous and steady, effectively hiding when the user is active or idle.

## Deployment Guide

#### Administrator Setup (The "Genesis" Event)

1.  **Generate Secrets:**
    - Generate the single **Permanent Secret Salt**.
    - Assign a unique, stable `node_id` (e.g., "exit01", "exit02") to each planned Exit Node.
2.  **Deploy Servers:**
    - Set up multiple Linux VPS instances.
    - On each server, install the Exit Node software.
    - Create a configuration file on each server containing its unique `node_id` and the shared **Permanent Salt**.
    - Run the software. It will automatically derive its epochal keys and begin publishing signed, encrypted updates to the DHT.
3.  **Distribute Client Credentials:**
    - Securely transmit the necessary credentials to each user. This includes:
      - The **Permanent Secret Salt**.
      - A **list of all trusted `node_id`s**.

#### Client Setup

1.  The user receives the Client Application executable.
2.  They create a configuration file containing the Permanent Salt and the list of trusted `node_id`s.
3.  They run the application.

## User Experience

1.  The user runs the client.
2.  The client automatically handles the entire discovery, verification, decryption, and connection process.
3.  It creates a local **SOCKS5 proxy** (e.g., at `127.0.0.1:1080`).
4.  The user configures their applications to use this proxy.

## Security & Limitations

- **Trust:** The security of the entire system relies on the secrecy of the Permanent Salt.
- **Discovery Resilience:** The use of BEP-44 with signed, sequenced updates provides excellent resilience against DHT-level pollution, eclipse, and replay attacks. This ensures discovery remains fast and reliable even in a hostile network environment.
- **Behavioral Resilience:** The use of constant-rate cover traffic hardens the system against timing and traffic correlation attacks by smoothing the data flow to more closely resemble a continuous file transfer.
- **Not for Public Torrenting:** This system is a VPN alternative. It should not be used to download public torrents, as this would create traceable network patterns that deviate from the system's intended behavioral camouflage.
