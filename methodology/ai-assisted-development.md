# My Approach to AI-Assisted Development

I build software using AI coding assistants (Claude Code, Gemini). This is a deliberate choice, not a shortcut — and I think it's worth being transparent about.

## How I Work

My development workflow:
1. **I define the architecture** — system design, data models, API contracts, security requirements
2. **I write specs** — detailed requirements for each feature, including edge cases and constraints
3. **AI assists with implementation** — code generation, boilerplate, test scaffolding
4. **I review everything** — every line of generated code gets reviewed for correctness, security, and alignment with the architecture
5. **I debug and iterate** — when things break (they do), I diagnose the root cause and direct fixes

## What This Means for My Code

- **I own the architecture.** The system design, data flow, security model, and integration patterns are mine. AI doesn't make architectural decisions — I do.
- **I own the product decisions.** Why GraphQL over REST? Why Claude Haiku instead of GPT-4? Why MCP instead of pure REST for agent integration? These are my calls, based on experience and trade-off analysis.
- **The code quality reflects both of us.** AI generates code fast. I ensure it's correct, secure, and maintainable. The 168 tests in ConnexusAI exist because I know that AI-generated code needs aggressive testing at the boundaries.

## Why I'm Transparent About This

AI-assisted development is becoming the norm. Pretending otherwise is dishonest and misses the point. The valuable skill isn't "can you type code fast" — it's:

- Can you design systems that work?
- Can you evaluate generated code critically?
- Can you debug when things go wrong?
- Can you make the right architectural trade-offs?
- Can you ship production-quality software?

I can do all of these. The AI makes me faster at the implementation layer. But speed without judgment produces bugs, not products.

## My AI Development Toolkit

| Tool | How I Use It |
|------|-------------|
| Claude Code | Primary coding assistant — implementation, refactoring, test writing |
| Architecture specs | I write detailed specs before any code generation |
| Code review | Every generated file gets manual review |
| Test-driven validation | Automated tests verify AI-generated code meets requirements |
| Git discipline | Small commits, clear messages, feature branches |

## The Meta Insight

I build AI products using AI tools. This gives me a practitioner's perspective on both sides:
- As an **AI product builder**, I understand how LLMs work, where they fail, and how to build guardrails.
- As an **AI tool user**, I understand the developer experience, the trust boundaries, and what makes AI-assisted development productive versus frustrating.

This dual perspective directly shaped aiVelox (designing the tools that AI agents use) and ConnexusAI (building the guardrails that make AI integration safe).
