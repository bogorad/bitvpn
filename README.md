
# Project BitVPN: A High-Stealth Obfuscated Tunneling System

## Overview

Project BitVPN is a personal, high-stealth VPN alternative designed to evade sophisticated network surveillance and Deep Packet Inspection (DPI). It achieves this by meticulously camouflaging its traffic to be indistinguishable from mundane BitTorrent activity.

This system is not a tool for using the BitTorrent network. Instead, it leverages the BitTorrent protocol as a disguise, creating a secure, resilient, and difficult-to-block tunnel for a small, trusted group of users. The architecture relies on decentralized discovery, where each exit node independently publishes its own encrypted contact information to secret locations within the public DHT. This, combined with support for multiple exit nodes, provides high availability and performance optimization through intelligent, latency-based node selection.

## Core Concepts

*   **BitTorrent Camouflage:** The entire lifecycle of a connection, from server discovery to data transfer, is wrapped in legitimate-looking BitTorrent protocol messages.
*   **Decentralized Discovery via Encrypted DHT Mailboxes:** The system avoids public, trackable announcements. Instead of relying on static IPs or public peer lists, each exit node encrypts its connection details into a "contact record" and stores this small blob in multiple, hard-to-find "mailboxes" on the public BitTorrent DHT. Clients use a shared secret to calculate the location of these mailboxes and decrypt their contents.
*   **Multi-Node Resilience & Performance:** The system is designed to run with multiple exit nodes distributed globally. Clients automatically discover all available nodes, select the fastest (lowest latency), and perform instant failover if a node becomes unreachable.
*   **Layered Security:** A multi-pronged defense combining strong cryptographic authentication, behavioral mimicry, protocol obfuscation, and payload protection.

## System Architecture

The system consists of three main components:

1.  **The Client Application:** A smart client run by each user. It handles the complex discovery process, including calculating mailbox locations, fetching and decrypting contact records, latency testing, node selection, failover, and all protocol camouflage logic. It exposes a simple SOCKS5 proxy for user applications.
2.  **The Exit Node Servers:** A fleet of servers you control, distributed in different geographic locations. Each server runs the BitVPN software, independently and periodically publishing its own encrypted contact record to the DHT and listening for authorized clients to forward their traffic to the public internet.
3.  **The Public BitTorrent DHT:** The global, decentralized BitTorrent "phonebook." We use its BEP-44 mutable-item storage feature as a robust, censorship-resistant, and decentralized database for our encrypted mailboxes.

## The Cryptographic Foundation

The system's security is built on a robust and modern cryptographic schedule, deriving multiple keys from a single master secret for different, independent purposes.

1.  **Asymmetric Keypairs (For Tunnel Authentication):** Standard public/private keypairs (e.g., ECDSA) are used within the XTLS protocol itself to authenticate the client and server during the tunnel's establishment, ensuring a client is talking to a legitimate server and vice-versa. These are not used for the discovery phase.

2.  **The Permanent Secret Salt (The Master Secret):** A single, long, random string of text generated once by the administrator. This **permanent, static** secret is the root from which all dynamic, time-based secrets are derived. It is the sole basis of trust for the discovery mechanism.

3.  **The Epochal Key Schedule (Dynamic Secrets):** All time-limited keys are derived using HKDF-SHA256 from the Permanent Salt and the current time epoch (e.g., `YYYY-MM-DD-HH`). Using HKDF with distinct "info" labels ensures that each key is cryptographically independent.
    *   `K_index = HKDF(salt, info="bitvpn_mailbox_v1", epoch)`: Used to calculate the DHT addresses of the mailboxes.
    *   `K_announce = HKDF(salt, info="bitvpn_announce_v1", epoch)`: An AEAD key (e.g., for XChaCha20-Poly1305) used to encrypt and authenticate the contact record itself.
    *   `obfuscation_key = HKDF(salt, info="bitvpn_tunnel_v1", epoch)`: Used to encrypt the inner XTLS tunnel handshake.

4.  **The Tunneling Protocol (XTLS):** The secure tunnel is built using XTLS (based on TLS 1.3). This protocol is chosen for its speed, modern security features, and battle-tested robustness.

## Flow of a Connection: A Step-by-Step Journey

This is the lifecycle of a connection, layering all stealth and resilience strategies.

#### Phase 1: Discovery & Node Selection (The Mailbox Hunt)

1.  A user starts the Client Application.
2.  The client calculates **this epoch's keys** (`K_index`, `K_announce`) using the Permanent Salt and the current UTC time.
3.  The client calculates a set of potential **mailbox addresses** in the DHT (e.g., `mailbox_address = SHA1(K_index || counter)`).
4.  It performs DHT `GET` requests for these addresses, retrieving any stored blobs (the encrypted contact records).
5.  For each blob received, the client attempts to **AEAD-decrypt** it using `K_announce`. The AEAD's authentication tag inherently verifies the blob was created by a party with the correct Permanent Salt. If decryption or the authentication check fails, the blob is discarded as invalid or garbage.
6.  The client checks the timestamp within the valid record to ensure it is current.
7.  The client aggregates the IP addresses from all valid records into a list of available Exit Nodes.
8.  It then performs a quick **latency test** on every IP in the list and sorts the nodes from fastest to slowest.

#### Phase 2: Connection & The Cover Story

1.  The client attempts to connect to the **fastest available node** from its sorted list.
2.  Upon successful TCP connection, it performs a **perfectly standard BitTorrent handshake**.
3.  To establish a benign context, it engages in a **"bait data" exchange**, requesting and receiving a few real `piece` messages from a pre-shared bait file.
4.  **Failover:** If the connection to the fastest node fails at any point, the client automatically discards it and repeats the process with the next-fastest node on its list.

#### Phase 3: The Secret Handshake

1.  With its cover story established, the client begins the real handshake, disguised within the common **`ut_metadata` protocol**.
2.  It calculates **this epoch's `obfuscation_key`**.
3.  It takes its XTLS `ClientHello` message, **encrypts it** with the `obfuscation_key`, and **wraps** it inside a `ut_metadata` message to send.
4.  The Exit Node follows the identical encrypt-then-wrap process for its responses.
5.  A secure XTLS tunnel is established, having defeated behavioral, protocol, and payload analysis.

#### Phase 4: The Secure Tunnel

1.  The XTLS tunnel is now active.
2.  All application traffic is encrypted by the tunnel and wrapped inside standard BitTorrent **`piece` messages** for transfer, appearing as a normal file-sharing session.

## Deployment Guide

#### Administrator Setup (The "Genesis" Event)

1.  **Generate Secrets:**
    *   Generate the single **Permanent Secret Salt**.
    *   Generate a unique keypair for **each Exit Node server** and **each user** for use by the XTLS protocol.
2.  **Deploy Servers:**
    *   Set up multiple Linux VPS instances.
    *   On each server, install the Exit Node software.
    *   Create a configuration file on each server containing its unique **private key** and the shared **Permanent Salt**.
    *   Run the software as a service. It will automatically start publishing its encrypted contact record to the DHT.
3.  **Distribute Client Credentials:**
    *   Securely transmit the necessary credentials to each user. This includes:
        *   Their unique **private key**.
        *   The **Permanent Secret Salt**.
        *   A **list of all trusted Exit Node public keys** (for authenticating the XTLS tunnel).

#### Client Setup

1.  The user receives the Client Application executable.
2.  They create a configuration file containing their private key, the Permanent Salt, and the list of trusted server public keys.
3.  They run the application.

## User Experience

The Client Application is designed to be simple to use.

1.  The user runs the client.
2.  The client automatically handles the entire discovery, decryption, and connection process.
3.  It creates a local **SOCKS5 proxy** on the user's machine (e.g., at `127.0.0.1:1080`).
4.  The user configures their web browser (or other application) to use this local proxy. All traffic is now securely and transparently routed through the BitVPN network.

## Security & Limitations

*   **Trust:** The administrator of the Exit Nodes has ultimate power and must be trusted completely. The security of the entire system relies on the secrecy of the Permanent Salt.
*   **Attack Surface:** Using multiple servers increases the system's attack surface. It is critical to keep all nodes secure and patched. To further enhance privacy, nodes may be configured to publish their DHT records through a generic upstream proxy, preventing their source IP from being exposed to DHT neighbors during the PUT operation.
*   **Not for Public Torrenting:** This system is a VPN alternative. It should not be used to download public torrents, as this would create traceable and risky network patterns that deviate from the system's intended behavioral camouflage.
