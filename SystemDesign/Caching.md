# Caching Essentials

Caching comes up almost every time you need to handle high read traffic. When your database becomes the bottleneck and latency starts creeping up, caching is the solution. A **cache** is a temporary storage layer that keeps recently used data handy so that you can retrieve it much faster the next time it's requested.

### Latency Comparison
| Operation | Storage Type | Estimated Latency | Speed Difference |
| :--- | :--- | :--- | :--- |
| **Server ➔ Database** | Disk (SSD) | ~ 1 ms | Base |
| **Server ➔ Cache** | Memory (RAM) | ~ 100 ns | **~10,000x faster** |

---

## 1. External Caching

An external cache is a standalone cache service that your application talks to over the network. This is what most people think of when they hear "caching." You store frequently accessed data in systems like **Redis** or **Memcached** so you do not have to hit the database every time. 

* **Advantage:** External caches scale well because every application server can share the same centralized cache.

> **💡 System Design Interview Tip:** > External caching with Redis is the default answer when discussing caching strategies. Interviewers expect you to mention it for any high-traffic system. Start here, then layer on other caching types (like CDN or client-side caching) only if the specific problem calls for them.

![[External_caching.png]]

---

## 2. Content Delivery Networks (CDN)

A CDN is a geographically distributed network of servers that caches content physically close to users. Instead of every request traveling to your origin server, a CDN stores copies of your content at **edge servers** around the world.

Modern CDNs (like *Cloudflare, Fastly, and Akamai*) can cache much more than static files. They can also cache public API responses, HTML pages, and even run edge logic to personalize content or enforce security rules before requests reach your servers. However, the most common and impactful use of a CDN is still **media delivery**.

### How a CDN Works (Media Delivery)
1. A user requests an image from your app.
2. The request is routed to the nearest CDN edge server.
3. **Cache Hit:** If the image is cached there, it is returned immediately.
4. **Cache Miss:** If not, the CDN fetches it from your origin server, stores it locally, and then returns it to the user.
5. Future users in that specific region get the image instantly from the CDN.

* **The Latency Impact:** Without a CDN, every image request travels to your origin. If your server is in Virginia and the user is in India, that adds **250–300 ms** of latency per request. With a CDN, the same image is served from a nearby edge server in **20–40 ms**. That is a massive performance difference.

> **💡 System Design Interview Tip:** > Even though modern CDNs can cache API responses and dynamic content, the safest time to introduce a CDN in an interview is when your system serves static media at scale. Start with that justification first, then expand only if the problem requires it.

### How CDNs Cache API Responses
While static files are easy because they rarely change, API responses are trickier because they often depend on user input or database state. However, if an API endpoint returns data that is **read-heavy and not unique to every single user**, it is a prime candidate for the edge.

1. **The Role of Cache-Control Headers**
   CDNs respect standard HTTP headers. If your origin server sends a response with `Cache-Control: public, max-age=60`, the CDN will store that JSON response for 60 seconds. Any subsequent request for that same endpoint within that minute is served directly from the CDN's Point of Presence (PoP), never touching your backend.
2. **GraphQL Caching**
   Since GraphQL typically uses `POST` requests (which aren't cached by default), modern CDNs and tools like Apollo or Fastly use **Persisted Queries**. This turns a complex `POST` request into a cacheable `GET` request tied to a unique hash.
3. **Edge Computing (Edge Functions)**
   Services like Cloudflare Workers or AWS Lambda@Edge allow you to run backend logic directly at the CDN level. You can:
   * Stitch together parts of an API response.
   * Handle authentication checks at the edge.
   * Modify the response body based on the user's location *before* it leaves the CDN.

---

## 3. In-Process Caching

Modern servers run on machines with massive amounts of RAM. By storing data directly in the application's memory space, we can completely eliminate the network call required to reach an external cache (like Redis).

* **Trade-off (Inconsistency):** Because the cache lives locally inside a specific server, you risk state inconsistency between different application servers (e.g., Server A might have a cached value that was just updated and cleared on Server B).

![[In_process_caching.png]]



