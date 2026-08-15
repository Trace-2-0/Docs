# Load Testing & Performance Benchmarks

To ensure Trace is capable of handling enterprise-scale workloads, we executed rigorous load testing using Grafana k6. The primary bottleneck in any analytical SaaS is the database. Our tests focused on a multi-table SQL join over Users, Teams, Shifts, and Breaks (`GET /api/company/dashboard/stats`).

Here are the Top 5 most critical benchmark runs from our load testing suite.

---

## 1. Render Production Cloud Benchmark (Sustained 30 VUs Load @ 1.1M DB Scale)
**Run 20 Summary:** This is the definitive production test. We hammered the live cloud container with 30 Virtual Users (VUs) against a 1.1 million record database, utilizing both our B-Tree composite index and Upstash Redis caching.
* **Environment:** Render Cloud Web Service + Upstash Redis Cloud
* **DB Data Load:** 1.1 Million Records (500 users, 500k shifts)
* **p95 Latency:** 1.49 seconds (Client-side)
* **Server-side Execution:** 4ms (Cache Hit) / 4014ms (Uncached DB Fallback)
* **Error Rate:** 0.00% (418 successful requests processed with zero drops)

---

## 2. Localhost Express Engine WITH Redis Response Caching (1.1 Million DB Scale)
**Run 17 Summary:** A controlled test on the local engine against the massive 1.1M record database, verifying the sheer impact of Upstash Redis response caching before deploying to the cloud.
* **Environment:** Localhost Engine (`http://localhost:4000/api`)
* **p95 Latency:** 7.00 seconds (Massive improvement from the 50.32s un-cached baseline)
* **Cache Hit Latency:** 12ms - 15ms (Sub-15ms RAM Lookup)
* **Error Rate:** 0.00% (100% system stability)

---

## 3. BullMQ Background Job Queue Benchmark
**Run 21 Summary:** Tested the `POST /api/agent/screenshot` endpoint to measure the impact of offloading heavy Image Compression (Sharp WebP) and cloud storage uploads to a background worker.
* **Synchronous Processing (Before):** Averaged ~850ms per request (which would block the Node.js event loop).
* **Asynchronous Queue (After):** Response time dropped to `< 15ms` (VPC internal Redis LPUSH). The API immediately returns `202 Accepted` while workers handle the heavy lifting.

---

## 4. Render Cloud Node with 100,000 DB Records (Container Stress Test)
**Run 8 Summary:** An intermediate scale test running against the live cloud container with a smaller 100k record database to verify base container stability.
* **Environment:** Render Cloud Node (`https://trace-backend-mt1h.onrender.com/api`)
* **DB Data Load:** 100,000+ Total DB Records
* **p95 Latency:** 4.15 seconds
* **Error Rate:** 0.00% (100% cloud container stability under a shared 0.1 vCPU limit)

---

## 5. Render Cloud Node WITHOUT B-Tree Index (1.1 Million DB Scale)
**Run 12 Summary:** The catastrophic worst-case scenario. We purposefully dropped the `@@index([Tenant ID, date])` B-Tree composite index and hit the 1.1 million record database to prove the absolute necessity of our indexing strategy.
* **Environment:** Render Cloud Node
* **p95 Latency:** 18.36 seconds (Spiked massively from 4.15s)
* **Error Rate:** 19.35% (Connections were closed by the remote host due to un-indexed sequential scan saturation causing timeouts)

---

## Conclusion
The architecture gracefully handles 1+ million record aggregations and high-concurrency image uploads. By leaning heavily on targeted PostgreSQL B-Tree indexing, Upstash Redis response caching, and BullMQ asynchronous job queuing, the Node.js main thread remains unblocked and highly responsive under enterprise load.
