# Trace Overview

Trace is a multi-tenant enterprise workforce monitoring SaaS designed to track employee productivity seamlessly. It provides organizations with actionable insights into time spent, application usage, and overall efficiency, all while maintaining strict data isolation between companies.

## Core Features
At its heart, Trace offers a robust set of tools for workforce management:
* **Time Tracking and Shift Management:** Employees can clock in, take breaks, and clock out using a dedicated desktop agent. The system automatically handles idle time and abrupt disconnects.
* **Activity Monitoring:** The desktop agent captures screenshots at configurable intervals and tracks foreground application usage to categorize time as productive, neutral, or unproductive.
* **Real-Time Dashboard:** Admins and managers can view live employee status updates, including clock-ins, breaks, and recent screenshots, pushed instantly via Server-Sent Events (SSE).
* **Automated Subscription Billing:** Integrated with Razorpay, Trace handles recurring billing, plan tiers, and limits automatically.
* **Data Privacy and Retention:** Screenshots are securely stored in Cloudflare R2 and served via pre-signed URLs. Automated background jobs handle the purging of old data based on the company's subscription plan.

## Business Logic
Trace operates on a B2B model. A company signs up, subscribes to a plan, and invites its managers and employees. The database enforces strict multi-tenancy at the row level, ensuring that no company can ever access another company's data.

The system relies on background processing to keep the user experience smooth. We use automated cron jobs to sweep for abandoned shifts, and a BullMQ queue to handle the heavy lifting of image compression and storage outside the main Node.js event loop.

## Tech Stack
We built Trace using a decoupled, modular approach:
* **Backend:** Node.js with Express and TypeScript. We chose Express for its simplicity and vast ecosystem, making it ideal for a heavy REST API that handles background processing and streaming.
* **Database:** PostgreSQL managed via Prisma ORM. We host the DB on Supabase to leverage their connection pooling and scalability. Prisma provides type safety and easy schema migrations.
* **Queueing and Caching:** Upstash Redis and BullMQ. Redis is used both for caching database query results and acting as the backing store for our asynchronous job queues.
* **Storage:** Cloudflare R2. We selected R2 over AWS S3 because of its zero egress fees, which is critical for a media-heavy application where admins frequently view screenshots.
* **Frontend:** Next.js (App Router) for the admin dashboard.
* **Desktop Agent:** Electron. Electron allows us to use web technologies while accessing native OS APIs for taking screenshots and monitoring active windows.

## Why These Choices?
You might wonder why we chose a decoupled Express backend instead of a monolithic Next.js application. While Next.js is fantastic for web-centric apps, Trace requires robust background job processing (BullMQ), long-lived connections for Server-Sent Events (SSE), and granular cron jobs for system maintenance. A dedicated Express server is far better suited for these sustained, background-heavy workloads than serverless functions.

Similarly, we rely on standard PostgreSQL rather than a NoSQL database because the relational integrity of our data is paramount. Features like cascade deletes ensure that when an admin removes an employee, all their associated shifts, breaks, and screenshots are reliably purged, preventing orphaned data.
