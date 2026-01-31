# Mock Interview Transcript

**Interviewer:** Sarah (Senior Staff Engineer)
**Candidate:** Alex
**Type:** System Design
**Duration:** 45 minutes
**Date:** January 25, 2026

---

## Transcript

**[00:00] Sarah:** Hey Alex! Good to see you. How's your day going?

**[00:05] Alex:** Hey Sarah! Pretty good, thanks. Been studying a lot for interviews lately, so a bit tired but excited.

**[00:15] Sarah:** That's the spirit! Quick question before we start - have you done system design interviews before?

**[00:22] Alex:** Yeah, I've done a few practice rounds. Still working on my pacing though.

**[00:30] Sarah:** Great, that's helpful to know. Alright, let's get started with the actual interview.

**[00:35] Sarah:** So today I'd like you to **design a system** for a URL shortener like bit.ly. You know, where you put in a long URL and get a short one back. Sound good?

**[00:45] Alex:** Sure, sounds good. So, um, let me start by understanding the requirements. What's the scale we're looking at?

**[00:55] Sarah:** Good question! Let's say we have 100 million new URLs created per month, and each shortened URL needs to work for at least 5 years.

**[01:05] Alex:** Okay, so 100 million per month... that's roughly, um, like 40 URLs per second? Let me think... 100 million divided by 30 days, divided by 24 hours, divided by 3600 seconds... yeah, about 40 writes per second.

**[01:20] Sarah:** Nice back-of-envelope math there.

**[01:25] Alex:** Thanks! And for reads, I'm assuming it's much higher since people share shortened URLs. Maybe 100:1 read to write ratio? So 4,000 reads per second?

**[01:35] Sarah:** That's a reasonable assumption. What else do you want to clarify?

**[01:40] Alex:** Let me ask about analytics - do we need to track click counts, geographic distribution, that kind of thing?

**[01:50] Sarah:** Let's say basic analytics - click count and timestamp of last access. We can skip geo for now.

**[02:00] Alex:** Perfect. And one more - should shortened URLs be predictable or random? Like, can users pick custom aliases?

**[02:10] Sarah:** Great question! Let's support both - auto-generated short codes and optional custom aliases.

**[02:15] Alex:** Got it. So to summarize the requirements:
- 100M new URLs/month, ~40 writes/sec
- 4,000 reads/sec
- 5-year retention
- Basic analytics (click count, last access)
- Support custom aliases

**[02:30] Sarah:** Perfect summary. Let's move on to the design.

**[02:35] Alex:** Alright, so for the high-level design, I'm thinking we need:
1. An API gateway to handle incoming requests
2. A URL shortening service
3. A database to store the mappings
4. A cache layer for frequently accessed URLs
5. Maybe a separate analytics service

**[02:55] Sarah:** Okay, tell me more about the database choice.

**[03:00] Alex:** So, um, basically I think we should use PostgreSQL because it's, like, reliable and supports ACID transactions.

**[03:10] Sarah:** Interesting. Why do you think you need ACID transactions for this use case?

**[03:15] Alex:** Oh, um, you know, to make sure... actually, wait. Each URL creation is independent, so we don't really need complex transactions. Maybe a NoSQL database would work better here?

**[03:30] Sarah:** Good catch on correcting yourself. What would be the trade-offs?

**[03:35] Alex:** With NoSQL like DynamoDB or Cassandra, we get:
- Better horizontal scaling
- Lower latency for key-value lookups
- But we lose complex queries

Since our access pattern is basically "given short code, return long URL" - that's a perfect fit for key-value storage.

**[03:55] Sarah:** I like how you're thinking about access patterns. What about the short code generation?

**[04:00] Alex:** So for generating short codes, we need something unique. I'm thinking we could use:
- Option 1: Hash the long URL (like MD5 or SHA)
- Option 2: Use a counter and convert to base62
- Option 3: Random generation with collision check

**[04:20] Sarah:** What are the trade-offs of each?

**[04:25] Alex:** Hashing has collision risk and the codes are longer. Counter-based is predictable which might be a security concern. Random with collision check adds database overhead.

I'd go with counter-based using base62 encoding, but distribute counters across multiple nodes to avoid a single point of failure. Each node gets a range of IDs.

**[04:50] Sarah:** That's not quite right about hash collisions. Can you think about why?

**[04:55] Alex:** Hmm, let me think... oh, actually with hashing, if we truncate the hash to make it shorter, that's where collisions come from. The full hash wouldn't collide, but a 7-character truncation could.

**[05:10] Sarah:** Better. What about the predictability concern with counters?

**[05:15] Alex:** We could add some randomization within each range, or use a bijective function to scramble the counter output while still guaranteeing uniqueness.

**[05:25] Sarah:** Good. Let's talk about caching now.

**[05:30] Alex:** For caching, Redis would be perfect here. With 4,000 reads per second and a power-law distribution of URL popularity, caching the hot URLs would dramatically reduce database load.

**[05:45] Sarah:** How would you handle cache invalidation?

**[05:50] Alex:** Since URL mappings don't change - once a short URL maps to a long one, it's permanent - we don't really need invalidation. We can use TTL-based expiration for cache management. Maybe 24-hour TTL for popular URLs.

**[06:05] Sarah:** What if a URL is deleted?

**[06:10] Alex:** Oh right, good point. If we support deletion, we'd need to invalidate the cache entry. We could use a pub/sub pattern - when a URL is deleted, publish an event that the cache subscribes to.

**[06:25] Sarah:** Good thinking about that edge case. Let's dive deeper into the redirect flow.

**[06:30] Alex:** So when a user clicks a shortened URL:
1. Request hits the API gateway
2. Gateway routes to the redirect service
3. Service checks Redis cache first
4. If cache miss, query the database
5. Update cache with the result
6. Return 301 redirect to the client
7. Async update analytics (click count, timestamp)

**[06:55] Sarah:** Why 301 specifically?

**[07:00] Alex:** 301 is a permanent redirect, which allows browsers to cache the redirect. This reduces load on our servers for repeat visits. But actually... for analytics purposes, we might want 302 temporary redirect so the browser always comes back to us.

**[07:20] Sarah:** That's a great insight about the analytics trade-off. What would you recommend?

**[07:25] Alex:** I'd use 302 if analytics are important to the business, 301 if we want to minimize server load. Since we specified basic analytics, I'd lean toward 302.

**[07:40] Sarah:** Makes sense. What about scaling this system?

**[07:45] Alex:** For scaling:
- **API layer**: Horizontal scaling with load balancer. Stateless services, so easy to add more instances.
- **Database**: Shard by short code prefix. First character gives us 62 shards (a-z, A-Z, 0-9).
- **Cache**: Redis cluster with consistent hashing for distribution.
- **Analytics**: Async processing with Kafka to handle burst writes.

**[08:15] Sarah:** Why shard by prefix instead of hash-based sharding?

**[08:20] Alex:** Hmm, actually hash-based might be more even. Prefix sharding could lead to hot shards if certain characters are more common. With consistent hashing, we get better distribution and easier rebalancing when adding nodes.

**[08:40] Sarah:** Good correction. One more thing - how would you handle the 5-year retention requirement?

**[08:45] Alex:** We'd need to track creation timestamp for each URL. For cleanup:
- Option 1: TTL at database level (if supported)
- Option 2: Batch job that scans for expired URLs
- Option 3: Lazy deletion - check expiry on access

I'd go with Option 2 running daily, plus Option 3 as a backup to catch any that slip through.

**[09:10] Sarah:** What about the storage calculation? Will we have enough space?

**[09:15] Alex:** Let me calculate:
- 100M URLs/month × 12 months × 5 years = 6 billion URLs
- Each record: short code (7 bytes) + long URL (avg 100 bytes) + metadata (50 bytes) = ~157 bytes
- Total: 6B × 157 bytes = 942 GB, roughly 1 TB

That's very manageable for a distributed database.

**[09:40] Sarah:** Excellent math. You really nailed the capacity estimation.

**[09:45] Alex:** Thanks! Should I draw out the architecture diagram?

**[09:50] Sarah:** Yes, please summarize the final design.

**[09:55] Alex:** Sure! Final architecture:
- Load balancer → API Gateway
- Gateway → URL Shortening Service (stateless, horizontally scalable)
- Redis Cluster for caching (consistent hashing)
- DynamoDB/Cassandra for storage (hash-based sharding)
- Kafka → Analytics Service (async processing)
- All services containerized in Kubernetes for orchestration

**[10:15] Sarah:** Great overview. A couple quick questions - what happens if the database is down?

**[10:20] Alex:** For reads, we could serve from cache for cached URLs, and return a 503 for cache misses. For writes, we could queue them in Kafka and process when DB recovers.

**[10:35] Sarah:** What about rate limiting?

**[10:40] Alex:** Good point! I'd implement rate limiting at the API gateway level. Per-IP limits for anonymous users, higher limits for authenticated users. Token bucket algorithm would work well here.

**[10:55] Sarah:** Alright Alex, that was really solid overall. Let me give you some feedback.

**[11:00] Sarah:** First, the positives - your back-of-envelope calculations were excellent. You asked good clarifying questions. I liked how you corrected yourself on the PostgreSQL point - that shows good self-awareness.

**[11:15] Sarah:** Areas to improve - you had some "um" and "like" filler words early on, but you got more confident as we went. Also, you initially jumped to PostgreSQL without thinking about the access patterns first. Always start with access patterns when choosing a database.

**[11:35] Sarah:** One mistake I noticed - you said "100 million per month is 40 writes per second." Let me check that... 100M / 30 / 24 / 3600 = about 38.5, so you rounded correctly. Actually that's fine.

**[11:50] Sarah:** But here's something important - when you talked about consistent hashing for the database, that's typically for caches, not for database sharding. Databases usually use range-based or hash-based partitioning with a partition key, not consistent hashing.

**[12:10] Alex:** Oh, good catch! I conflated the two concepts. So for the database, I should have said hash-based partitioning with the short code as the partition key.

**[12:25] Sarah:** Exactly. Overall, I'd say this is solid E6 level performance. You demonstrated good systems thinking, covered most edge cases, and your calculations were accurate.

**[12:40] Sarah:** For next steps, I'd recommend:
1. Practice more with caching strategies - read about cache-aside vs write-through patterns
2. Review database partitioning vs consistent hashing distinction
3. Check out the "Designing Data-Intensive Applications" book by Martin Kleppmann - chapter 6 on partitioning is excellent

**[13:00] Alex:** Thank you so much Sarah! That's really helpful feedback. I'll definitely work on those areas.

**[13:10] Sarah:** You're welcome! Any questions for me?

**[13:15] Alex:** Actually yes - how do you handle custom aliases at scale? I didn't fully address that.

**[13:25] Sarah:** Good question to bring back up. Custom aliases need additional validation - check for profanity, existing usage, length limits. You'd need a separate lookup table or a reserved namespace in your main table.

**[13:45] Sarah:** Alright, that's time! Thanks Alex, good luck with your interviews.

**[13:50] Alex:** Thanks Sarah, really appreciate it!

---

**End of Transcript**
