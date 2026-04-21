# Network Protocols Overview

A **Protocol** is a set of rules governing how data is transmitted over a network—it lives at the transport or application layer. Think of it as the physical road and traffic laws.

Protocols define:
* How connections are established
* How data is framed
* How errors are handled
* How bytes move over the wire

---

## 1. The Network / Transport Layer

### TCP/IP — The Foundation
TCP (Transmission Control Protocol) over IP (Internet Protocol) is the bedrock of the internet. IP handles addressing (getting packets to the right machine), and TCP handles reliability (getting them there correctly).

* **How it works:** TCP establishes a connection via a 3-way handshake (`SYN` → `SYN-ACK` → `ACK`), then guarantees ordered, reliable, error-checked delivery. If a packet is lost, TCP detects it and retransmits. Every byte sent is acknowledged.
* **The Cost:** Latency. You pay for the handshake upfront, and you pay for ACKs and retransmits throughout. Everything above TCP (HTTP, WebRTC, gRPC) runs on this layer.
* **Use when:** You need guaranteed delivery, order matters, and you're building anything over the internet.

### UDP — Fire and Forget
UDP (User Datagram Protocol) skips everything TCP guarantees. No handshake, no acknowledgment, no ordering, no retransmit. You send a packet, and it either arrives or it doesn't. The sender never knows.

* **Why use it?** This sounds like a bug, but it's a feature for latency-critical applications. In a video call, a retransmitted frame from 200 ms ago is useless—you'd rather drop it and move on. In online gaming, old position data is garbage.
* **Flexibility:** UDP lets you trade correctness for speed and gives you the freedom to build your own selective reliability on top (which is exactly what QUIC and WebRTC do).
* **Use when:** Real-time media, gaming, DNS lookups, IoT sensors—anywhere stale data is worse than lost data.

---

## 2. HTTP — The Web's Language

HTTP (HyperText Transfer Protocol) is a request-response protocol built on TCP. It is stateless by design—each request is independent, which is why REST was built on it.

* **HTTP/1.1:** Has one request per connection at a time (suffers from head-of-line blocking).
* **HTTP/2:** Introduced multiplexing (many requests over one TCP connection concurrently), plus header compression (HPACK) and server push.
* **HTTP/3:** Runs on QUIC instead of TCP, eliminating TCP's head-of-line blocking entirely.
* **Use when:** All standard web communication, REST APIs, browsers, and CDNs.

### What is Head-of-Line (HOL) Blocking?
> **🛒 Analogy:** Imagine a single-lane grocery checkout. If the person at the front of the line has a massive cart (a large image or a slow database query), everyone behind them is stuck until that transaction finishes.

**In networking:**
1. The client sends a request for `index.html`.
2. The server processes it and sends it back.
3. Only then can the client request `style.css`.
If `index.html` takes 2 seconds to generate, the `style.css` and `script.js` are "blocked" at the head of the line.

### HTTP/2: Binary Framing Layer & Multiplexing
HTTP/2 solved application-layer HOL blocking by breaking every message into small, independent binary frames rather than treating requests as a single block of text.

* **Streams:** A bidirectional flow of bytes assigned a unique Stream ID.
* **Messages:** A complete sequence of frames mapping to a logical request/response.
* **Frames:** The smallest unit of communication (e.g., HEADERS, DATA).
* *How it works:* The receiver uses Stream IDs to reassemble frames in any order. The server can "pause" a 5MB image (Stream 1), inject a tiny CSS file (Stream 2), and resume the image.

**Does HTTP/2 completely eliminate HOL Blocking?**
No. It solves it at the Application Layer (L7), but because it runs on TCP (L4), **TCP-level HOL blocking remains**. If a single packet is lost on the wire, TCP stops *everything* to wait for re transmission. Even if the lost packet belonged to Stream 1, Streams 2 and 3 are blocked. 

| Feature | HTTP/1.1 | HTTP/2 |
| :--- | :--- | :--- |
| **Format** | Plain Text | Binary Frames |
| **Concurrency** | Sequential (one at a time) | Multiplexed (simultaneous) |
| **Connections** | Multiple (expensive) | Single (efficient) |
| **HOL Blocking** | Present at L7 and L4 | Present only at L4 (TCP) |
| **Header Compression** | None (Verbose) | HPACK (Compressed) |

---

## 3. QUIC — HTTP/3's Engine

QUIC (Quick UDP Internet Connections) runs on UDP but re implements TCP's reliability and TLS's security at the user-space level. It is the backbone of HTTP/3, YouTube, and Google Search. 

### Core Architecture
* **Connection IDs (CID):** TCP ties connections to your IP/Port. If you switch from Wi-Fi to 5G, the connection breaks. QUIC uses a CID. If your IP changes, the connection stays alive (**Connection Migration**).
* **Stream-Level Multiplexing:** QUIC treats files as independent streams. If a packet for Stream A is lost, Stream B keeps moving. TCP-level HOL blocking is eliminated.

### The Connection Handshake
QUIC's primary goal is reducing latency by collapsing transport and security handshakes.

* **1-RTT (The First Meet):** Combines transport parameters and TLS 1.3 keys into the first packet. Encryption and connection take just 1 Round Trip (compared to TCP/TLS 1.2 which took 3).
* **0-RTT (The Second Date):** If a client connected previously, it uses a saved Session Ticket. It sends encrypted data immediately in the very first packet.

#### The Replay Attack Vulnerability
While 0-RTT is fast, an attacker can intercept and "replay" the first packet multiple times.

| Handshake | Mechanism | Security Status |
| :--- | :--- | :--- |
| **1-RTT** | Both sides exchange fresh random numbers (Nonces) | **Immune:** Replayed packets fail because the server's nonce changes. |
| **0-RTT** | Uses a saved "Pre-Shared Key" (PSK) | **Vulnerable:** Attacker can replay the first packet. |

* **Safe:** `GET` requests (fetching the same data twice is harmless).
* **Dangerous:** `POST` requests (e.g., `/pay?amount=100` could trigger multiple charges).
* *Fix:* Systems only allow 0-RTT for **Idempotent** requests (actions that don't change server state).

### Fixing Retransmission Ambiguity
TCP uses the same sequence number for original and retransmitted packets, causing ambiguity (did the ACK confirm the first try or the second?). QUIC decouples network delivery from application data:
1.  **Strictly Increasing Packet Numbers:** Packet numbers never repeat. If #5 is lost, data is resent in #8. This allows for perfect RTT measurement.
2.  **Stream Offsets:** QUIC uses offsets inside the packet to place data. Packet #5 and #8 both say "Stream 1, Bytes 1000-1500." The browser just slots the bytes into the file using the offset.

### Smarter, Richer ACKs
TCP ACKs are cumulative (e.g., "I have everything up to byte 1000"). If a middle chunk is missing, TCP panics and slows down. A single QUIC ACK can acknowledge up to **256 distinct blocks** (e.g., "Got 1-10, missed 11-12, got 13-50"). The sender surgically re-transmits only the missing data without slowing the stream.

---

## 4. WebRTC — Peer-to-Peer Real-Time

WebRTC (Web Real-Time Communication) enables direct browser-to-browser communication for audio, video, and data without a central media server. 

* **Tech Stack:** Uses UDP underneath, ICE/STUN/TURN for NAT traversal, DTLS for encryption, and SRTP for media.
* **How it works:** Peers exchange SDP offer/answer messages via your own signaling server (WebSockets, REST, etc.) to figure out public addresses, then establish a direct P2P connection.
* **Use when:** Video calls (Google Meet), screen sharing, multiplayer browser games, P2P file transfers.

---

## 5. Protocol Comparison Summary

| Protocol | Layer | Reliability | Latency | Key Feature | Use When |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **TCP/IP** | Transport | ✅ Guaranteed, ordered | Medium | 3-way handshake, ACKs, retransmit | All internet communication needing reliability. |
| **UDP** | Transport | ❌ Best-effort, unordered | Very low | No handshake, no ACK, just send | Gaming, DNS, real-time media, IoT sensors. |
| **HTTP/1.1** | App | ✅ (via TCP) | Med-High | Text-based, stateless, pipelining | Standard web APIs, browsers. |
| **HTTP/2** | App | ✅ (via TCP) | Low-Med | Multiplexing, HPACK, server push | Modern web apps, gRPC transport. |
| **HTTP/3** | App | ✅ (QUIC-level) | Very Low | 0-RTT resumption, no TCP blocking | High-performance web, CDNs, streaming. |
| **QUIC** | Hybrid | ✅ (User-space TCP+TLS) | Very Low | TLS 1.3 built-in, 0-RTT, no HOL blocking | HTTP/3 transport, low-latency apps. |
| **WebRTC** | Hybrid | Partial (SRTP/SCTP) | Very Low | ICE/STUN/TURN NAT traversal | Video calls, screen share, P2P data transfer. |

---

## 6. Protocol Stack Relationship (OSI Reference)

Higher layers depend on lower layers. API styles sit on top of all of this.

| Layer | Protocol / Standard | What it provides |
| :--- | :--- | :--- |
| **L7 App** | HTTP, WebSocket, gRPC, MQTT | Application semantics, API patterns |
| **L6 Security** | TLS/SSL | Encryption, cert verification |
| **L5 Session** | QUIC (partially) | Session management |
| **L4 Transport** | TCP, UDP | Port addressing, reliability/speed tradeoff |
| **L3 Network** | IP (IPv4/IPv6) | Routing, addressing between machines |
| **L2 Data Link** | Ethernet, Wi-Fi | Node-to-node data transfer |
| **L1 Physical** | Cables, radio, fiber | Bits on the wire |
