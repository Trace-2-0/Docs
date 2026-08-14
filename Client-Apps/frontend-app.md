# Frontend Application Architecture

The Trace admin dashboard is built as a single-page application experience using Next.js (App Router) and React. It serves as the command center for managers and administrators to view live activity, manage billing, and analyze historical data.

## Core Responsibilities

Unlike a monolithic Next.js application where the frontend and backend are tightly coupled, the Trace frontend is strictly a presentational and client-side logic layer. It interacts with the Express backend entirely through REST APIs and Server-Sent Events (SSE).

### 1. State Management & Data Fetching
The frontend relies heavily on modern data-fetching libraries (like SWR or React Query) to maintain synchronization with the backend. This is particularly important for our Pre-Signed URL caching strategy. The frontend caches image URLs alongside their `expiresAt` timestamps. Re-renders of the dashboard pull instantly from this local cache, only reaching out to the backend to request a new URL when the timestamp has expired.

### 2. Live SSE Integration
The dashboard establishes an ongoing connection to the backend's `/api/sse/stream` endpoint. When an event fires (e.g., an employee clocks in or a new screenshot is processed), the frontend receives the JSON payload over the open connection and dispatches state updates. This allows status badges to turn green and new screenshots to slide into the feed without requiring the user to refresh the page.

### 3. Subscription & Billing Flow
The frontend implements the user-facing side of the Razorpay integration. When a user clicks "Upgrade", the Next.js app requests a subscription intent ID from the backend, injects the Razorpay checkout script, and opens the native payment modal over the UI. Upon completion, it waits for the backend to verify the webhook before unlocking premium UI components.

## UI/UX Considerations
Because Trace is an enterprise SaaS, the UI emphasizes density, readability, and performance. Large data tables for shift history and high-quality grids for screenshots require careful virtualization and lazy loading to prevent the browser's DOM from becoming sluggish.
