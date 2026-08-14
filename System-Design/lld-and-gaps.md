# Low-Level Design (LLD) & Implementation Gaps

This document dives into the Low-Level Design (LLD) of specific critical components in the Trace backend, while candidly acknowledging where the current implementation deviates from the ideal architectural design. 

## Component LLD: The Shift Sweep Engine

One of the most complex low-level operations in Trace is managing abandoned shifts. When an employee's machine loses power or internet, the desktop agent cannot send a clean `clock-out` request.

### Implementation Logic
We run a background cron job (`src/cron/shiftSweep.ts`) every 60 seconds to detect and close stale shifts.
1. **Fetch Candidates:** Query Prisma for all active shifts where `endTime` is NULL.
2. **Evaluate Staleness:** Compare `lastHeartbeatAt` to the current time. If the difference exceeds the company's configured `heartbeatGraceSecs` (default 90 minutes), the shift is deemed stale.
3. **Auto-Checkout:** The system updates the shift `endTime` using either the explicit `disconnectAt` timestamp (if the OS warned us of a shutdown) or the `lastHeartbeatAt` timestamp. The `checkoutType` is flagged as `heartbeat_timeout`.
4. **Broadcast & Audit:** The system computes the final work totals, broadcasts a `shift.clock_out` event to the admin dashboard via SSE, and logs an immutable audit trail.

## Implementation Gaps & Technical Debt

While the system is robust, there are areas where the codebase currently diverges from the optimal design. We document these here for transparency and future roadmap planning.

### 1. Redis Queue Payload Sizes
**Ideal Design:** The BullMQ queue should only handle lightweight metadata. For screenshots, the Express controller should upload the raw image to a temporary storage bucket or pass a streaming buffer, putting only a pointer (URL/key) into Redis.
**Current Implementation:** The Express controller pushes the entire Base64 encoded screenshot string into the Redis memory list. 
**Impact:** Under massive scale, pushing 2MB-5MB strings into Redis RAM increases memory pressure significantly and slows down the Redis instance.

### 2. Single-Node In-Memory SSE Manager
**Ideal Design:** Real-time Server-Sent Events should be backed by a pub/sub system like Redis Pub/Sub. This allows multiple instances of the Node.js backend to scale horizontally, broadcasting events to users connected to any server node.
**Current Implementation:** The `SSEManager` is a singleton class maintaining an in-memory Map of active connections on the current Node.js process.
**Impact:** If we scale the backend horizontally behind a load balancer, an admin connected to Instance A will not receive a clock-in event processed by Instance B. We are currently bound to vertical scaling for the SSE feature to work perfectly.

### 3. Database Connection Saturation Without PgBouncer
**Ideal Design:** A dedicated connection pooler like PgBouncer should sit in front of the PostgreSQL database to multiplex thousands of incoming Express connections down to a few dozen actual DB connections.
**Current Implementation:** We rely on Prisma's built-in connection pooling and Supabase's managed pooler.
**Impact:** During load tests at the 1.1 million record scale, we observed connection pool exhaustion when B-Tree indexes were intentionally dropped. A more aggressive external pooler would handle these edge cases more gracefully.
