# Trace Universal Documentation

Welcome to the central documentation repository for **Trace**, an enterprise-grade multi-tenant workforce monitoring SaaS.

This repository serves as the single source of truth for the entire Trace system, covering the architectural decisions and low-level designs for the Backend, Frontend, and Desktop Agent.

## 📚 Table of Contents

### Overview
* [Trace Overview](./Overview/trace-overview.md) - Features, tech stack, and business logic.
* [Architecture Overview](./Overview/architecture-overview.md) - Full system diagram and metrics.

### System Design
* [High-Level Design (HLD)](./System-Design/hld-and-principles.md) - Core engineering principles.
* [Low-Level Design (LLD)](./System-Design/lld-and-gaps.md) - Shift engine logic and technical debt.
* [Database ERD](./System-Design/database-erd.md) - Relational models and schema.

### Backend Architecture
* [Authentication & Security](./Backend-Architecture/auth-and-security.md) - Multi-tenancy and JWT/Agent security.
* [Background Jobs & Queuing](./Backend-Architecture/background-jobs.md) - BullMQ queues and automated Cron sweeps.
* [Real-Time Events (SSE)](./Backend-Architecture/real-time-events.md) - Server-Sent Events architecture.
* [Storage & Media](./Backend-Architecture/storage-and-media.md) - Cloudflare R2, image compression, and Pre-Signed URLs.
* [Payment & Webhooks](./Backend-Architecture/payment-and-webhooks.md) - Razorpay subscription flow.

### Load Testing
* [Performance Benchmarks](./Load-Testing/benchmark.md) - 1.1 Million DB scale k6 testing logs.

### Client Applications
* [Desktop Agent Architecture](./Client-Apps/desktop-agent.md) - Electron background tracking.
* [Frontend App Architecture](./Client-Apps/frontend-app.md) - Next.js dashboard UI.
