# Real-Time Server-Sent Events (SSE) Architecture

A core feature of the Trace admin dashboard is live visibility into employee activity. We power this real-time monitoring system using Server-Sent Events (SSE) rather than WebSockets or HTTP Short Polling.

## Why SSE over WebSockets?

When building real-time dashboards, WebSockets are often the default choice. However, WebSockets establish a bidirectional (`<->`) communication channel. In Trace, the flow of live data is strictly unidirectional: the server pushes updates (clock-ins, new screenshots) to the client (`->`). 

SSE is built on top of standard HTTP (`text/event-stream`), making it significantly lighter on overhead, easier to proxy through Nginx or Cloudflare, and benefits from the browser's native, built-in auto-reconnection logic.

## How SSE Works Under the Hood

Unlike standard REST endpoints that send a JSON response and immediately close the socket, our SSE controller keeps the TCP connection open indefinitely.

We enforce this via specific HTTP headers in `src/lib/sse.ts`:
```typescript
res.writeHead(200, {
  'Content-Type': 'text/event-stream',  // Tells the browser this is an endless stream
  'Cache-Control': 'no-cache',          // Prevents CDNs from caching the stream
  'Connection': 'keep-alive',           // Keeps the TCP socket open
  'X-Accel-Buffering': 'no',            // Disables proxy buffering for instant delivery
});
```

## Multi-Tenant Memory Architecture (`SSEManager`)

We maintain a singleton `SSEManager` class to handle connections. Because Trace is multi-tenant, it is vital that events from Company A are never broadcasted to Company B's dashboard.

The manager keeps an in-memory Map where the key is the `companyId` and the value is an array of active response connections for that company's admins. When the system processes a new screenshot, it calls `sseManager.broadcast('companyA', 'screenshot.captured', data)`. The manager loops only through Company A's sockets and pushes the plain-text event.

### Keep-Alive and Memory Management
Cloud load balancers will aggressively drop idle HTTP connections. To keep our SSE streams alive, the `SSEManager` runs an interval that pushes a lightweight comment (`:keepalive\n\n`) to all connected clients every 30 seconds.

Furthermore, we carefully listen to the `close` event on the Express response object. If an admin closes their browser tab, the manager instantly removes their socket from the Map, preventing memory leaks on the Node.js server.
