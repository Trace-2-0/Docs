# Load Testing & Performance Benchmarks

To ensure Trace is capable of handling enterprise-scale workloads, we executed rigorous load testing using Grafana k6. The primary bottleneck in any analytical SaaS is the database. Our tests focused on a multi-table SQL join over Users, Teams, Shifts, and Breaks (`GET /api/company/dashboard/stats`).

## 100,000 Record Baseline Test
We initially tested the system with 100,000 records (100 users, 36.5k shifts, 36.5k breaks). 
* **Cloud Performance:** Reached a p95 latency of 4.15s and 0.00% error rate on a shared 0.1 vCPU container.
* **Impact of Indexes:** Removing the composite index degraded performance slightly to 4.35s on cloud, but spiked from 7.10s to 14.29s locally, showing early signs of sequential scan strain.

## The 1.1 Million Record Scale Test

We seeded the database with over 1.1 million records (500 users, 500k shifts, 100k app usages) to simulate a large enterprise over several months of usage. We then hammered the API with 30 concurrent Virtual Users (VUs) continuously for 35 seconds.

### Before Optimization (Un-indexed Baseline)
We ran the initial tests without composite indexes to establish a worst-case baseline. 
* **p95 Latency:** 20.68 seconds
* **System Stability:** Poor. The database fell back to sequential scans. Connection pools saturated, leading to a massive 65.2% request failure rate under peak load. The server struggled to stay upright.

### After Optimization (B-Tree Indexes + Upstash Redis)
We applied a targeted B-Tree composite index (`@@index([Tenant ID, date])`) to the Prisma schema and introduced Upstash Redis as an in-memory response cache with a 300-second TTL.

* **p95 Latency:** 1.49 seconds (Client-side), with server execution dropping to 4ms on cache hits.
* **System Stability:** Perfect. 0.00% failure rate (418 successful requests processed with zero drops).

## Background Queue Performance (BullMQ)

We also benchmarked the screenshot upload endpoint (`POST /api/agent/screenshot`), which involves Base64 decoding, Sharp WebP compression, and disk/cloud writes.

* **Synchronous Processing (Before):** Averaged 850ms per request. Under load, this would block the event loop and trigger 504 Gateway Timeouts.
* **Asynchronous Queue (After):** By offloading the processing to BullMQ, the Express API response time dropped to under 15ms. The Express route merely pushes the payload to Redis and returns a `202 Accepted`, leaving the background worker to execute the 850ms job asynchronously.

## Conclusion

The architecture gracefully handles 1+ million record aggregations and high-concurrency image uploads. By leaning heavily on Redis for both response caching and asynchronous job queuing, the Node.js main thread remains unblocked and highly responsive.
