# ConnexusAI — Architecture

## System Overview

```mermaid
graph TD
    subgraph Input Channels
        WebApp[Web App<br/>Quick Capture UI]
        TGBot[Telegram Bot<br/>Mobile capture]
    end

    subgraph AI Processing Pipeline
        Sanitize[Input Sanitization<br/>Control chars, length limits]
        LLM[Claude Haiku API<br/>Entity extraction]
        Validate[Output Validation<br/>Hand-written validators]
        Resolve[Contact Resolution<br/>Server-side ID lookup]
    end

    subgraph API Layer
        GQL[GraphQL API<br/>Yoga Server + Apollo Client]
        Auth[Better Auth<br/>OAuth 2.0 - Google, GitHub]
        RBAC[RBAC Engine<br/>4 roles per group]
    end

    subgraph Data Layer
        DB[(Turso<br/>Edge SQLite)]
        R2[Cloudflare R2<br/>File storage]
        Encrypt[AES-256<br/>Column encryption]
    end

    subgraph Integrations
        GCal[Google Calendar<br/>Two-way sync]
        Cron[Daily Cron<br/>Scoring recalculation]
    end

    WebApp --> Sanitize
    TGBot --> Sanitize
    Sanitize --> LLM
    LLM --> Validate
    Validate --> Resolve
    Resolve --> GQL
    GQL --> Auth
    Auth --> RBAC
    GQL --> DB
    GQL --> R2
    DB --> Encrypt
    DB --> Cron
    GQL --> GCal
```

## Data Model

```mermaid
erDiagram
    USER ||--o{ MEMBERSHIP : has
    USER ||--o{ CONTACT_OVERLAY : personalizes
    USER ||--o{ REMINDER : creates
    USER ||--o{ TELEGRAM_LINK : connects
    GROUP ||--o{ MEMBERSHIP : contains
    GROUP ||--o{ CONTACT : shares
    CONTACT ||--o{ CONTACT_OVERLAY : "viewed through"
    CONTACT ||--o{ INTERACTION : has
    CONTACT ||--o{ LIFE_EVENT : has

    USER {
        string id PK
        string email
        string name
        string provider "google | github"
    }

    CONTACT {
        string id PK
        string firstName
        string lastName
        string email "optional"
        string phone "optional"
        string company "optional"
        string jobTitle "optional"
    }

    CONTACT_OVERLAY {
        string userId FK
        string contactId FK
        int circle "1-5, inner to outer"
        float score "0-100, time-decayed"
        text privateNotes "AES-256 encrypted"
    }

    INTERACTION {
        string id PK
        string contactId FK
        string userId FK
        string type "meeting|call|email|message|social|other"
        string title
        text notes "AES-256 encrypted"
        datetime occurredAt
    }

    REMINDER {
        string id PK
        string userId FK
        string contactId FK "optional"
        string title
        datetime dueAt
        boolean isRecurring
        string recurrenceRule "RRULE format"
    }

    GROUP {
        string id PK
        string name
        string type "family | organization"
    }

    MEMBERSHIP {
        string userId FK
        string groupId FK
        string role "super_admin|admin|super_user|user"
    }
```

**Key design decision: Shared contacts + personal overlays.** Contacts are shared within a group (family/org), but each user has their own overlay — private notes, circle placement, and relationship score. This eliminates duplicate contacts while preserving individual perspective.

## AI Processing Pipeline

```mermaid
sequenceDiagram
    participant U as User
    participant S as Sanitizer
    participant C as Claude Haiku
    participant V as Validator
    participant R as Resolver
    participant DB as Database

    U->>S: Free text input
    S->>S: Strip control chars
    S->>S: Enforce length limits
    S->>C: Sanitized text + contact names + current datetime
    C->>C: Parse intent (reminder | interaction | contact | other)
    C->>V: JSON response
    V->>V: Type check all fields
    V->>V: Validate dates (must be future for reminders)
    V->>V: Validate interaction types
    V->>V: Check string lengths
    alt Valid
        V->>R: Validated entities
        R->>DB: Fuzzy match contact name → ID
        R->>DB: Store entities with resolved IDs
        R->>U: Success + structured preview
    else Invalid
        V->>U: "Could you rephrase that?"
    end
```

**Security boundary:** The LLM receives contact *names* for fuzzy matching but never sees database IDs. Contact ID resolution happens server-side after validation. This prevents prompt injection from referencing arbitrary contacts.

## Relationship Scoring

```mermaid
graph LR
    subgraph Interaction History
        I1[Meeting<br/>weight: 15]
        I2[Call<br/>weight: 10]
        I3[Social<br/>weight: 8]
        I4[Email<br/>weight: 6]
        I5[Message<br/>weight: 4]
    end

    subgraph Time Decay
        D[Exponential decay<br/>30-day half-life<br/>score = weight × e^(-λt)]
    end

    subgraph Output
        S[Aggregate Score<br/>capped at 100]
        C[Circle Placement<br/>1-5 rings]
        R[Reach-out Suggestions<br/>declining scores]
    end

    I1 --> D
    I2 --> D
    I3 --> D
    I4 --> D
    I5 --> D
    D --> S
    S --> C
    S --> R
```

Recalculated daily via cron job. 24-hour staleness is acceptable — relationship health changes slowly.

## Multi-Tenancy & RBAC

```mermaid
graph TD
    subgraph Roles
        SA[super_admin<br/>Full control]
        A[admin<br/>Manage members + contacts]
        SU[super_user<br/>Add/edit shared contacts]
        U[user<br/>Read shared, edit own overlay]
    end

    subgraph Permissions
        SA --> MC[Manage contacts]
        SA --> MM[Manage members]
        SA --> MG[Manage group settings]
        A --> MC
        A --> MM
        SU --> MC
        U --> RO[Read shared contacts]
        U --> EO[Edit own overlay/notes]
    end
```

## Encryption Strategy

| Data | Encryption | Rationale |
|------|-----------|-----------|
| Name, email, phone, company | None | Needed for server-side search and display |
| Private notes | AES-256 column-level | Sensitive personal observations |
| Interaction notes | AES-256 column-level | May contain confidential details |
| API keys, tokens | Environment variables | Never stored in database |

## Technology Choices

| Decision | Choice | Why Not the Alternative |
|----------|--------|----------------------|
| API style | GraphQL | Relationship data is graph-shaped. REST would require N+1 calls or complex includes. |
| Database | Turso (edge SQLite) | Global edge latency. PostgreSQL was overkill for the data volume. |
| LLM | Claude Haiku | Fastest and cheapest for structured extraction. GPT-4 is slower and more expensive for this use case. |
| Auth | Better Auth + OAuth | Drop-in, supports Google/GitHub. No password management needed. |
| State | Apollo Client + Zustand | Apollo for server cache (GraphQL), Zustand for local UI state. Clean separation. |
| Bot | Telegram | Most accessible for mobile quick capture. WhatsApp Business API has higher friction to set up. |
