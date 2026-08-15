# Trace Overview

Trace is a B2B multi-tenant enterprise workforce monitoring SaaS providing actionable insights into employee productivity with strict data isolation.

## Core Features
<table>
  <tr>
    <td><b>Time Tracking</b><br/>Clock in/out and break management via desktop agent. Handles idle time and disconnects.</td>
    <td><b>Activity Monitoring</b><br/>Captures screenshots and tracks foreground apps to categorize productivity.</td>
  </tr>
  <tr>
    <td><b>Real-Time Dashboard</b><br/>Live employee status updates and screenshots via SSE.</td>
    <td><b>Automated Billing</b><br/>Recurring billing and plan management via Razorpay.</td>
  </tr>
  <tr>
    <td colspan="2"><b>Data Privacy</b><br/>Secure screenshot storage in Cloudflare R2 with automated background purging.</td>
  </tr>
</table>

## Tech Stack
<table>
  <tr>
    <td><b>Backend:</b> Node.js, Express, TypeScript</td>
    <td><b>Database:</b> PostgreSQL (Supabase), Prisma ORM</td>
  </tr>
  <tr>
    <td><b>Queueing & Caching:</b> Upstash Redis, BullMQ</td>
    <td><b>Storage:</b> Cloudflare R2</td>
  </tr>
  <tr>
    <td><b>Frontend:</b> Next.js (App Router)</td>
    <td><b>Desktop Agent:</b> Electron</td>
  </tr>
</table>

## Why These Choices?
* **Express over Next.js API:** Better suited for heavy background jobs (BullMQ), long-lived SSE connections, and cron maintenance.
* **PostgreSQL over NoSQL:** Relational integrity and cascade deletes ensure perfect data cleanup and prevent orphaned records.
