# Claritas — Architecture

## System Overview

```mermaid
graph TD
    subgraph User Interface
        Dash[Dashboard<br/>Custom categories & items]
        Detail[Item Detail<br/>Comments, AI analysis]
        Config[Settings<br/>Provider config, API keys]
    end

    subgraph AI Abstraction Layer
        Interface[Provider Interface<br/>Uniform API]
        Gemini[Gemini]
        Groq[Groq]
        Mistral[Mistral]
    end

    subgraph Data Layer
        Prisma[Prisma ORM<br/>Type-safe queries]
        DB[(SQLite)]
        Vault[Encrypted Key Store<br/>API keys at rest]
    end

    Dash --> Detail
    Detail --> Interface
    Config --> Vault
    Vault --> Interface
    Interface --> Gemini
    Interface --> Groq
    Interface --> Mistral
    Detail --> Prisma
    Config --> Prisma
    Prisma --> DB
```

## Due Diligence Framework

```mermaid
graph TD
    subgraph 8 Assessment Categories
        C1[Infrastructure<br/>Hosting, DR, network, monitoring]
        C2[Security<br/>ISO27001, access control, incidents]
        C3[Applications<br/>Core systems, tech stack, debt]
        C4[DevOps<br/>CI/CD, testing, deployment]
        C5[Architecture<br/>Integration, APIs, scalability]
        C6[Data<br/>Quality, governance, migration]
        C7[Team<br/>Skills, retention, key person risk]
        C8[Compliance<br/>Regulatory, licensing, audits]
    end

    subgraph Per Item
        Status[Status tracking<br/>Not started → In progress → Complete]
        Risk[Risk scoring<br/>Low → Medium → High → Critical]
        Comments[Comments & clarifications]
        AI[AI Analysis<br/>Completeness, red flags, follow-ups]
    end

    C1 --> Status
    C2 --> Status
    C3 --> Status
    C4 --> Status
    C5 --> Status
    C6 --> Status
    C7 --> Status
    C8 --> Status
    Status --> Risk
    Status --> Comments
    Status --> AI
```

The category and item structure is fully configurable per project. The default templates are informed by real IT due diligence I've led at Cigna Insurance (TPA acquisition) and Zurich Insurance (retail insurance acquisition).

## Multi-Provider AI Abstraction

```mermaid
sequenceDiagram
    participant User
    participant App as Application Code
    participant Abs as Provider Abstraction
    participant Vault as Encrypted Key Store
    participant LLM as Active LLM Provider

    User->>App: Request AI analysis
    App->>Abs: analyze(itemContext)
    Abs->>Vault: Get API key for active provider
    Vault->>Vault: Decrypt key
    Vault->>Abs: Decrypted API key
    Abs->>LLM: API call with context
    LLM->>Abs: Analysis response
    Abs->>App: Normalized AnalysisResult
    App->>User: Display analysis
```

**Key pattern:** Application code calls a uniform interface. Provider selection is a configuration concern, not a code concern. Swapping from Gemini to Groq requires changing one setting, not modifying application logic.

```mermaid
graph TD
    subgraph Application Code
        A[analyze request]
    end

    subgraph Abstraction Layer
        I[Provider Interface<br/>analyze, summarize]
    end

    subgraph Providers
        G[Gemini Adapter<br/>Best reasoning]
        GR[Groq Adapter<br/>Fastest, cheapest]
        M[Mistral Adapter<br/>European data residency]
    end

    A --> I
    I --> G
    I --> GR
    I --> M
```

**Why three providers?**
| Provider | Strength | When to Use |
|----------|---------|-------------|
| Gemini | Complex reasoning | Deep analysis of architectural risks |
| Groq | Speed and cost | Bulk analysis of straightforward items |
| Mistral | EU data residency | When target company data must stay in EU |

## Data Model

```mermaid
erDiagram
    CATEGORY ||--o{ DD_ITEM : contains
    DD_ITEM ||--o{ COMMENT : has
    DD_ITEM ||--o{ AI_ANALYSIS : has
    AI_PROVIDER ||--o{ AI_ANALYSIS : generates

    CATEGORY {
        string id PK
        string name "user-defined"
        int sortOrder
    }

    DD_ITEM {
        string id PK
        string categoryId FK
        string title
        text description
        string status "not_started|in_progress|complete"
        string risk "low|medium|high|critical"
        text response
        datetime updatedAt
    }

    COMMENT {
        string id PK
        string itemId FK
        text content
        string author
        datetime createdAt
    }

    AI_ANALYSIS {
        string id PK
        string itemId FK
        string providerId FK
        text analysis
        text suggestedFollowUps
        text redFlags
        datetime generatedAt
    }

    AI_PROVIDER {
        string id PK
        string name "gemini|groq|mistral"
        text encryptedApiKey "AES encrypted"
        boolean isActive
    }
```

## Security

| Concern | Approach |
|---------|---------|
| API keys at rest | Encrypted in database — never stored as plaintext |
| API keys in transit | HTTPS only, keys loaded into memory only when needed |
| User data | SQLite local-first — no cloud dependency for sensitive DD data |
| LLM data exposure | Only DD item context sent to LLM — no full database exports |

## Technology Choices

| Decision | Choice | Why |
|----------|--------|-----|
| Database | SQLite via Prisma | Local-first. DD data is sensitive — no cloud DB dependency needed. |
| ORM | Prisma 7 | Type-safe, migrations, introspection. Best DX for SQLite. |
| UI | shadcn/ui (51 components) | Consistent, accessible. Faster than building from scratch. |
| AI | Multi-provider abstraction | Provider flexibility for cost, speed, and data residency requirements. |
| Encryption | Application-level | API keys encrypted before database write. Decrypted on read. |
