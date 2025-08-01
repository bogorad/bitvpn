# Project BitVPN: A High-Stealth Obfuscated Tunneling System

## Overview

Project BitVPN is a personal, high-stealth VPN alternative designed to evade sophisticated network surveillance and Deep Packet Inspection (DPI). It achieves this by meticulously camouflaging its traffic to be indistinguishable from mundane BitTorrent activity.

This system is not a tool for using the BitTorrent network. Instead, it leverages the BitTorrent protocol as a disguise, creating a secure, resilient, and difficult-to-block tunnel for a small, trusted group of users. The architecture supports multiple exit nodes, providing high availability and performance optimization through intelligent, latency-based node selection.

## Core Concepts

- **BitTorrent Camouflage:** The entire lifecycle of a connection, from server discovery to data transfer, is wrapped in legitimate-looking BitTorrent protocol messages.
- **Dynamic Discovery via DHT:** Instead of relying on static IP addresses, clients find the exit nodes by querying the public BitTorrent DHT for a secret, daily-rotating identifier.
- **Multi-Node Resilience & Performance:** The system is designed to run with multiple exit nodes distributed globally. Clients automatically select the fastest (lowest latency) node and perform instant failover if a node becomes unreachable.
- **Layered Security:** A multi-pronged defense combining strong cryptographic authentication, behavioral mimicry, protocol obfuscation, and payload protection.

## System Architecture

The system consists of three main components:

1.  **The Client Application:** A smart client run by each user. It handles discovery, latency testing, node selection, failover, and all the protocol camouflage logic. It exposes a simple SOCKS5 proxy for user applications.
2.  **The Exit Node Servers:** A fleet of servers you control, distributed in different geographic locations. Each server runs the BitVPN software, listening for authorized clients and forwarding their traffic to the public internet.
3.  **The Public BitTorrent DHT:** The global, decentralized BitTorrent "phonebook." We use this public infrastructure as a secure and robust way for clients to discover the IP addresses of the active Exit Nodes.

## The Cryptographic Foundation

The system's security is built on five distinct cryptographic elements.

1.  **Asymmetric Keypairs (For Authentication):** A standard public/private keypair (e.g., ECDSA) is generated for **each Exit Node** and **each user**. This ensures that only authorized clients can connect, and they will only connect to legitimate, trusted servers.

2.  **The Permanent Secret Salt (The Master Secret):** A single, long, random string of text generated once by the administrator. This **permanent, static** secret is the root from which all daily dynamic secrets are derived.

3.  **The Daily `info_hash` (The Dynamic Public Meeting Point):** A 20-byte code that changes every day, used to find the Exit Nodes on the DHT. It is calculated by all parties using the formula: `info_hash = SHA1(Permanent_Secret_Salt + Current_UTC_Date_String)`. This provides **location forward secrecy**.

4.  **The Daily Obfuscation Key (The Dynamic Internal Secret):** A key used to encrypt the tunnel handshake itself, also changing daily. It is derived independently to ensure it's different from the `info_hash`: `obfuscation_key = SHA1(Permanent_Secret_Salt + Current_UTC_Date_String + "bitvpn_v1")`.

5.  **The Tunneling Protocol (XTLS):** The secure tunnel is built using XTLS (based on TLS 1.3). This protocol is chosen for its speed (leveraging CPU hardware acceleration for AES), modern security features, and battle-tested robustness as the core of HTTPS.

## Flow of a Connection: A Step-by-Step Journey

This is the lifecycle of a connection, layering all stealth and resilience strategies.

#### Phase 1: Discovery & Node Selection

1.  A user starts the Client Application.
2.  The client calculates **today's `info_hash`** using the Permanent Salt and the current UTC date.
3.  It queries the public BitTorrent DHT for peers associated with that `info_hash`. The DHT returns a **list of IP addresses** for all active Exit Nodes.
4.  **The client now performs a quick latency test** on every IP in the list to determine the round-trip time.
5.  It sorts the nodes from lowest latency (fastest) to highest latency (slowest).

#### Phase 2: Connection & The Cover Story

1.  The client attempts to connect to the **fastest available node** from its sorted list.
2.  Upon successful TCP connection, it performs a **perfectly standard BitTorrent handshake**.
3.  To establish a benign context, it engages in a **"bait data" exchange**, requesting and receiving a few real `piece` messages from a pre-shared bait file.
4.  **Failover:** If the connection to the fastest node fails at any point in this phase, the client automatically discards that node and repeats the process with the next-fastest node on its list.

#### Phase 3: The Secret Handshake

1.  With its cover story established, the client begins the real handshake, disguised within the common **`ut_metadata` protocol**.
2.  It calculates **today's `obfuscation_key`**.
3.  It takes its XTLS `ClientHello` message, **encrypts it** with the `obfuscation_key`, and **wraps** it inside a `ut_metadata` message to send.
4.  The Exit Node follows the identical encrypt-then-wrap process for its responses.
5.  A secure XTLS tunnel is established, having defeated behavioral, protocol, and payload analysis.

#### Phase 4: The Secure Tunnel

1.  The XTLS tunnel is now active.
2.  All application traffic is encrypted by the tunnel and wrapped inside standard BitTorrent **`piece` messages** for transfer, appearing as a normal file-sharing session.

## Deployment Guide

#### Administrator Setup (The "Genesis" Event)

1.  **Generate Secrets:**
    - Generate the single **Permanent Secret Salt**.
    - Generate a unique keypair for **each Exit Node server**.
    - Generate a unique keypair for **each user**.
2.  **Deploy Servers:**
    - Set up multiple Linux VPS instances in your desired locations.
    - On each server, install the Exit Node software.
    - Create a configuration file on each server containing its unique **private key** and the shared **Permanent Salt**.
    - Run the software as a service. It will automatically start announcing itself to the DHT.
3.  **Distribute Client Credentials:**
    - Securely transmit the necessary credentials to each user. This includes:
      - Their unique **private key**.
      - The **Permanent Secret Salt**.
      - A **list of all trusted Exit Node public keys**.

#### Client Setup

1.  The user receives the Client Application executable.
2.  They create a configuration file containing their private key, the Permanent Salt, and the list of trusted server public keys.
3.  They run the application.

## User Experience

The Client Application is designed to be simple to use.

1.  The user runs the client.
2.  The client automatically handles discovery, latency testing, and connection.
3.  It creates a local **SOCKS5 proxy** on the user's machine (e.g., at `127.0.0.1:1080`).
4.  The user configures their web browser (or other application) to use this local proxy for all network traffic. All traffic from that application is now securely and transparently routed through the BitVPN network.

## Security & Limitations

- **Trust:** The administrator of the Exit Nodes has ultimate power and must be trusted completely.
- **Attack Surface:** Using multiple servers increases the system's attack surface. It is critical to keep all nodes secure and patched.
- **Not for Public Torrenting:** This system is a VPN alternative. It should not be used to download public torrents, as this would create traceable and risky network patterns.
- **Complexity:** This is a sophisticated, custom system, not a simple one-click consumer application.
