# Authentication & Security Architecture

Trace handles sensitive workforce data, making robust authentication and strict multi-tenant isolation critical. We utilize a dual-authentication architecture to serve both web administrators and desktop agents effectively.

## The Dual-Authentication Model

We separate web frontend requests from desktop client requests because they have fundamentally different lifecycle needs.

| Auth System | Used By | Header Format | Token Type | Lifetime & Revocation |
| :--- | :--- | :--- | :--- | :--- |
| **`jwtAuth`** | React Admin Dashboard | `Authorization: Bearer <token>` | Signed JWT | Expires in 7 Days. Contains `userId`, `Tenant ID`, `role`. |
| **`agentAuth`** | Desktop Agent App | `x-agent-token: <unique-token>` | Database API Key | Persistent. Revocable instantly by Admins. |

### Why not use JWTs for the Desktop Agent?
Desktop agents run quietly in the OS background. Forcing them to log out every 7 days due to an expired JWT would disrupt automated time tracking and frustrate employees. 

Instead, we issue a persistent API key (`x-agent-token`). Our `agentAuth` middleware queries the database to verify the `user.isActive` and `company.isActive` flags on every single API call. This means an admin can terminate an employee's access in zero milliseconds, instantly severing the agent's connection to the backend.

## Multi-Tenant Data Isolation Strategy

Multi-tenancy means multiple companies share the exact same database and server infrastructure. It is our responsibility to ensure neither company can ever see another company's data.

### How Trace Enforces Isolation:

1. **Foreign Key Attachment (`Tenant ID`):**
   Every table in Prisma (`User`, `Team`, `Shift`, `Screenshot`, `AppUsage`) has a required `Tenant ID` foreign key.

2. **Automatic Scope Injection:**
   Because our authentication middlewares (`jwtAuth` and `agentAuth`) extract the `Tenant ID` from the verified token, our route controllers **never trust input from the request body** for tenant identification. They always query using the authenticated scope:

   ```typescript
   // SAFE: Always scoped to the authenticated user's company
   const users = await prisma.user.findMany({
     where: { 
       Tenant ID: req.user.Tenant ID // Guarantees zero data leakage
     }
   });
   ```

## Password Security and Audit Trails

* **Password Security:** User passwords are never stored as plain text. We hash all passwords during registration and login using `bcrypt` with a salt factor of 10.
* **Audit Trails:** We maintain an immutable `AuditLog` database table. Administrative and security events (e.g., `company.registered`, `user.login`, `shift.auto_checkout`) are logged here to ensure compliance and traceability.
