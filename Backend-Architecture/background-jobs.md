# Background Jobs & Queuing Architecture

Trace handles heavy tasks like image compression and complex chron-based aggregations. Doing this synchronously inside an HTTP request handler would block the Node.js event loop, leading to gateway timeouts and crashed instances. We solve this using BullMQ backed by Upstash Redis, combined with native cron jobs.

## The Asynchronous Queue Solution (BullMQ)

When a desktop agent uploads a 2MB-5MB Base64 screenshot, running C++ image compression bindings (`sharp`) directly on the main V8 JavaScript thread is dangerous. If 20 agents hit the server simultaneously, response latency spikes over 10 seconds.

We decouple the reception from the processing:

1. **Express Reception:** The Express controller receives the payload, verifies the user, and pushes a job object to the Redis queue.
2. **Instant Handshake:** Express instantly returns an `HTTP 202 Accepted` status to the desktop agent, typically in under 5ms.
3. **Background Execution:** A separate BullMQ worker thread consumes the job from Redis. It decodes the Base64 string, runs WebP compression, uploads to Cloudflare R2, and inserts the metadata into PostgreSQL.

### Architectural Trade-offs
This architecture introduces **eventual consistency**. The admin dashboard might not see the new screenshot for 500ms while it processes. However, this is an acceptable trade-off to ensure 100% server stability and prevent event loop starvation under traffic spikes.

## The Shift Sweep Cron Engine (`shiftSweep.ts`)

A core problem in workforce monitoring is the "abandoned shift". If an employee's computer loses power or their internet crashes, the desktop agent cannot send a `/clock-out` API call. The shift would remain open in the database forever, inflating billing hours.

We solve this using a background cron sweeper.

Every 60 seconds, the backend executes `shiftSweep.ts`:
1. **Find Candidates:** Queries the DB for active shifts (`endTime` is NULL) that have a recent heartbeat.
2. **Staleness Check:** If `now - lastHeartbeatAt` exceeds the company's grace period (default 90 minutes), the shift is flagged as stale.
3. **Auto-Close:** The system forces the shift closed, setting the `checkoutType` to `heartbeat_timeout`. It retroactively calculates total work seconds up to the very last known heartbeat.
4. **SSE Notification:** It broadcasts a `clock_out` event to the web dashboard so admins see the user instantly go offline.

## Data Retention Cron Engine
To manage storage costs and comply with privacy limits, we run automated daily retention sweeps. Based on a company's subscription plan (e.g., Free tier keeps data for 7 days), the cron job identifies old screenshots, deletes the actual files from Cloudflare R2 via the AWS SDK, and then deletes the metadata rows from PostgreSQL.
