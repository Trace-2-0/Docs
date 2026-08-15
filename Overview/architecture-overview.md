# Architecture Overview

This document provides a high-level view of how the different components of Trace interact to form a cohesive, scalable system. 

## Full System Architecture

Trace is divided into three primary layers: the Client Layer, the Gateway/API Layer, and the Data/Infrastructure Layer. 

```mermaid
flowchart TD
    %% Styling to make it bold and big
    classDef default fill:#f9f9f9,stroke:#333,stroke-width:2px,font-size:16px,font-weight:bold;
    classDef api fill:#e0e7ff,stroke:#4f46e5,stroke-width:3px;
    classDef storage fill:#dcfce7,stroke:#16a34a,stroke-width:3px;
    classDef client fill:#fef3c7,stroke:#d97706,stroke-width:3px;
    classDef queue fill:#ffe4e6,stroke:#e11d48,stroke-width:3px;

    %% Components
    Agent[Desktop Agent]:::client
    Admin[Web Dashboard]:::client
    
    API[Node.js API Server]:::api
    
    Queue[(Redis Queue)]:::queue
    Worker[Background Workers]:::queue
    
    DB[(PostgreSQL)]:::storage
    Storage[(Cloudflare R2)]:::storage

    %% Interactions
    Agent -- "Heartbeats & Screenshots" --> API
    Admin -- "Real-time updates (SSE)" --> API
    
    API -- "Read/Write Data" --> DB
    API -- "Enqueue Images" --> Queue
    
    Queue -- "Process Image" --> Worker
    Worker -- "Save File URL" --> DB
    Worker -- "Upload WebP" --> Storage
```

## Metrics and Scalability

Our backend is containerized using a multi-stage Docker build (node:20-alpine) and runs on Render. It is designed to handle high concurrency, primarily driven by thousands of desktop agents sending periodic heartbeats and screenshot payloads.

Based on our load testing benchmarks, the system maintains excellent stability even at a scale of over 1.1 million database records. 
* **Throughput:** Supported sustained load tests of 30 Virtual Users generating continuous multi-table SQL aggregations.
* **Latency:** By implementing composite B-Tree indexes and Upstash Redis response caching, we reduced p95 query latency from 50+ seconds down to under 3.5 seconds for complex dashboard statistics.
* **Background Processing:** Pushing image compression (Sharp WebP) to BullMQ asynchronous workers keeps the main event loop free. Our endpoints acknowledge screenshot uploads in less than 5ms, while the actual processing completes in the background.

This decoupled architecture ensures that spikes in agent activity do not degrade the experience for administrators viewing the web dashboard.
