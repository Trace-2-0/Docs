# Architecture Overview

This document provides a high-level view of how the different components of Trace interact to form a cohesive, scalable system. 

## Full System Architecture

Trace is divided into three primary layers: the Client Layer, the Gateway/API Layer, and the Data/Infrastructure Layer. 

```mermaid
flowchart TD
    classDef client fill:#3b82f6,color:#fff,stroke:#1d4ed8,stroke-width:2px;
    classDef api fill:#8b5cf6,color:#fff,stroke:#6d28d9,stroke-width:2px;
    classDef data fill:#10b981,color:#fff,stroke:#047857,stroke-width:2px;
    classDef external fill:#f59e0b,color:#fff,stroke:#b45309,stroke-width:2px;

    subgraph ClientLayer ["Client Layer"]
        Web["Admin Web Dashboard (Next.js)"]:::client
        Agent["Desktop Agent (Electron)"]:::client
    end

    subgraph APILayer ["API & Processing Layer (Express / Node.js)"]
        Gateway["REST API Gateway & Middlewares"]:::api
        SSE["SSE Manager (Real-Time Stream)"]:::api
        Workers["BullMQ Background Workers"]:::api
        Cron["Cron Jobs (Shift Sweeper)"]:::api
    end

    subgraph DataLayer ["Data & Storage Layer"]
        PG[("PostgreSQL (Prisma / Supabase)")]:::data
        Redis[("Redis (Upstash) - Cache & Queue")]:::data
    end

    subgraph External ["External Services"]
        R2["Cloudflare R2 (Screenshots)"]:::external
        Razorpay["Razorpay (Billing Webhooks)"]:::external
    end

    Web <-->|JWT Auth, REST| Gateway
    Web <-->|Server-Sent Events| SSE
    Agent <-->|Agent Token, REST| Gateway
    
    Gateway <-->|Read/Write| PG
    Gateway -->|Enqueue Image Processing| Redis
    Redis -->|Dequeue Job| Workers
    Workers -->|Save Metadata| PG
    Workers -->|Upload WebP| R2
    
    Cron -->|Detect Stale Shifts| PG
    Cron -->|Trigger Auto-Checkout| SSE
    
    Razorpay -->|Payment Webhook| Gateway
```

## Metrics and Scalability

Our backend is containerized using a multi-stage Docker build (node:20-alpine) and runs on Render. It is designed to handle high concurrency, primarily driven by thousands of desktop agents sending periodic heartbeats and screenshot payloads.

Based on our load testing benchmarks, the system maintains excellent stability even at a scale of over 1.1 million database records. 
* **Throughput:** Supported sustained load tests of 30 Virtual Users generating continuous multi-table SQL aggregations.
* **Latency:** By implementing composite B-Tree indexes and Upstash Redis response caching, we reduced p95 query latency from 50+ seconds down to under 3.5 seconds for complex dashboard statistics.
* **Background Processing:** Pushing image compression (Sharp WebP) to BullMQ asynchronous workers keeps the main event loop free. Our endpoints acknowledge screenshot uploads in less than 5ms, while the actual processing completes in the background.

This decoupled architecture ensures that spikes in agent activity do not degrade the experience for administrators viewing the web dashboard.
