# High-Level Design (HLD) & Core Principles

Trace is architected to handle continuous, high-volume telemetry from thousands of desktop agents while simultaneously serving real-time, aggregated analytics to web dashboards. This requires a robust High-Level Design (HLD) based on strict engineering principles.

<img width="862" height="659" alt="image" src="https://github.com/user-attachments/assets/a608aed6-2019-4b80-a3d0-9ca811f1c748" />


## Core Engineering Principles

1. **Strict Multi-Tenancy (Row-Level Isolation)**
   Trace is a B2B application where data privacy is non-negotiable. We avoid logical separation in the application layer alone. Instead, every database table carries a `Tenant ID` foreign key. All queries are scoped to the authenticated user's `Tenant ID` by the middleware. This guarantees zero data leakage between tenants.

2. **Decoupling Heavy Workloads from the Event Loop**
   Node.js is single-threaded. CPU-intensive tasks like Base64 decoding and WebP image compression can easily starve the event loop, causing API timeouts for all users. We enforce a strict rule: all heavy processing must be offloaded. We use BullMQ and Redis to queue these tasks, allowing the Express server to return `202 Accepted` instantly while background workers handle the heavy lifting.

3. **Stateless APIs with Purpose-Built Authentication**
   The REST APIs are stateless, meaning the server does not store session data in memory.
   * **Web Clients** use short-lived JSON Web Tokens (JWT) for standard authorization.
   * **Desktop Agents** use persistent, database-backed API keys (`x-agent-token`). This allows administrators to revoke an agent's access instantly without waiting for a token to expire.

4. **Eventual Consistency for Analytics**
   We prioritize high availability and low latency for incoming telemetry. It is acceptable if an admin dashboard takes a few hundred milliseconds to reflect a newly uploaded screenshot. This eventual consistency model allows us to use background queues and read caches aggressively.

5. **Asynchronous Real-Time Streaming (SSE)**
   Instead of forcing the web dashboard to poll the server for updates, we use Server-Sent Events (SSE) to push state changes (e.g., clock-ins, new screenshots) instantly. We chose SSE over WebSockets because our data flow is unidirectional (server to client) and SSE requires significantly less overhead to scale.

## Infrastructure Design

* **Compute:** Containerized Node.js applications deployed on cloud infrastructure. This ensures parity between our local environments and production.
* **Database:** Managed PostgreSQL instances. We rely on Supabase Cloud Pooler to handle connection pooling, preventing our Express servers from exhausting database connections during traffic spikes.
* **Storage:** Cloudflare R2 provides an S3-compatible API for object storage. All buckets are private. We serve files to clients exclusively via temporary pre-signed URLs to maintain tight access control.
* **Queueing:** Upstash Redis handles our BullMQ job queues and acts as a high-speed caching layer for complex SQL aggregations.
