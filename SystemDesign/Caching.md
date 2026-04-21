caching comes up almost every time you need to handle high read traffic. Your database becomes the bottleneck, latency starts creeping up. A Cache is just a temporary storage that keeps recently used data handy so that you can get it much faster next time.


Server --> Database( disk )  ~ 1 ms
Server --> Cache( memory ) ~ 100 ns

Ram is ~ 10000 times faster than ssd


Where to Cache

1. **External Caching:** An external cache is a standalone cache service that your application talks to over the network. This is what most people think of when they hear caching. You store frequently accessed data in something like [Redis](https://www.hellointerview.com/learn/system-design/deep-dives/redis) or [Memcached](https://memcached.org/) so you do not have to hit the database every time.
	![[External_caching.png]]


2. In Process Caching --> Modern machine runs on big machines, we can save network call for the external cache.
		Tradeoff--> inconsistency between application servers as it is with per application server.

![[In_process_caching.png]]



3. CDNs --> 