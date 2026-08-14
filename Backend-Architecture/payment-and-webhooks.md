# Payment & Webhooks Architecture

This document outlines the step-by-step payment and subscription architecture implemented in Trace, detailing the flow from a user clicking "Upgrade" to the final database update via the Razorpay Webhook.

## End-to-End Architecture Flowchart

```mermaid
flowchart TD
    classDef fe fill:#3b82f6,color:#fff,stroke:#1d4ed8,stroke-width:2px;
    classDef be fill:#8b5cf6,color:#fff,stroke:#6d28d9,stroke-width:2px;
    classDef gateway fill:#f59e0b,color:#fff,stroke:#b45309,stroke-width:2px;
    classDef db fill:#10b981,color:#fff,stroke:#047857,stroke-width:2px;

    A["User Clicks 'Upgrade'<br/>(Frontend)"]:::fe
    -->|"POST /api/subscription/checkout"| B["subscription.controller.ts -> checkout()"]:::be

    B -->|"Attaches notes: { companyId }"| C["Razorpay API<br/>(Creates Subscription Intent)"]:::gateway

    C -->|"Returns subscriptionId"| D["Opens Razorpay Checkout Modal<br/>(Frontend Popup)"]:::fe

    D --> E["User Pays via UPI/Card"]:::gateway
    E --> F["Razorpay Captures Money & Fires Webhook<br/>(POST /api/subscription/webhook)"]:::gateway

    F --> G{"Verify x-razorpay-signature<br/>(HMAC SHA-256)"}:::be

    G -- "Valid Signature" --> H["Extract companyId & planId from Payload Notes"]:::be

    H --> I["Prisma Updates Company Table<br/>(data: { plan: 'starter' | 'business' })"]:::db
    I --> J["Return 200 OK to Razorpay"]:::be
    I --> K["subscriptionAuth Middleware Unlocks Higher Limits"]:::db
```

## Security & Verification

The most critical part of this flow is the webhook signature verification. When Razorpay's servers send a success notification to our `POST /api/subscription/webhook` endpoint, it does not carry a standard JWT token.

To prevent malicious users from hitting this endpoint to grant themselves free subscriptions, we verify Razorpay's cryptographic signature. We hash the incoming raw body using our shared `RAZORPAY_WEBHOOK_SECRET`. If the generated HMAC SHA-256 hash matches the `x-razorpay-signature` header, we can guarantee the payload genuinely originated from Razorpay.

## Linking Payments to Tenants

When we initially create the subscription intent in step two, we attach the `companyId` inside the `notes` metadata payload sent to Razorpay. 

When Razorpay fires the webhook back to us after a successful payment, they include this exact `notes` object. This clever pattern allows our backend to immediately identify which company in our database just paid, without needing complex lookup tables or session states. We extract the `companyId`, update the `Company.plan` tier in Prisma, and our authorization middlewares instantly unlock the new features for that tenant.
