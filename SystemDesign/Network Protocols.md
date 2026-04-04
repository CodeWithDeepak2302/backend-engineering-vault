

**A Protocol** is a set of rules governing _how_ data is transmitted over a network — it lives at the transport or application layer. Think of it as the _physical road and traffic laws_.

Protocols define
1. How connections are established
2. How data is framed
3. How errors are handled
4. How bytes move over the wire

**Protocols (The Network/Transport Layer)**

### TCP/IP — The Foundation

TCP (Transmission Control Protocol) over IP (Internet Protocol) is the bedrock of the internet. IP handles addressing (getting packets to the right machine), TCP handles reliability (getting them there _correctly_).

TCP establishes a connection via a **3-way handshake** (SYN → SYN-ACK → ACK), then guarantees **ordered, reliable, error-checked delivery**. If a packet is lost, TCP detects it and retransmits. Every byte sent is acknowledged. This reliability has a cost: latency. You pay for the handshake upfront, and you pay for ACKs and retransmits throughout. Everything above TCP — [[#HTTP — The Web's Language]], [[#WebRTC — Peer-to-Peer Real-Time]], [[API Architecture styles#gRPC — High-Performance Internal APIs#]] — runs on this layer.

**Use when:** You need guaranteed delivery, order matters, and you're building anything over the internet.

### UDP — Fire and Forget

UDP (User Datagram Protocol) skips everything TCP guarantees. No handshake, no acknowledgment, no ordering, no retransmit. You send a packet and it either arrives or it doesn't. The sender never knows.

This sounds like a bug, but it's a feature for latency-critical applications. In a video call, a retransmitted frame from 200 ms ago is _useless_ — you'd rather drop it and move on. In online gaming, old position data is garbage. UDP lets you trade correctness for speed, and gives you the freedom to build your own selective reliability on top (which is exactly what QUIC and WebRTC do).

**Use when:** Real-time media, gaming, DNS lookups, IoT sensors — anywhere stale data is worse than lost data.


----
### HTTP — The Web's Language

HTTP (HyperText Transfer Protocol) is a request-response protocol built on TCP. HTTP/1.1 has one request per connection at a time (head-of-line blocking). HTTP/2 introduced **multiplexing** — many requests over one TCP connection concurrently, plus header compression (HPACK) and server push. HTTP/3 (the latest) runs on QUIC instead of TCP, eliminating TCP's head-of-line blocking entirely.

HTTP is stateless by design — each request is independent. This simplicity is why REST was built on it.

**Use when:** All standard web communication. REST APIs, browsers, CDNs — everything.

### What is Head-of-Line (HOL) Blocking?

Imagine a single-lane grocery checkout. If the person at the front of the line has a massive cart (a large image or a slow database query), everyone behind them is stuck until that transaction finishes.

In networking:

- The client sends a request for `index.html`.
   
- The server processes it and sends it back.
  
- **Only then** can the client request `style.css`.
  
- If `index.html` takes 2 seconds to generate, the `style.css` and `script.js` are "blocked" at the head of the line.

## HTTP/2: The Binary Framing Layer & Multiplexing

HTTP/2 solved HOL blocking by introducing a **Binary Framing Layer**. Instead of treating the request as a single block of text, it breaks every message into small, independent **frames**.

### How Multiplexing Works

Multiplexing allows the client and server to interleave multiple requests and responses over a **single TCP connection** simultaneously.

- **Streams:** A bidirectional flow of bytes. Each "request/response" pair is assigned a unique **Stream ID**.
    
- **Messages:** A complete sequence of frames that map to a logical request or response.
    
- **Frames:** The smallest unit of communication (e.g., HEADERS frame, DATA frame).


----
### QUIC — HTTP/3's Engine

QUIC (Quick UDP Internet Connections) was built by Google, now an IETF standard. It runs on UDP but _reimplements_ TCP's reliability and TLS's security — all baked into one protocol at the user-space level (not the kernel). The key win: **0-RTT connection establishment** for known servers (you can send data in the very first packet), and **independent streams** (a lost packet only blocks its own stream, not every other request on the connection — solving TCP's head-of-line blocking).

QUIC is what makes HTTP/3 fast. It's also used internally by YouTube, Google Search, and is increasingly the backbone of modern CDNs.

**Use when:** High-performance web apps, especially on unreliable networks (mobile). You generally use it implicitly through HTTP/3.

----
### WebRTC — Peer-to-Peer Real-Time

WebRTC (Web Real-Time Communication) is a suite of protocols enabling direct browser-to-browser communication for audio, video, and data — without a server in the media path. It uses UDP underneath (via ICE/STUN/TURN for NAT traversal), DTLS for encryption, and SRTP for media transport.

The flow: two peers exchange _SDP offer/answer_ messages (via a signaling server you build), use STUN/TURN to figure out their public addresses, then establish a direct peer-to-peer connection. The signaling channel can be anything — WebSockets, REST, even email. WebRTC defines only what happens _after_ the connection.

**Use when:** Video calls (Google Meet, Jitsi), screen sharing, multiplayer browser games, peer-to-peer file transfer.

----


| Protocol     | Layer     | Connection            | Reliability                             | Latency            | Direction                        | Key Feature                                  | Pros                                               | Cons                                                         | Use When                                       |
| ------------ | --------- | --------------------- | --------------------------------------- | ------------------ | -------------------------------- | -------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------ | ---------------------------------------------- |
| **TCP/IP**   | Transport | Connection-oriented   | ✅ Guaranteed, ordered                   | Medium (handshake) | Full-duplex                      | 3-way handshake, ACKs, retransmit            | Reliable delivery, universal support               | Handshake overhead, head-of-line blocking                    | All internet communication needing reliability |
| **UDP**      | Transport | Connectionless        | ❌ Best-effort, unordered                | Very low           | Full-duplex                      | No handshake, no ACK, just send              | Ultra-low latency, no connection cost              | Packet loss, no ordering, no congestion control              | Gaming, DNS, real-time media, IoT sensors      |
| **HTTP/1.1** | App       | Over TCP              | ✅ (via TCP)                             | Medium-High        | Request-response                 | Text-based, stateless, pipelining            | Universal, human-readable, cacheable               | Head-of-line blocking, verbose headers                       | Standard web APIs, browsers                    |
| **HTTP/2**   | App       | Over TCP              | ✅ (via TCP)                             | Low-Medium         | Request-response + push          | Multiplexing, HPACK compression, server push | Multiple streams, header compression               | TCP head-of-line blocking still exists                       | Modern web apps, gRPC transport                |
| **HTTP/3**   | App       | Over QUIC (UDP)       | ✅ (QUIC-level)                          | Very Low           | Request-response + push          | 0-RTT resumption, no TCP blocking            | Fastest, mobile-friendly, independent streams      | Newer, less middleware support, UDP firewalls                | High-performance web, CDNs, streaming          |
| **QUIC**     | Hybrid    | Over UDP              | ✅ (reimplemented in user-space)         | Very Low (0-RTT)   | Full-duplex, multiplexed streams | TLS 1.3 built-in, 0-RTT, no HOL blocking     | Combines UDP speed + TCP reliability + TLS         | Complex to implement, UDP blocked sometimes                  | HTTP/3 transport, low-latency apps             |
| **WebRTC**   | Hybrid    | Peer-to-peer over UDP | Partial (SRTP for media, SCTP for data) | Very Low           | Full-duplex P2P                  | ICE/STUN/TURN NAT traversal, DTLS encryption | Direct P2P, no media server needed, browser native | Complex signaling, NAT traversal issues, needs TURN fallback | Video calls, screen share, P2P data transfer   |

Protocol Stack Relationship

Higher layers depend on lower layers. API styles sit on top of all of this.

|Layer|Protocol/Standard|What it provides|
|---|---|---|
|L7 App|HTTP, WebSocket, gRPC, MQTT|Application semantics, API patterns|
|L6 Security|TLS/SSL|Encryption, cert verification|
|L5 Session|QUIC (partially)|Session management|
|L4 Transport|TCP, UDP|Port addressing, reliability/speed tradeoff|
|L3 Network|IP (IPv4/IPv6)|Routing, addressing between machines|
|L2 Data Link|Ethernet, Wi-Fi|Node-to-node data transfer|
|L1 Physical|Cables, radio, fiber|Bits on the wire|
