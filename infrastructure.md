# Klayrai — Infrastructure Decisions

## Principle
Managed everything. Own nothing. MCP handles all external API communication.
Every infrastructure decision is documented here from day one through V3.

---

## Hosting

| Service | Provider | Rationale |
|---------|----------|-----------|
| Next.js app (frontend + backend) | Vercel | Push-to-deploy, preview URLs per PR, Cron jobs, Edge Middleware |
| MCP server (klayrai-meta-mcp) | Render.com | Always-on Node.js process, free tier, simple deploy from GitHub |
| DNS, CDN, CAPTCHA | Cloudflare | Free CDN, DDoS protection, Turnstile CAPTCHA |
| Domain registrar | Cloudflare Registrar | klayrai.com + app.klayrai.com subdomain |

---

## Database

| Layer | Service | Rationale |
|-------|---------|-----------|
| Primary database | Supabase Postgres | Managed Postgres, built-in connection pooling via Supavisor, dashboard and SQL editor |
| ORM | Prisma v5 | Type-safe, schema-first, migrations with history |
| Connection pooling | Supabase Supavisor (built-in) | Handles serverless cold starts without exhausting connections |
| Backups | Supabase automatic daily backups | No manual backup management, 7-day retention on Pro plan |
| V2 extension | PgVector on Supabase | Enable with CREATE EXTENSION vector — already supported by Supabase |

---

## Meta API — MCP-Only Strategy

| Layer | Implementation |
|-------|---------------|
| Source of truth for all Meta data | klayrai-meta-mcp MCP server (fork of brijr/meta-mcp) |
| API version | Marketing API v23.0 (pinned, never auto-upgrade) |
| Tools available | 25 tools covering analytics, campaigns, ad sets, ads, audiences, creatives |
| Authentication | Meta OAuth managed entirely by MCP via env vars |
| Rate limiting | Built into MCP with per-account tracking and intelligent backoff |
| Retry logic | Automatic retries with exponential backoff built into MCP |
| Next.js app | ZERO direct Meta API calls — MCP is the only interface |

---

## Authentication

| Layer | Service | Rationale |
|-------|---------|-----------|
| Auth provider | Firebase Auth | Ready-made, handles Google OAuth, sessions, ID tokens, Edge-compatible via Admin SDK |
| Session strategy | Firebase ID tokens verified server-side | Stateless, works on Vercel Edge Functions |
| MFA | Firebase Auth built-in (activate in V2) | No custom implementation required |
| User sync | First-request provisioning | Creates User record in Postgres on first authenticated request if not exists |

---

## LLM and AI

| Use Case | Service | Model | Notes |
|----------|---------|-------|-------|
| Diagnostic analysis | Anthropic Claude API | claude-opus-4-5 | Best structured reasoning, v1 |
| Report generation | Anthropic Claude API | claude-sonnet-4-5 | Faster and cheaper for volume |
| V2: Embeddings | Anthropic or OpenAI text-embedding-3-small | PgVector | For RAG on campaign history |
| V2: Multi-agent | LangGraph | — | Plan for, do not build in v1 |

---

## Email

| Use Case | Service |
|----------|---------|
| Transactional emails (welcome, report ready, alerts, demo confirmation) | Resend |
| Email templates | React Email (JSX-based, works with Resend) |

---

## Payments

- Provider: Stripe
- Products: Starter EUR 49/mo, Pro EUR 149/mo, Agency EUR 499/mo
- Self-serve: Stripe Customer Portal for plan upgrades, downgrades, cancellations
- Webhooks: /api/webhooks/stripe with Stripe-Signature header verification
- PCI compliance: Stripe handles it entirely — no card data ever touches our database

---

## File Storage

- Provider: UploadThing
- Use case: future asset uploads (ad creative attachments, report file exports in V2)
- CDN delivery: included in UploadThing
- File type validation: server-side via UploadThing config
- Not needed in V1 — prepare integration but do not build until required

---

## Security Infrastructure

| Layer | Tool | Configuration |
|-------|------|---------------|
| CAPTCHA | Cloudflare Turnstile | Auth sign-up form + Book Demo form |
| Rate limiting | @upstash/ratelimit + Upstash Redis | Apply to: all tRPC mutations, Server Actions, demo form |
| CORS | Next.js middleware | Restrict to https://klayrai.com and https://app.klayrai.com only |
| Row Level Security | Postgres native RLS policies | All multi-tenant tables scoped to workspaceId |
| Meta token storage | Not in app DB — MCP handles token storage via env vars | No encrypted tokens in Postgres in V1 |
| Secrets management | Vercel Env Manager | Never hardcode, never commit, use .env.example for documentation |

---

## Monitoring and Observability

| Tool | Purpose | Setup Day |
|------|---------|-----------|
| Sentry | Error tracking for client and server | Day 1 — before first deploy |
| PostHog | Product analytics, user event tracking, funnels | Day 1 — before first deploy |
| Vercel Analytics | Core Web Vitals and performance | Day 1 — built into Vercel, zero config |
| Vercel Speed Insights | Real user performance data | Day 1 — built into Vercel |
| Axiom | Log aggregation for debugging | V2 — when log volume justifies it |
| Lighthouse CI | Automated performance audits on PRs | Pre-launch and V2 CI setup |

---

## CI/CD Pipeline

| Trigger | Action |
|---------|--------|
| Push to any branch | Vercel creates preview deployment with unique URL |
| Push to main | Vercel deploys to production automatically |
| V2: PR opened | GitHub Actions runs npm audit + TypeScript strict check |
| V2: Pre-merge | GitHub Actions runs Lighthouse CI score check |

---

## Environment Variables (.env.example — copy this, never commit real values)

App URLs
NEXT_PUBLIC_APP_URL=https://app.klayrai.com
NEXT_PUBLIC_MARKETING_URL=https://www.klayrai.com
NODE_ENV=production

Firebase Auth
NEXT_PUBLIC_FIREBASE_API_KEY=
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=
NEXT_PUBLIC_FIREBASE_PROJECT_ID=
FIREBASE_ADMIN_PROJECT_ID=
FIREBASE_ADMIN_CLIENT_EMAIL=
FIREBASE_ADMIN_PRIVATE_KEY=

Database (Supabase Postgres)
DATABASE_URL=
DATABASE_URL_UNPOOLED=

Anthropic Claude API
ANTHROPIC_API_KEY=

Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=
STRIPE_STARTER_PRICE_ID=
STRIPE_PRO_PRICE_ID=
STRIPE_AGENCY_PRICE_ID=

Resend (Email)
RESEND_API_KEY=
RESEND_FROM_EMAIL=hello@klayrai.com

PostHog Analytics
NEXT_PUBLIC_POSTHOG_KEY=
NEXT_PUBLIC_POSTHOG_HOST=https://eu.posthog.com

Sentry Error Tracking
NEXT_PUBLIC_SENTRY_DSN=
SENTRY_AUTH_TOKEN=
SENTRY_ORG=
SENTRY_PROJECT=

Upstash Redis (Rate Limiting)
UPSTASH_REDIS_REST_URL=
UPSTASH_REDIS_REST_TOKEN=

Cloudflare Turnstile (CAPTCHA)
NEXT_PUBLIC_TURNSTILE_SITE_KEY=
TURNSTILE_SECRET_KEY=

UploadThing (File Uploads — V2)
UPLOADTHING_SECRET=
UPLOADTHING_APP_ID=

MCP Server env vars — set on Render.com, NOT on Vercel
META_APP_ID=
META_APP_SECRET=
META_ACCESS_TOKEN=
META_API_VERSION=v23.0
META_BUSINESS_ID=
META_AUTO_REFRESH=true

text

---

## V2 Infrastructure Additions (plan now, do not build yet)

- Enable PgVector extension on Supabase: CREATE EXTENSION vector
- Create Typesense Cloud account and collection for campaign search
- Set up Supabase Realtime for live dashboard updates (already on Supabase)
- Deploy LangGraph orchestration on Modal.com for long-running agent tasks
- Add Upstash Vector as alternative to PgVector if needed

## V3 Infrastructure Additions

- Upgrade Supabase to dedicated compute for heavy analytics queries
- Add Supabase read replica for reporting and analytics queries
- Evaluate ClickHouse for time-series ad performance data at scale
- Separate analytics database from operational database