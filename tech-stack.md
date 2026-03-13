# Klayrai — Tech Stack

## Core Framework

| Layer | Technology | Version | Why |
|-------|-----------|---------|-----|
| Framework | Next.js App Router | 14.x | RSC, Server Actions, Edge Middleware, native Cron |
| Language | TypeScript | 5.x strict | End-to-end type safety, fewer runtime bugs |
| Runtime | Node.js | 20.x LTS | Vercel native, stable long-term support |

## Frontend UI

| Layer | Technology | Why |
|-------|-----------|-----|
| Styling | Tailwind CSS v3 | Constraint-based system, zero raw CSS needed |
| Component library | shadcn/ui | Production-ready, copy-paste, Radix-based, fully customizable |
| Primitives | Radix UI | Accessible, unstyled, battle-tested base components |
| Icons | Lucide React | Consistent, clean, tree-shakeable |
| Charts | Recharts (via shadcn/ui charts) | React-native, simple API, no external dependencies |
| Animations | Framer Motion | Micro-interactions on landing page hero and transitions |
| Fonts | Geist Sans + Geist Mono (Vercel) | Clean, modern, fast loading, optimized for Next.js |

## State Management

| Layer | Technology | Why |
|-------|-----------|-----|
| Client UI state | Zustand v4 | Simple, minimal boilerplate, no provider wrapping |
| Server state | React Server Components | Zero client JS for data fetching, streaming by default |
| Client async state | TanStack Query v5 | Handles tRPC client-side calls, caching, background refetch |

## Backend and API

| Layer | Technology | Why |
|-------|-----------|-----|
| API layer | tRPC v11 | Type-safe E2E, no REST boilerplate, works natively with Next.js |
| Mutations | Next.js Server Actions | Form submissions, simple data mutations without API routes |
| Input validation | Zod v3 | Schema validation on every input — server and client |
| Form state | React Hook Form v7 | Integrates with Zod resolver, handles complex form state cleanly |

## Database

| Layer | Technology | Why |
|-------|-----------|-----|
| Database | Supabase Postgres | Managed Postgres, built-in connection pooling via Supavisor, dashboard and SQL editor |
| ORM | Prisma v5 | Type-safe queries, schema-first migrations, easy to refactor |
| Connection pool | Supabase Supavisor | Handles serverless function cold starts safely |
| V2: Vector store | PgVector (Supabase extension) | RAG embeddings without a separate vector database |

## Authentication

| Layer | Technology | Why |
|-------|-----------|-----|
| Auth provider | Firebase Auth | Handles sessions, OAuth, ID tokens, Edge-compatible via Firebase Admin SDK |
| Route protection | Firebase Auth middleware | Protects (app) routes by verifying ID tokens server-side |
| User sync | First-request provisioning | Creates User record in Postgres on first authenticated request if not exists |

## MCP and AI Layer

| Layer | Technology | Why |
|-------|-----------|-----|
| Meta data source | klayrai-meta-mcp (fork brijr/meta-mcp) | 25 ready-made tools, v23.0 pinned, Claude-native MCP format |
| Agent orchestration | Anthropic Claude API + Claude Code Skills | Diagnostic and report agents with structured tool use |
| Diagnostic model | claude-opus-4-5 | Best structured reasoning for diagnostic analysis |
| Report model | claude-sonnet-4-5 | Faster and cheaper for report generation volume |
| V2 orchestration | LangGraph | Multi-agent graph workflows (plan for, do not build in v1) |
| V2 embeddings | text-embedding-3-small + PgVector | RAG over historical campaign data |

## External Services

| Service | SDK | Why |
|---------|-----|-----|
| Stripe | stripe npm + @stripe/stripe-js | Billing, subscriptions, PCI compliance |
| Resend | resend npm | Transactional emails with React Email templates |
| UploadThing | uploadthing + @uploadthing/react | File uploads with CDN, type validation, size limits (V2) |

## Security

| Layer | Technology | Configuration |
|-------|-----------|---------------|
| CAPTCHA | Cloudflare Turnstile | Applied to auth sign-up form and Book Demo form |
| Rate limiting | @upstash/ratelimit + Upstash Redis | Applied to all mutation routes and Server Actions |
| CORS | Next.js middleware | Restricted to klayrai.com and app.klayrai.com |
| Row Level Security | Postgres native RLS | All multi-tenant tables scoped to workspaceId |
| Secrets | Vercel Env Manager | Never hardcoded, never committed to version control |

## Monitoring

| Tool | Purpose | Day |
|------|---------|-----|
| Sentry | Client and server error tracking with stack traces | Day 1 |
| PostHog | Product analytics, event tracking, user funnels, A/B tests | Day 1 |
| Vercel Analytics | Core Web Vitals, page performance | Day 1 (zero config) |
| Vercel Speed Insights | Real user monitoring | Day 1 (zero config) |
| Lighthouse CI | Automated performance score on every release | Pre-launch |

## DevOps

| Layer | Technology | Why |
|-------|-----------|-----|
| App hosting | Vercel | Push-to-deploy, preview URLs on every PR, Cron jobs |
| MCP server hosting | Render.com | Always-on Node.js process, deploy from GitHub |
| DNS and CDN | Cloudflare | Free CDN, DDoS, Turnstile CAPTCHA |
| Secrets management | Vercel Env Manager | Production secrets managed securely |

## Explicitly NOT Used and Why

| Technology | Why Not |
|-----------|---------|
| Redux or Context API chains | Zustand handles it cleanly with no boilerplate |
| Custom REST API routes | tRPC is faster, type-safe, and zero config |
| Custom auth system | Firebase Auth handles everything including Google OAuth |
| Raw CSS | Tailwind covers 100% of what we need |
| Raw SQL | Prisma covers 100% of what we need |
| Direct Meta API calls from Next.js | klayrai-meta-mcp handles all Meta communication |
| Custom payment system | Stripe handles PCI compliance, subscriptions, disputes |
| Custom search engine | Postgres FTS in v1, Typesense in v2 |
| WebSockets from scratch | Not needed in v1, Supabase Realtime in v2 if needed |
| LangGraph in v1 | Single Claude agent with Skills is sufficient for MVP |
| RAG or VectorDB in v1 | Rules engine and structured context is sufficient for MVP |
| Manual deployments | Vercel automates everything, manual = human error |
