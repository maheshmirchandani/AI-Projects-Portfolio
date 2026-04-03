# aiVelox — Architecture

## System Overview

```mermaid
graph TD
    subgraph Human Interface
        PM[Project Manager<br/>Browser - Full access]
        Viewer[Viewers<br/>Browser - Read only]
    end

    subgraph AI Agent Interface
        MCP[MCP Server<br/>stdio - 5 tools]
        WH[Webhook API<br/>REST - API key auth]
        CLI[LaneKit CLI<br/>Terminal commands]
    end

    subgraph Core Platform
        NX[Next.js 16<br/>App Router]
        MW[Auth Middleware<br/>Session + API key validation]
        RBAC[RBAC Engine<br/>4 roles]
        AL[Audit Logger<br/>All mutations tracked]
    end

    subgraph Data
        DB[(Turso Database<br/>7 tables)]
    end

    PM --> NX
    Viewer --> NX
    MCP --> NX
    WH --> MW
    CLI --> WH
    MW --> RBAC
    NX --> RBAC
    RBAC --> DB
    NX --> AL
    WH --> AL
    AL --> DB
```

## MCP Integration Architecture

```mermaid
sequenceDiagram
    participant Agent as AI Agent<br/>(Claude Code, Gemini)
    participant MCP as MCP Server<br/>(stdio)
    participant Auth as Auth Layer
    participant RBAC as RBAC Check
    participant Logic as Business Logic
    participant Audit as Audit Logger
    participant DB as Turso DB

    Agent->>MCP: Tool call: aivelox_update_status
    MCP->>Auth: Validate agent identity
    Auth->>RBAC: Check role permissions
    alt Authorized
        RBAC->>Logic: Execute operation
        Logic->>DB: Update task
        Logic->>Audit: Log action (who, what, when)
        Audit->>DB: Store audit record
        Logic->>MCP: Success response
        MCP->>Agent: Result
    else Unauthorized
        RBAC->>MCP: Permission denied
        MCP->>Agent: Error: insufficient permissions
    end
```

## MCP Tools

```mermaid
graph LR
    subgraph MCP Server - 5 Tools
        T1[aivelox_get_tasks<br/>Read tasks by project/column]
        T2[aivelox_add_task<br/>Create in backlog or rework ONLY]
        T3["aivelox_update_status<br/>working | waiting | done"]
        T4[aivelox_move_to_rework<br/>Flag with reason]
        T5[aivelox_add_note<br/>Comment on any task]
    end

    subgraph Trust Boundary
        CAN[Agent CAN<br/>Read, update status, add notes,<br/>create in backlog/rework]
        CANNOT[Agent CANNOT<br/>Approve, deploy, delete,<br/>modify other agents' work]
    end

    T1 --> CAN
    T2 --> CAN
    T3 --> CAN
    T4 --> CAN
    T5 --> CAN
```

**Design principle:** AI agents are powerful executors but should not make judgment calls about production readiness. The PM reviews and promotes. The agent reports and suggests.

## Dual Integration: MCP + Webhooks

```mermaid
graph TD
    subgraph Entry Points
        MCP[MCP Tool Call<br/>Native AI assistants]
        WH[Webhook POST<br/>Any HTTP client]
    end

    subgraph Shared Pipeline
        Auth[Authentication<br/>Session OR API key]
        RBAC[RBAC Check<br/>Role-based permissions]
        BL[Business Logic<br/>Task CRUD operations]
        AL[Audit Log<br/>Every mutation recorded]
    end

    subgraph Storage
        DB[(Turso)]
    end

    MCP --> Auth
    WH --> Auth
    Auth --> RBAC
    RBAC --> BL
    BL --> AL
    AL --> DB
    BL --> DB
```

Both entry points converge at the auth layer. One codebase, one audit trail, consistent security.

## Kanban Workflow (9 Columns)

```mermaid
graph LR
    B[Backlog<br/>AI can add] --> S[Sprint<br/>PM prioritizes]
    S --> ID[In Development<br/>AI working]
    ID --> D[Developed<br/>AI done]
    D --> R[Rework<br/>AI can add bugs]
    R --> ID
    D --> AB[Approved<br/>Branch]
    AB --> AP[Approved<br/>Production]
    AP --> DEP[Deployed<br/>Live]
    DEP --> AR[Archive<br/>Complete]
```

**Column access by role:**

| Column | PM | Agent | Viewer |
|--------|-----|-------|--------|
| Backlog | Read/Write | Add only | Read |
| Sprint → Developed | Read/Write | Status updates | Read |
| Rework | Read/Write | Add + move to | Read |
| Approved → Archive | Read/Write | No access | Read |

## Data Model

```mermaid
erDiagram
    USER ||--o{ SESSION : has
    USER ||--o{ OTP_CODE : receives
    USER ||--o{ AUDIT_LOG : generates
    PROJECT ||--o{ TASK : contains
    USER ||--o{ PROJECT : manages

    USER {
        string id PK
        string email
        string name
        string role "super_admin|pm|viewer|agent"
        string apiKeyHash "SHA-256, agents only"
        int failedAttempts
        datetime lockedUntil
    }

    PROJECT {
        string id PK
        string name
        string description
        string ownerId FK
    }

    TASK {
        string id PK
        string projectId FK
        string title
        string description
        string column "9 possible values"
        string priority "low|medium|high"
        string aiStatus "working|waiting|done|null"
        string source "manual|ai-suggested"
        string tags "JSON array"
        int position "ordering within column"
        datetime createdAt
        datetime updatedAt
    }

    AUDIT_LOG {
        string id PK
        string action
        string entityType
        string entityId
        string userId FK
        text details "JSON - before/after state"
        datetime timestamp
    }

    SESSION {
        string id PK
        string userId FK
        datetime expiresAt "30 days"
    }
```

## Authentication Architecture

```mermaid
graph TD
    subgraph Human Auth
        Login[Email Input] --> OTP[Generate 6-digit OTP]
        OTP --> Email[Send via Nodemailer]
        Email --> Verify[Verify OTP<br/>10-min expiry]
        Verify --> Session[Create Session<br/>httpOnly cookie, 30 days]
    end

    subgraph Agent Auth
        APIKey[API Key in x-api-key header] --> Hash[SHA-256 hash]
        Hash --> Lookup[Match against stored hash]
        Lookup --> AgentID[Identify agent + role]
    end

    subgraph Protection
        BF[Brute Force<br/>5 attempts → 30-min lockout]
        RL[Rate Limiting<br/>Per endpoint]
    end

    Verify --> BF
    APIKey --> RL
```

**Why SHA-256 for API keys (not bcrypt)?** API keys are already high-entropy random strings. bcrypt's salting and cost factor protect weak passwords — unnecessary for 256-bit keys. SHA-256 is faster and sufficient.

## Technology Choices

| Decision | Choice | Why Not the Alternative |
|----------|--------|----------------------|
| Agent protocol | MCP (primary) + Webhooks (fallback) | MCP is self-describing — agents discover tools automatically. Webhooks for non-MCP agents. |
| Database | Turso (edge SQLite) | Fast reads for dashboard. PostgreSQL unnecessary for task management data volume. |
| Drag & drop | @dnd-kit | Accessible, performant. react-beautiful-dnd is deprecated. |
| Auth (humans) | OTP via email | No passwords to manage. Simple, secure, friction-appropriate. |
| Auth (agents) | API keys (SHA-256 hashed) | Stateless, no session management needed for machine clients. |
| Audit storage | Same database | Separate audit DB would add complexity without benefit at this scale. |
| MCP transport | stdio | Standard for local AI assistants. HTTP transport planned for remote agents. |
