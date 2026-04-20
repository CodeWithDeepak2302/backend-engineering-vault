

**An API Architecture Style** is a design pattern for _how systems expose and communicate functionality_ to each other — it lives at the application design layer. Think of it as _how you design the vehicles and what roads they're allowed to use_.

API Architecture Style define
1. How clients talk to servers
2. How requests are structured
3. how responses are shaped
4. how contracts are enforced

### REST — The Dominant Web Standard

REST (Representational State Transfer) isn't a protocol — it's an architectural _style_. A truly RESTful API has six constraints: statelessness, client-server separation, cacheability, uniform interface (resources identified by URLs), layered system, and code-on-demand (optional).

In practice, REST means: **resources are nouns in URLs** (`/users/42/orders`), **HTTP verbs are actions** (GET to read, POST to create, PUT/PATCH to update, DELETE to remove), responses are self-descriptive (usually JSON), and each request carries all needed context (stateless). You get browser caching, CDN caching, and HTTP infrastructure for free. Over-fetching (getting more data than you need) and under-fetching (needing multiple requests to get related data) are the classic pain points — which is exactly why GraphQL was invented.

**Uniform Interface (Resources & Verbs):** Everything is treated as a "Resource" (a noun) and identified by a URL. You manipulate these resources using standard HTTP "Verbs" (actions).

- `GET /users/123` (Read user 123)
    
- `POST /users` (Create a new user)
    
- `PUT /users/123` (Replace user 123 entirely)
    
- `PATCH /users/123` (Update a specific field of user 123)
    
- `DELETE /users/123` (Remove user 123)

- **Statelessness:** The server does not remember anything about previous requests. Every single request from the client must contain all the information necessary for the server to understand and process it (e.g., passing an auth token in the header every time).
    
- **Client-Server Separation:** The frontend (UI) and the backend (data storage) are completely independent. They only communicate through the API interface.
    
- **Cacheability:** Responses must explicitly declare whether they can be cached by the browser or a CDN, and for how long.


### The Pros

Why did REST kill SOAP and take over the world?

- **Universal Compatibility:** It runs on standard HTTP. Every programming language, web browser, and server in existence knows how to talk HTTP and parse JSON. You don't need special libraries to make a REST call.
    
- **Highly Scalable:** Because it is strictly stateless, scaling is trivial. You can put 100 API servers behind a load balancer, and it doesn't matter which server handles a user's request because no local session state is stored.
    
- **Free Infrastructure (Caching):** Because REST maps `GET` requests to URLs, you get to leverage the entire internet's existing caching infrastructure. A CDN (like Cloudflare) can cache `GET /api/products` at the edge, meaning your database never even gets hit.
    
- **Intuitive:** Developers understand nouns and verbs. Organizing a system into `/authors`, `/books`, and `/reviews` is a very natural way to model data.


### The Cons

REST is showing its age, particularly in modern, highly interactive web and mobile apps.

- **Over-fetching (The Payload Problem):** Let's say you are building a mobile app that displays a list of users, but you only need their `Name` and `Avatar`. A REST endpoint `GET /users` might return a massive JSON object for every user containing their address, email, phone number, and purchase history. You are downloading wasted data, draining the user's mobile data and battery.
    
- **Under-fetching (The N+1 Problem):** If you want to load a blog post, its author's details, and its top 5 comments, REST might force you to make multiple round trips:
    
    1. `GET /posts/1`
        
    2. `GET /users/42` (the author)
        
    3. `GET /posts/1/comments` This drastically increases latency compared to getting it all at once.
        
- **No Built-In Contract:** Unlike SOAP (WSDL) or gRPC (Protobuf), REST doesn't inherently force the client and server to agree on the shape of the data. If the backend team renames a field from `firstName` to `first_name`, the frontend will break unless they are explicitly communicating. (Tools like Swagger/OpenAPI were invented specifically to patch this weakness).
    
- **Bulky Text:** JSON is text-heavy and takes up more bandwidth and CPU parsing time than binary formats.

**Use when:** Public APIs, CRUD applications, when you need broad client compatibility, microservices communicating across organizations.

----
### SOAP — The Enterprise Veteran

**SOAP (Simple Object Access Protocol)** is the "Enterprise Veteran." If REST is a lightweight, flexible backpack, SOAP is a heavy-duty, armored shipping container.

The most critical distinction to make right away: **REST is an architectural style, but SOAP is a strict protocol.** It has rigid rules, built-in compliance standards, and absolutely no room for interpretation.

SOAP (Simple Object Access Protocol) is a _protocol_ (not just a style) for exchanging structured information using XML. Every SOAP request is an XML envelope with a header and body, sent over HTTP (or SMTP, or JMS). The service is described by a **WSDL** (Web Services Description Language) file — a machine-readable contract that clients can use to auto-generate stubs.

SOAP has built-in standards for security (WS-Security), transactions (WS-AtomicTransaction), and reliability (WS-ReliableMessaging) — which is why banking, healthcare, and government systems still use it. The tradeoff: verbose XML, complex tooling, tight coupling via WSDL, and no browser support without a library. REST + JSON replaced SOAP for most web use cases, but SOAP persists where its enterprise standards are legally or operationally required.

### 1. How it Works: The XML Envelope

SOAP does not use JSON. It communicates exclusively using **XML**. Every single message sent or received must be wrapped in a standardized structure called the **SOAP Envelope**.

- **The Envelope:** The mandatory root element that defines the XML document as a SOAP message.
    
- **The Header (Optional but powerful):** Contains routing data, security credentials (like tokens or digital signatures), and transaction management details.
    
- **The Body:** Contains the actual payload (the data being requested or sent).
    
- **The Fault (Inside the Body):** A standardized way to report errors. If something breaks, SOAP doesn't just return an HTTP 500 status code; it returns a detailed XML "Fault" block explaining exactly what went wrong.
    

### 2. The Secret Weapon: WSDL

You cannot talk about SOAP without talking about **WSDL** (Web Services Description Language).

A WSDL is a massive XML file hosted by the server that acts as an iron-clad legal contract. It explicitly defines:

- Every function the server can perform.
    
- The exact data types (strings, integers) required for every input.
    
- The exact shape of the output.
    

**The Magic:** Because the WSDL is machine-readable, a developer writing in Java or C# can point their IDE at the WSDL URL, and the IDE will automatically write thousands of lines of "stub" code. The developer can then call a remote server just by typing `client.GetBankBalance(accountId)`, without ever manually formatting an HTTP request.


**Use when:** Enterprise integrations, banking/financial systems, healthcare (HL7/FHIR historically used SOAP), legacy system integration where a WSDL contract is required.

----
### GraphQL — Ask for Exactly What You Need

**GraphQL** is often described as the "next evolution" of APIs. While REST is centered around **Resources** (URLs), GraphQL is centered around **Data Graphs** and **Types**.. Instead of multiple REST endpoints each returning fixed shapes, GraphQL exposes a **single endpoint** (`/graphql`) with a **strongly-typed schema**. Clients write queries that describe _exactly_ what fields they want — no more, no less. A mobile app can request a lightweight version of a resource; a desktop app can request the full version. Same endpoint, same server, zero over-fetching.

GraphQL has three operation types: **Query** (read), **Mutation** (write), **Subscription** (real-time, usually over WebSockets). The schema is the contract — introspective and self-documenting. The tradeoffs: complex queries can cause **N+1 problems** (solved by DataLoader), caching is harder (no GET requests by default, so no HTTP cache), and there's a learning curve for backend devs used to thinking in endpoints.

### How it Works: The "Menu" vs. The "Order"

Think of a REST API like a **Fast Food Combo**: You order "Combo #1" (The `/user` endpoint), and you get a burger, fries, and a drink, even if you only wanted the burger.

Think of GraphQL like a **Customizable Buffet**:

- **The Schema (The Menu):** The server defines exactly what data is available (e.g., a User has a Name, an Email, and a list of Orders).
    
- **The Query (The Order):** The client sends a single request describing exactly what it wants. If the client only asks for `Name`, the server only sends the `Name`.

### Core Concepts

- **The Schema:** A strongly typed map of your data. It acts as a "contract" between the frontend and backend.
    
- **The Single Endpoint:** Unlike REST, which has dozens of URLs (`/users`, `/posts`, `/comments`), GraphQL typically has only **one** URL: `/graphql`.
    
- **Resolvers:** Functions on the server that "fetch" the data for each specific field in the query.

**Use when:** Diverse clients (web + mobile + TV) with different data needs, rapid frontend iteration, when over/under-fetching is a real problem.

Example:
##### 1. Define the Data Shapes
type User {
  id: ID!
  username: String!
  email: String!
  posts: [Post!]!     # A user can have many posts
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!       # Every post has one author
  publishedAt: String
}

##### 2. Define how we FETCH data
type Query {
  getUser(id: ID!): User
  recentPosts(limit: Int): [Post!]!
}

##### 3. Define how we CHANGE data
type Mutation {
  createPost(title: String!, content: String!, authorId: ID!): Post!
}

#####  The Query (Fetching Data)

In REST, to get a user and their posts, you might have to hit two endpoints. In GraphQL, you do it in **one request**.

**The Request:** You send this JSON-like structure to the server.

GraphQL

```
query GetUserData {
  getUser(id: "u123") {
    username
    posts {
      title
      publishedAt
    }
  }
}
```

**The Response:** The server responds with a JSON object that mirrors the shape of your query exactly.

JSON

```
{
  "data": {
    "getUser": {
      "username": "dev_deepak",
      "posts": [
        {
          "title": "Understanding QUIC Handshakes",
          "publishedAt": "2026-04-19"
        },
        {
          "title": "The Beauty of B+ Trees",
          "publishedAt": "2026-03-15"
        }
      ]
    }
  }
}
```



----
### gRPC — High-Performance Internal APIs

gRPC (Google Remote Procedure Call) is a framework that lets you call a function on a remote server as if it were local. You define your service and messages in **Protocol Buffers** (`.proto` files) — a binary serialization format 3–10x smaller and faster than JSON. The `.proto` file is compiled to client and server stubs in your target language. Wire format is binary HTTP/2.

gRPC supports four communication patterns: unary (one request, one response), server streaming, client streaming, and **bidirectional streaming** — all over a single HTTP/2 connection. Compared to REST+JSON, gRPC is dramatically faster (binary over HTTP/2 with multiplexing), but requires a `.proto` contract and gRPC toolchain. **==Browser support requires gRPC-Web (a proxy layer), which limits direct browser usage.==**

**Implement with:** Official libraries in Go, Java, Python, C#, Node, Ruby, Dart, Kotlin, Swift. Proto compiler generates stubs.

**Use when:** Internal microservice communication, polyglot services where you want a strict typed contract, high-throughput systems, when latency and payload size matter (e.g., ML inference APIs, inter-datacenter calls).

----
### WebSockets — Full-Duplex Persistent Connection

WebSockets is an application-layer protocol that starts as an HTTP/1.1 request with an `Upgrade: websocket` header. The server agrees, and the connection _transforms_ — no more HTTP request-response cycles. Both sides now have a persistent, **full-duplex** channel: either party can send a message at any time, with low overhead (just a few bytes of framing per message, no HTTP headers).

The connection lives until someone closes it (or the network drops). The server can push data to the client without the client asking. This is the foundation for real-time chat, collaborative editing (Google Docs), live sports scores, trading dashboards, and multiplayer games. The tradeoff: you manage connection state, heartbeats (ping/pong), reconnection logic, and horizontal scaling complexity (sticky sessions or a pub-sub broker to fan out to all server instances).

**Implement with:** Socket.io (Node, adds reconnection/rooms/namespaces on top), ws (Node, bare WebSocket), Channels (Django), ActionCable (Rails). Native browser `WebSocket` API on the client.

**Use when:** Chat, collaborative tools, live dashboards, gaming, any scenario where the server needs to push data frequently.

----
### Server-Sent Events (SSE) — One-Way Server Push

SSE is HTTP — nothing more. The client makes a normal GET request, and the server responds with `Content-Type: text/event-stream`. Instead of closing the connection after sending the body, the server _keeps it open_ and streams newline-delimited events whenever it has something to say. The client's browser handles reconnection automatically (with `Last-Event-ID` to resume), de-duplicates messages, and fires `EventSource` events in JavaScript.

SSE is **unidirectional** (server → client only) and simpler than WebSockets in every dimension: no protocol upgrade, no binary framing, works through HTTP proxies and load balancers, and is trivially cacheable/CDN-friendly. It's now the streaming mechanism of choice for **LLM APIs** (OpenAI, Anthropic — every token streamed to you is an SSE event). The limitation: no client-to-server push, and HTTP/1.1 limits you to 6 concurrent SSE connections per domain (HTTP/2 removes this limit).

**Implement with:** Literally any HTTP server. A `for` loop that writes `data: ...\n\n` to the response stream.

**Use when:** AI token streaming, live notifications, news feeds, stock tickers, progress updates — anything where the server has data to push but the client never needs to send data back over the same channel.


| **Feature**    | **Standard REST (Polling)**               | **WebSockets**                        | **SSE (Server-Sent Events)**               |
| -------------- | ----------------------------------------- | ------------------------------------- | ------------------------------------------ |
| **Direction**  | Client asks $\rightarrow$ Server answers. | Both can talk at once (Full Duplex).  | Server pushes to Client (Uni-directional). |
| **Protocol**   | Standard HTTP.                            | Upgrades to a new protocol (`ws://`). | **Standard HTTP.**                         |
| **Efficiency** | High overhead (new headers every time).   | Low overhead once open.               | **Very low overhead.**                     |
| **Resilience** | Good.                                     | Complex (requires "heartbeats").      | **Automatic reconnection built-in.**       |
| **Example**    | Checking for new emails.                  | High-speed trading or Chat apps.      | **AI streaming, News feeds.**              |
When you ask an AI a second question, you aren't sending that question _through_ the open SSE stream.

1. **Query 1:** Client sends a **POST** request $\rightarrow$ Server opens **SSE Stream A** $\rightarrow$ Server sends tokens $\rightarrow$ **Stream A Closes.**
    
2. **Query 2:** Client sends a **brand new POST** request $\rightarrow$ Server opens **SSE Stream B** $\rightarrow$ Server sends tokens $\rightarrow$ **Stream B Closes.**


----
### Event-Driven Architecture / Pub-Sub — Decoupled Async Messaging

[[Event Driven Architecture]] isn't a protocol or an API style — it's a system architecture pattern. Services communicate by **publishing events** to a broker (Kafka, RabbitMQ, AWS SNS/SQS, Google Pub/Sub) rather than calling each other directly. Subscribers declare interest in event types and receive them asynchronously.

The key properties: **temporal decoupling** (publisher and subscriber don't need to be running at the same time), **spatial decoupling** (publisher doesn't know who's subscribing), and **scale decoupling** (a Kafka topic can fan out to 100 consumers with no change to the producer). Events are immutable facts — "OrderPlaced", "UserSignedUp", "PaymentFailed". This enables [[Event Sourcing and CQRS]] event sourcing, CQRS, and audit logs as first-class patterns. The complexity: eventual consistency is hard to reason about, debugging requires distributed tracing, and message ordering guarantees vary by broker.

**Implement with:** Apache Kafka (high-throughput event streaming), RabbitMQ (flexible routing), AWS SQS/SNS, Redis Pub/Sub (ephemeral, simple), NATS.

**Use when:** Microservice integration, order processing pipelines, notification systems, event sourcing, any workflow where services should be loosely coupled.

----
### Webhooks — The Reverse API

A webhook is conceptually the inverse of a REST API call. Instead of your server calling Stripe's API to ask "did anything happen?", you register a URL with Stripe, and Stripe calls _your_ server when something happens. The payload is a normal HTTP POST with JSON.

Simple, but there are real operational concerns: you need to **verify the signature** (the sender includes an HMAC hash you verify with your shared secret — never skip this), respond with 2xx quickly (do heavy processing async), handle retries idempotently (Stripe will retry failed webhooks, so your handler must be idempotent), and expose a public HTTPS endpoint (hard in local dev — use ngrok or similar). Webhooks are polling eliminated: instead of calling `/payments?status=pending` every minute, Stripe tells you the instant a payment completes.

**Implement with:** Any HTTP server + signature verification. Stripe, GitHub, Twilio, Shopify, and virtually every SaaS product supports webhooks.

**Use when:** Reacting to third-party events (payment completed, PR merged, form submitted), triggering CI/CD pipelines, syncing data between SaaS tools, replacing polling.


| Style             | Type                     | Transport                  | Data Format                | Direction                             | Coupling                        | Schema?                         | Browser Support          | Pros                                                                                     | Cons                                                                                            | Use When                                                                         |
| ----------------- | ------------------------ | -------------------------- | -------------------------- | ------------------------------------- | ------------------------------- | ------------------------------- | ------------------------ | ---------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **REST**          | Architecture Style       | HTTP                       | JSON, XML, any             | Req / Resp                            | Low                             | Optional (OpenAPI)              | ✅ Native                 | Simple, cacheable, universal, HTTP verbs are intuitive, massive ecosystem                | Over/under-fetching, multiple round trips, no strict contract by default                        | Public APIs, CRUD apps, broad client compatibility                               |
| **SOAP**          | Protocol + Style         | HTTP, SMTP, JMS            | XML only                   | Req / Resp                            | Very High (WSDL)                | ✅ WSDL (mandatory)              | ❌ Needs library          | WS-Security, WS-Transaction, auto-generated stubs, strict contract                       | Verbose XML, heavy tooling, tight coupling, no browser support, slow                            | Banking, healthcare, enterprise legacy, where compliance requires WSDL           |
| **GraphQL**       | Query Language + Runtime | HTTP (usually POST)        | JSON                       | Req / Resp + Subscriptions (WS)       | Medium                          | ✅ SDL (mandatory)               | ✅ Native                 | No over/under-fetching, one endpoint, self-documenting, introspection, strong typing     | N+1 queries, complex caching, learning curve, POST-only kills HTTP cache                        | Diverse clients (mobile/web/TV), rapid frontend iteration, complex object graphs |
| **gRPC**          | RPC Framework            | HTTP/2                     | Protobuf (binary)          | Unary + Streaming (bi-directional)    | High (proto contract)           | ✅ .proto (mandatory)            | ⚠️ gRPC-Web proxy needed | 3-10x smaller payloads vs JSON, multiplexed streams, auto-generated stubs, strict typing | No native browser support, proto toolchain required, binary hard to debug, steep learning curve | Internal microservices, polyglot systems, high-throughput APIs, ML model serving |
| **WebSockets**    | Protocol + Pattern       | HTTP → upgrades to WS      | Any (text/binary frames)   | Full-duplex, bidirectional persistent | Medium                          | None built-in                   | ✅ Native (WebSocket API) | Real-time bidirectional, low overhead per message, server push                           | Stateful (scaling complexity), no built-in reconnect, proxy/firewall issues                     | Chat, collaborative editing, multiplayer games, live trading                     |
| **SSE**           | HTTP Streaming Pattern   | HTTP (long-lived GET)      | Text (UTF-8 events)        | Server → Client only                  | Low                             | None                            | ✅ EventSource API        | Dead simple, auto-reconnect, CDN/proxy friendly, just HTTP                               | Unidirectional only, limited to text, HTTP/1.1 has 6 connection limit                           | LLM token streaming, live notifications, news feed, progress bars                |
| **EDA / Pub-Sub** | Architecture Pattern     | Varies (Kafka, AMQP, etc.) | Any (JSON, Avro, Protobuf) | Async, decoupled                      | Very Low (decoupled via broker) | Optional (Avro schema registry) | ❌ Not applicable         | Temporal + spatial decoupling, unlimited fanout, audit log, event sourcing               | Eventual consistency, hard to debug, no direct response, ordering complexity                    | Microservice integration, order processing, notifications, event sourcing, CQRS  |
| **Webhooks**      | Reverse API Pattern      | HTTP (POST)                | JSON (typically)           | Third-party → Your server             | Low                             | None (provider docs)            | N/A (server-side)        | Push-based (no polling), simple HTTP, works with any stack                               | Must expose public URL, retry handling + idempotency needed, signature verification critical    | Reacting to SaaS events (payment, PR, form), CI/CD triggers, SaaS integration    |


Decision Guide — What to use and when

| Scenario                                                      | Best Choice               | Runner-up                        | Why                                                                               |
| ------------------------------------------------------------- | ------------------------- | -------------------------------- | --------------------------------------------------------------------------------- |
| Public web API for third parties                              | REST                      | GraphQL                          | Universal support, familiar, cacheable, no toolchain required                     |
| Mobile app with many different screens needing different data | GraphQL                   | REST                             | Each screen queries exactly what it needs — no over-fetching                      |
| Internal microservice-to-microservice calls                   | gRPC                      | REST                             | Binary efficiency, bidirectional streaming, auto-generated stubs, strict contract |
| Real-time chat / collaborative document editing               | WebSockets                | gRPC streaming                   | Persistent full-duplex, server and client both push                               |
| Stream LLM output token-by-token to browser                   | SSE                       | WebSockets                       | Unidirectional, just HTTP, auto-reconnect, CDN-friendly                           |
| Live sports scores / stock ticker in browser                  | SSE                       | WebSockets                       | Server-push only, simpler than WebSockets, no stateful server needed              |
| Video / audio call in browser                                 | WebRTC                    | WebSockets + media server        | P2P direct, low latency, browser-native, no media relay needed                    |
| React to payment / PR / form submission (SaaS event)          | Webhooks                  | Polling (if webhook unavailable) | Push eliminates polling, trivially simple to receive                              |
| Decouple 10 microservices that need to react to one event     | EDA / Pub-Sub             | Webhooks (simpler)               | Temporal + spatial decoupling, unlimited fanout, durable replay                   |
| Enterprise banking / legacy system integration                | SOAP                      | REST                             | WS-Security/WS-Transaction standards, WSDL auto-stubs, compliance                 |
| High-frequency gaming (UDP needed)                            | UDP / WebRTC data channel | WebSockets                       | UDP: no retransmit cost, stale data drops; WebRTC bridges UDP to browser          |
| High-volume internal event streaming (ML pipelines)           | Kafka (EDA)               | gRPC streaming                   | Durable log, replay, partitioned scale, exactly-once semantics                    |
| Simple server-to-server API, reliability needed               | REST over HTTP/2          | gRPC                             | HTTP/2 multiplexing without needing proto toolchain                               |


Where each technology lives in the stack

Understanding the layer hierarchy clarifies why these aren't competing — they're composable.

|Technology|Category|Sits on top of|Sync / Async|Connection model|Real-time?|Payload|
|---|---|---|---|---|---|---|
|**TCP/IP**|Protocol|IP network|Both|Connection-oriented|N/A (foundation)|Raw bytes|
|**UDP**|Protocol|IP network|Async|Connectionless|✅ Yes|Datagrams|
|**QUIC**|Protocol|UDP|Both|Multiplexed streams|✅ Yes|QUIC frames|
|**HTTP/1.1–3**|Protocol|TCP / QUIC|Sync (req/resp)|Per-request|⚠️ HTTP/2 push|Headers + body|
|**WebRTC**|Protocol suite|UDP (ICE/STUN/TURN)|Async|P2P direct|✅ Yes|RTP/SRTP/SCTP|
|**REST**|API Style|HTTP|Sync|Stateless req/resp|❌|JSON / XML|
|**SOAP**|API Style + Protocol|HTTP / SMTP|Sync|Stateless req/resp|❌|XML only|
|**GraphQL**|API Style|HTTP + WS (subscriptions)|Sync + Async (subs)|Single endpoint|⚠️ Subscriptions only|JSON|
|**gRPC**|API Style + Framework|HTTP/2|Both|Multiplexed streams|✅ Streaming|Protobuf (binary)|
|**WebSockets**|Protocol + Pattern|HTTP upgrade → TCP|Async|Persistent full-duplex|✅ Yes|Text or binary frames|
|**SSE**|HTTP Pattern|HTTP|Async (server push)|Long-lived GET|✅ Yes (server→client)|UTF-8 text events|
|**Webhooks**|Pattern|HTTP POST|Async|3rd party → your server|⚠️ Near-real-time|JSON|
|**EDA / Pub-Sub**|Architecture Pattern|Kafka / AMQP / Redis etc.|Async|Broker-mediated|⚠️ Near-real-time|Any (JSON, Avro, Protobuf)|

The "Is it real-time?" spectrum

|Mechanism|Latency range|How real-time|Notes|
|---|---|---|---|
|UDP|<1ms|🟢🟢🟢🟢🟢 Fastest possible|No reliability — pure speed|
|WebRTC|10–50ms|🟢🟢🟢🟢 Excellent|P2P, built on UDP|
|QUIC / HTTP/3|20–80ms|🟢🟢🟢🟢 Excellent|0-RTT resumption|
|WebSockets|30–100ms|🟢🟢🟢🟡 Very good|Persistent TCP connection|
|gRPC streaming|30–100ms|🟢🟢🟢🟡 Very good|HTTP/2 multiplexed|
|SSE|50–200ms|🟢🟢🟢 Good|HTTP long-poll, server-push only|
|Webhooks|100–2000ms|🟢🟢 Near-real-time|Depends on 3rd party retry logic|
|EDA / Pub-Sub (Kafka)|50–500ms|🟢🟢 Near-real-time|Consumer lag depends on throughput|
|REST polling|1–60 seconds|🟡 Simulated real-time|Delay = polling interval|
|SOAP|100–500ms+|🟡 Synchronous only|XML parsing adds overhead|