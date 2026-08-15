# 🏆 Portfolio Comparison: Trace 1.0 vs. Contest Tracker

This document compares **Trace 1.0 (Workforce Monitoring SaaS)** and **Contest Tracker (Developer Platform)** to help present both projects effectively in technical interviews and resume portfolios.

---

## 📊 1. Architectural Overview & Positioning

| Dimension | 🚀 Trace 1.0 (Enterprise B2B SaaS) | 🎯 Contest Tracker (Developer B2C App) |
| :--- | :--- | :--- |
| **Project Type** | Multi-Tenant Enterprise Workforce SaaS | Developer Community & Contest Monitoring App |
| **System Architecture** | Decoupled Architecture (Express REST API + Next.js Frontend + Electron Desktop Agent) | Monolithic Full-Stack Next.js (App Router + Server Actions / Next APIs) |
| **Primary Database** | PostgreSQL via Prisma ORM (Strict Multi-Tenancy) | Supabase (PostgreSQL) + MongoDB Mongoose |
| **Media Storage** | Cloudflare R2 Bucket (Sharp WebP compression + 1-hr pre-signed URLs) | Supabase Storage / External links |
| **Real-Time System** | Custom Server-Sent Events (SSE) with Tenant Isolation | Supabase Realtime / Polling |
| **DevOps & Packaging** | Multi-stage Docker Container (`node:20-alpine` + OpenSSL) + Docker Compose | Vercel Serverless Deployment |

---

## 🏗️ 2. Detailed Technical Comparison

### A. Backend Architecture & Tenant Isolation
* **Trace 1.0:** Uses a dedicated Express Node.js server built for heavy background processing. Every table carries `companyId` for strict row-level multi-tenant data isolation. Includes `jwtAuth` (Admin Dashboard) and `agentAuth` (`x-agent-token` for desktop tracking).
* **Contest Tracker:** Leverages Next.js App Router API endpoints and Supabase Auth. Perfect for consumer-facing developer applications where users interact directly with personal profiles and contest lists.

### B. Background Workers & Automation
* **Trace 1.0:** Includes **2 production background cron engines**:
  1. `shiftSweep.ts`: Auto-closes abandoned employee shifts after 90 minutes of missing heartbeats.
  2. `retentionSweep.ts`: Automated daily data cleanup that purges screenshots older than subscription plan retention (e.g. 7 days for Free/Starter) from **Cloudflare R2** and **Postgres DB**.
* **Contest Tracker:** Utilizes cron jobs and email notifications (Nodemailer) for contest reminders and score tracking.

### C. Monetization & Security
* **Trace 1.0:** Full Razorpay Subscription pipeline with signature-verified Webhooks (`crypto.createHmac('sha256')`), trial auto-expiration guards (30 days), and employee limit middleware (`checkEmployeeLimit`).
* **Contest Tracker:** OAuth authentication (Supabase Auth) with user roles and community contest tracking rules.

### D. Documentation & Standards
* **Trace 1.0:** Fully modularized **Swagger OpenAPI 3.0** documentation interactive UI at `/api/docs`.
* **Contest Tracker:** Documented via Markdown guides and Markdown rules.

---

## 🎯 3. How to Present Both in Interviews & Resume

### 💡 Resume Bullet Points:

#### **Trace 1.0 (Enterprise Multi-Tenant Monitoring SaaS):**
> *"Architected a decoupled multi-tenant Node.js Express backend with PostgreSQL (Prisma ORM) and Cloudflare R2 object storage. Built custom SSE streaming for real-time employee monitoring, automated retention purge cron jobs, dual JWT/Agent token authentication, Razorpay payment webhooks, and containerized the system using multi-stage Docker builds."*

#### **Contest Tracker (Full-Stack Developer Platform):**
> *"Developed a full-stack Next.js developer platform featuring Supabase integration, rich text editing (Tiptap), email notifications, and automated contest deadline tracking."*

---

### 🎙️ Interview Talking Points:

* **When asked about Complex Backend & System Design:** Talk about **Trace 1.0** (Focus on Multi-Tenancy, Cloudflare R2 pre-signed URLs, SSE tenant streaming, Dockerization, and background cron sweepers).
* **When asked about Full-Stack Next.js & User Experience:** Talk about **Contest Tracker** (Focus on Next.js App Router, Supabase Auth, Tiptap rich editing, and UI reactivity).
