

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

- Because each frame has a Stream ID, the receiver (browser or server) can receive them in any order and "reassemble" them correctly. If a 5MB image is being sent (Stream 1), the server can "pause" it, inject a few frames of a tiny CSS file (Stream 2), and then resume the image.

_"Does HTTP/2 completely eliminate Head-of-Line Blocking?"_

The answer is **No**.

HTTP/2 solves HOL blocking at the **Application Layer (L7)**, but it still runs on **TCP (L4)**. TCP is a "reliable" protocol. If a single packet in the TCP stream is lost on the wire, TCP stops _everything_ and waits for that packet to be retransmitted before letting the data move to the application.

Even if the lost packet belonged to Stream 1, Stream 2 and Stream 3 are now blocked at the TCP level. This is exactly why **HTTP/3** was created—it moves from TCP to **QUIC (UDP)**, where loss in one stream does not affect others.

|**Feature**|**HTTP/1.1**|**HTTP/2**|
|---|---|---|
|**Format**|Plain Text|Binary Frames|
|**Concurrency**|Sequential (one at a time)|Multiplexed (simultaneous)|
|**Connections**|Multiple (expensive)|Single (efficient)|
|**HOL Blocking**|Present at L7 and L4|Present only at L4 (TCP)|
|**Header Compression**|None (Verbose)|HPACK (Compressed)|

----
### QUIC — HTTP/3's Engine

QUIC (Quick UDP Internet Connections) was built by Google, now an IETF standard. It runs on UDP but _reimplements_ TCP's reliability and  [[API Security]] TLS's security — all baked into one protocol at the user-space level (not the kernel). The key win: **0-RTT connection establishment** for known servers (you can send data in the very first packet), and **independent streams** (a lost packet only blocks its own stream, not every other request on the connection — solving TCP's head-of-line blocking).

QUIC is what makes HTTP/3 fast. It's also used internally by YouTube, Google Search, and is increasingly the backbone of modern CDNs.

**Use when:** High-performance web apps, especially on unreliable networks (mobile). You generally use it implicitly through HTTP/3.

## The Core Architecture of QUIC

QUIC changes how we identify a connection. In TCP, a connection is tied to a **4-tuple**: (Source IP, Source Port, Dest IP, Dest Port). If you switch from Wi-Fi to 5G, your IP changes, the TCP connection breaks, and you have to handshake all over again.

- **Connection IDs:** QUIC uses a unique **Connection ID (CID)**. As long as the CID remains the same, the connection stays alive even if your IP address or port changes. This is called **Connection Migration**.
    
- **Stream-Level Multiplexing:** As we discussed, QUIC treats every file (CSS, JS, Image) as an independent stream. If a packet for "Stream A" is lost, "Stream B" keeps moving. **TCP-level Head-of-Line blocking is dead here.**


## The 1-RTT Handshake (The First Meet)

Before we get to 0-RTT, we need to see how QUIC handles a fresh connection. In the old world, you needed one trip for TCP and two for TLS. QUIC does both at once.

1. **Client Hello:** The client sends a UDP packet containing its transport parameters and a TLS 1.3 "Client Hello" (Diffie-Hellman keys).
    
2. **Server Hello:** The server responds with its own keys, transport parameters, and a certificate.
    

**Result:** After **one** round trip (1-RTT), the connection is established AND encrypted.

## How 0-RTT Works (The "Second Date")

0-RTT (Zero Round Trip Time) allows a client to send **encrypted data** in the very first packet of a new connection. This is only possible if the client and server have spoken before.

### The Mechanism: Session Resumption

1. **The First Connection:** During the initial 1-RTT handshake, the server sends the client a **Session Ticket** (specifically a NewSessionTicket frame in TLS 1.3). This ticket contains a "Pre-Shared Key" (PSK) and the server's transport parameters, encrypted so only the server can read it.
    
2. **The Cache:** The client saves this ticket locally.
    
3. **The Re-connection (0-RTT):** When the client wants to connect again, it doesn't wait for a handshake. It sends:
    
    - The **Session Ticket**.
        
    - The **Encrypted Data** (using the PSK from the ticket).
        
4. **Server Receipt:** The server decrypts the ticket, gets the key, decrypts the data, and processes the request immediately.
    

> **Crucial Note:** The client "guesses" that the server will accept the ticket and sends the data right away. If the ticket is expired or rejected, the server just falls back to a 1-RTT handshake.


## The "Catch": Replay Attacks

0-RTT is fast, but it has a significant security trade-off: **Replay Attacks**.

Because the first packet (the one containing the data) is self-contained, a hacker could intercept that UDP packet and "replay" it to the server 100 times.

- **Safe for:** "GET" requests (reading data). Replaying a request for `index.html` just sends the same page back.
    
- **Dangerous for:** "POST" requests (writing data). Replaying a request like `POST /pay?amount=100` could result in multiple charges.
    

**System Design Tip:** Most browsers and servers only allow 0-RTT for "Idempotent" requests (requests that don't change state, like GET).


## The 1-RTT Handshake: Why it is Immune to Replay

In a standard 1-RTT setup, Alice and Bob combine their own temporary secrets to create a completely unique lock for this specific conversation.

- **Message 1 (Alice ➔ Bob):** "Hi Bob, I want to connect. My random number (Nonce) for today is **12345**."
    
- **Message 2 (Bob ➔ Alice):** "Hi Alice. My random number for today is **98765**. Let's mix our numbers together to make our session key."
    
- _Both calculate the key:_ `Key = f(12345 + 98765 + Secrets)`.
    

**The Replay Attack (Mallory's Attempt):** Tomorrow, Mallory wants to trick Bob. She takes the exact packet Alice sent yesterday and replays it.

- **Mallory ➔ Bob:** "Hi Bob, I want to connect. My random number for today is **12345**." _(Mallory just hit play on her recording)._
    
- **Bob ➔ Mallory:** "Hi Alice. My random number for today is **55555**. Let's mix our numbers."
    
- **Why Mallory Fails:** Bob generated a _new_ random number (**55555**). The session key is now `f(12345 + 55555 + Secrets)`. Mallory cannot calculate this new key because she doesn't have Alice's underlying private secrets to do the math. The connection drops.


## The 0-RTT Vulnerability: How Replay Works

In 0-RTT, Alice is in a hurry. She and Bob agreed yesterday to use a Pre-Shared Key (PSK), let's call it **"BlueBird"**.

- **Message 1 (Alice ➔ Bob):** `[Encrypted using "BlueBird": POST /transfer?amount=$100]`
    

**The Replay Attack (Mallory's Attempt):** Mallory intercepts this packet. She cannot read it because she doesn't know the word "BlueBird." But she knows it’s a valid packet going to Bob's bank server.

- **Mallory ➔ Bob:** `[Encrypted using "BlueBird": POST /transfer?amount=$100]`
    
- **Why Mallory Succeeds:** Bob receives the packet. He checks his system and sees, "Ah, 'BlueBird' is a valid PSK for Alice." He decrypts it successfully and processes the $100 transfer.
    
- Mallory hits replay 10 times, and Bob processes 10 transfers. Bob has no way of knowing this is a recording because **he hasn't had a chance to inject a fresh random number yet.**

## The Transition to Safety: Why Subsequent Messages are Secure

You asked: _If the first packet can be replayed, why can't the rest of the conversation be replayed?_

This is the brilliant part of the protocol design. The 0-RTT vulnerability **only exists for the very first split-second**, before Bob replies.

Here is what happens immediately after Alice sends that first 0-RTT message:

- **Message 1 (Alice ➔ Bob):** `[0-RTT: POST /transfer?amount=$100]`
    
- **Message 2 (Bob ➔ Alice):** "I got your transfer. By the way, here is my new random number for today: **77777**. Let's switch to a new, fresh session key right now."
    

**From Message 3 onward, the "BlueBird" key is thrown in the trash.** They are now using a brand new key derived from Bob's fresh random number (**77777**).

If Mallory tries to replay anything from the middle of the conversation, it fails for two reasons:

### Defense Mechanism A: Sequence Numbers

Let's say Alice sends:

- **Message 3:** `[Encrypted with New Key | Sequence #1: GET /balance]`
    

Mallory copies this packet and replays it 5 minutes later. Bob receives it, decrypts it, and looks at the QUIC header. He sees `Sequence #1`. Bob's internal logic says: _"Wait, my counter is already at Sequence #45. I already processed Sequence #1."_ Bob instantly drops the packet as a duplicate.


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
