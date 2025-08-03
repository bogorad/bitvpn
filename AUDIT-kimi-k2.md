> Audit performed by `moonshotai/kimi-k2` on [OpenRouter.ai](https://openrouter.ai/moonshotai/kimi-k2)

### Security & Correctness Audit

**Document: “Project BitVPN: A High-Stealth Obfuscated Tunneling System”**

#### 1. Executive Summary

Under the given axiomatic constraints (_blocking BitTorrent is politically impossible, DPI is the sole weapon, automated mass-surveillance only_), the document proposes a plausible “BitTorrent-shaped” obfuscation layer. Three broad categories of findings emerge:

- **Correctness / Soundness of Crypto Design**: CRYPTO
- **Information Leakage & Fingerprintability**: DPIL
- **Operational Pitfalls & User-Error Surface**: OPS

Labels are attached to each finding for quick reference.

---

#### 2. CRYPTO — Cryptographic Construction

| #   | Item                                                  | Severity | Notes & Recommendation                                                                                                                                                                                                                                                                             |
| --- | ----------------------------------------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| C1  | Reuse of Permanent Salt across epochs                 | Med-High | A single compromise of the salt allows unlimited retroactive and prospective traffic key recovery. Mitigation: derive an _ephemeral_ root (e.g., HKDF(salt, epoch)) and continue chaining (`k_i = HKDF(k_{i-1}, "epoch")`).                                                                        |
| C2  | AEAD associated data bound only to `pk                |          | seq`                                                                                                                                                                                                                                                                                               | Low | Shortly replaying a _valid_ record under a different epoch would pass signature check but fail decryption. Because `pk` is epoch-bound, the attack is moot. Consider adding explicit epoch tag in AD anyway for documentation clarity. |
| C3  | `obfuscation_key` used as single-stream symmetric key | Med      | An adversary replaying the ClientHello would produce a bitwise identical encrypted blob. While TLS 1.3 prevents session resumption with changed keys, replay can still leak _send-time_ metadata. Add a 96-bit ephemeral nonce (and include in AEAD AD) or switch to a double ratchet – cheap fix. |
| C4  | Derivation formula visibility                         | Low      | Mention that HKDF output length always equal to underlying hash length (32 bytes for SHA256). If larger keys desired (e.g., 256-bit ChaCha20 key + 256-bit Poly1305 sub-key), either call HKDF twice or respect the spec. Proposal: make purpose byte size explicit.                               |

---

#### 3. DPIL — Deep-Packet-Inspection & Behaviour Leakage

| #   | Item                                      | Severity | Notes & Recommendation                                                                                                                                                                                                                                                                                                                                                                  |
| --- | ----------------------------------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | Packet-length side channels               | Medium   | `piece` messages for the bait exchange and cover traffic must be exactly 16 KiB or follow a distribution distilled from real torrent traffic. Current design says “standard BitTorrent piece” but neither enforces nor documents the resulting length pattern. A fixed 16 KiB window fingerprintable against genuine clients having mixed sizes.                                        |
| D2  | Inter-message timing variance             | Medium   | Constant-rate cover promises “continuous stream” but tight burstiness can be highly regular compared to real p2p seeding. Add randomized **jitter** ±10 % around target μ (seed siphoned from TLS 1.3 handshake entropy).                                                                                                                                                               |
| D3  | µTP vs TCP fingerprint                    | Medium   | Many ISP DPI engines flag port-6889/TCP traffic suspect. By forcing clients to reach _your_ server on arbitrary TCP port the packet alone is no longer suspicious, but missing µTP messages (used by most modern torrent clients) may actually _add_ a distinguishability profile. Provide optional µTP endpoint or at least variable port mapping doc.                                 |
| D4  | Protocol identification string            | High     | BitTorrent `handshake` includes 20-byte protocol ID (`0x13'BitTorrent protocol'`) and 8-byte zero-or-ID extension flags. That exact preamble plus 20-byte info-hash is unmistakable fingerprint unless the info-hash itself falls into a whitelisted pool (recurring hashes are block-listed in many GFW nodes). Allow adding a pool of dozens of benign info-hashes rotated per epoch. |
| D5  | Metadata exchange (`ut_metadata`) pattern | Medium   | After <10 pieces most clients stop metadata exchange; continuing long sequence shifted only by idle packets can be abnormal. Mix legitimate `ut_metadata` traffic with _actual_ torrent metadata of a clip-art collection.                                                                                                                                                              |
| D6  | Cover traffic bandwidth floor             | High     | Continuous 16 KiB every 50 ms → 2.6 Mb/s. Unusual for residential ADSL connections → easily profiled. Provide both slow & fast modes, auto-calibrated once per day.                                                                                                                                                                                                                     |

---

#### 4. OPS — Operational & Admin Errors

| #   | Item                                 | Severity | Notes & Recommendation                                                                                                                                                                                                                                                                                         |
| --- | ------------------------------------ | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| O1  | One gigantic shared `Permanent Salt` | High     | If one user leaks it (cargo-cult blog post, github mishap) the entire infrastructure irrevocably deanonymized. Split into: - Network-wide salt (for key derivation) - Per-node root (encrypted with each `node_id`-bound key) Admin-only rotatable nightly shared secrets could be added with forward secrecy. |
| O2  | Epoch discretisation DDoS            | Medium   | All nodes globally publish at HH:00:XX – a naturally attackable synchronization point. Randomize jitter ±15 s and/or implement back-off retry.                                                                                                                                                                 |
| O3  | Geolocation pre-reachability tests   | Low      | Latency test step probes from client address. In aggressive NAT regimes, ICMP may be blocked → apparent “fast” proxy unreachable after TCP three-way handshake. Recommend convergence with real BitTorrent negotiation to test before declaring the node “live”.                                               |
| O4  | Lack of audit logging                | Med-High | No mention of per-node rotation logs in case of compromise. Provide signed append-only lightweight log (e.g., public keyed-hash chain to detect silent reversion).                                                                                                                                             |
| O5  | Client list explosion                | Medium   | Every new user currently gets the same _shared_ salt. Without an onboarding threshold DHT key-space anonymity collapses once client count grows (~N → O(N²)).                                                                                                                                                  |

---

#### 5. Miscellaneous Concerns

| #   | Item                            | Notes                                                                                                                                                                              |
| --- | ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| M1  | SOCKS5 endpoint exposure        | Docs say “binds on 127.0.0.1:1080” — multiple LAN user setups could accidentally expose to 0.0.0.0 if mis-edited. Provide CLI flag for explicit bind interface with safe default.  |
| M2  | `ut_metadata` content filtering | End-node needs to _serve_ bait data files — avoiding distribution of copyrighted material. Use a lightweight public-domain fake torrent (e.g., RFC text or creative commons clip). |

---

### 6. Recommendations (Quick-Action Checklist)

1. **Implement epoch-bound root keys** (mitigation C1).
2. **Add jitter & variable piece length** (D1, D2).
3. **Rotate _info-hash_ pool alongside epoch** (D4).
4. **Introduce µTP stealth option** (D3).
5. **Lower default bandwidth floor or provide adaptive logic** (D6).
6. **Begin per-node / per-user secret splitting architecture** (O1).
7. **Include detailed threat-diagram appendix**: show attenuation vs known DPI libraries (libprotoident, nDPI, RADIUS flow export).

---
