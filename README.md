# AI Projects Portfolio

Architecture, design decisions, and code patterns from my AI projects. Source code is in private repositories — this repo showcases the thinking behind the systems.

---

## Projects

### 1. ConnexusAI — AI-Powered Relationship Manager
[Full writeup](./connexusai/README.md) | [Architecture](./connexusai/architecture.md) | [Code Patterns](./connexusai/snippets/)

An AI-powered platform for managing professional and personal relationships. Users capture interactions via natural language — in-app or through a Telegram bot — and Claude AI parses free text into structured data.

**Key AI capabilities:**
- Claude Haiku integration for natural language entity extraction
- Prompt injection defense with input sanitization and output validation
- Fuzzy contact matching against user's contact list
- Multi-channel AI capture (web app + Telegram bot)

**Stack:** Next.js 16, GraphQL (Yoga + Apollo), Claude Haiku API, Turso, Telegram Bot API, Vitest (168 tests)

---

### 2. aiVelox — AI Agent Coordination Platform
[Full writeup](./aivelox/README.md) | [Architecture](./aivelox/architecture.md) | [Code Patterns](./aivelox/snippets/)

A project management platform built for human-AI collaboration. AI agents connect via MCP protocol or webhook APIs to manage their own task lifecycle — picking up tasks, reporting status, adding notes, and flagging issues.

**Key AI capabilities:**
- Native MCP server with 5 exposed tools for AI agent self-service
- Webhook API with SHA-256 API key authentication for external agents
- Agent role in RBAC system (restricted to backlog/rework columns)
- Real-time AI status tracking with visual indicators

**Stack:** Next.js 16, Turso, MCP protocol (stdio), TypeScript, @dnd-kit

---

### 3. Claritas — AI-Assisted Due Diligence
[Full writeup](./claritas/README.md) | [Architecture](./claritas/architecture.md) | [Code Patterns](./claritas/snippets/)

A domain-specific tool for IT due diligence in M&A transactions. Tracks 57+ assessment items across 8 categories with multi-provider AI analysis.

**Key AI capabilities:**
- Multi-provider AI abstraction (Gemini, Groq, Mistral) with encrypted API key storage
- Provider-agnostic interface — swap LLM providers without changing application code
- AI-assisted response analysis and summary generation

**Stack:** Next.js 16, Prisma, SQLite, multi-provider AI, shadcn/ui

---

## Why This Portfolio Exists

I'm a technology leader (25 years CIO/CTO) who made a deliberate pivot into building AI systems. These projects represent that transition — from managing technology to engineering it.

Each project directory contains:
- **README.md** — Problem, approach, outcome, and what I learned
- **architecture.md** — System design with Mermaid diagrams
- **decisions/** — Architecture Decision Records (ADRs) explaining key trade-offs
- **snippets/** — Selected code patterns that demonstrate AI engineering thinking

---

## About Me

**Mahesh Mirchandani** — Dubai, UAE
- [GitHub](https://github.com/maheshmirchandani) | [LinkedIn](https://www.linkedin.com/in/maheshmirchandani/)
- Currently: Co-founder & CITO at Cumulus Healthcare | Fractional CTO
- Previously: CIO at Cigna Insurance MEA | Group CTO at TruDoc Healthcare | CIO at Zurich Insurance ME
