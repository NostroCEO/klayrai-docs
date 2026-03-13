# FILE 2 — `/docs/architecture.md`

```markdown
# Klayrai — Architecture

---

## System Diagram

text
                      Cloudflare
                (DNS, CDN, DDoS, Turnstile)
                          |
      ┌───────────────────┼───────────────────┐
      |                   |                   |
www.klayrai.com app.klayrai.com Render.com
(Marketing + Demo) (PWA Dashboard) (MCP Server)
        | | |
    └───────────────────┼───────────────────┘
|
                ┌─────────▼─────────┐
                │ Vercel │
                │ Next.js 14 │
                │ App Router + RSC │
                └─────────┬─────────┘
|
    ┌────────────────────┼────────────────────┐
    | | |
    ┌─────▼─────┐ ┌───────▼───────┐ ┌──────▼──────┐
    │Firebase│ │Supabase Postgres│ │ Anthropic │
    │ Auth │ │ + Prisma ORM │ │ Claude API │
    └───────────┘ └───────────────┘ └──────┬──────┘
    | | |
    ┌─────▼─────┐ ┌───────▼───────┐ ┌──────▼──────┐
    │ Stripe │ │ Upstash Redis │ │ klayrai- │
    │ Billing │ │ Rate Limiting │ │ meta-mcp │
    └───────────┘ └───────────────┘ └──────┬──────┘
|
                ┌─────────▼──────────┐
                │ Meta Marketing API │
                │ v23.0 │
                └────────────────────┘



---

## Request Flow — Campaign Diagnostic

This is the primary value flow of the product. Every step is described in order.

Step 1
User opens /campaigns/[id] on app.klayrai.com

Step 2
Next.js Server Component renders the page shell immediately
Suspense boundary shows loading skeleton while data loads

Step 3
tRPC query fires: diagnostics.getDiagnosis
Input: { campaignId, dateRange, workspaceId }
Middleware: Firebase Auth check, rate limit check via Upstash

Step 4
diagnostic.service.ts receives the call

Queries Postgres for existing UserBehaviorProfile (Andromeda)

Checks if a fresh snapshot exists (created within last 6 hours)

If fresh snapshot exists: return it immediately (skip steps 5-8)

If no fresh snapshot: continue to step 5

Step 5
llm.service.ts calls Anthropic Claude API
System prompt: contents of .claude/skills/diagnostic-engine/SKILL.md
User message includes:
- campaignId and dateRange
- user_behavior_profile JSON from Andromeda
- instruction to use klayrai-meta-mcp tools
MCP tools available: klayrai-meta-mcp

Step 6
Claude diagnostic-agent executes tool calls
Tool call 1: get_insights({ campaignId, dateRange })
Tool call 2: get_ad_sets({ campaignId })
Tool call 3: compare_performance({ campaignId, currentPeriod, previousPeriod }) if trending analysis needed

Step 7
klayrai-meta-mcp (running on Render.com)

Receives tool call from Claude

Calls Meta Marketing API v23.0

Handles rate limiting and pagination internally

Returns normalized JSON to Claude

Step 8
Claude agent processes MCP data

Applies deterministic diagnostic rules (learning_stuck, creative_fatigue, pacing_issue, auction_overlap, budget_degradation)

Applies Andromeda profile to adapt tone, priorities, and language

Generates structured JSON output

Step 9
diagnostic.service.ts receives Claude output

Validates output shape with Zod

Saves Diagnostic record to Postgres

Saves Recommendation records to Postgres

Updates MetaAccountRef.lastDiagnosedAt

Returns result to tRPC handler

Step 10
Client receives data via tRPC

DiagnosticPanel renders flags and diagnosis text

RecommendationCard renders each recommendation with priority badge

DegradationPanel renders "why Meta is still spending here" if budget_degradation flag is true

text

---

## Request Flow — Book Demo (Landing Page)

Step 1
User fills out the Book Demo form on www.klayrai.com

Step 2
React Hook Form validates fields client-side with Zod schema
Cloudflare Turnstile CAPTCHA token is generated

Step 3
Form submits to Next.js Server Action: actions/book-demo.ts

Step 4
Server Action re-validates all fields with Zod (never trust client)
Verifies Turnstile token via Cloudflare API
Rate limit check via Upstash (3 requests per hour per IP)

Step 5
Saves DemoRequest record to Postgres

Step 6
Sends confirmation email via Resend with React Email template

Step 7
Returns success response
Client shows success message and resets form

text

---

## Request Flow — User Onboarding

Step 1
New user signs up via Firebase Auth (email or Google OAuth)

Step 2
Firebase Auth creates the user record. On first authenticated request, the app checks if a User record exists in Postgres.

Step 3
If no User record exists, the app creates User record in Postgres
Creates default Workspace record
Creates WorkspaceMember record with OWNER role
Creates empty UserBehaviorProfile record for Andromeda

Step 4
User is redirected to /onboarding

Step 5
Onboarding Step 1: Connect Meta account
User clicks "Connect Meta" button
OAuth flow via Meta App (managed externally, not in Next.js)
MCP receives and stores the access token

Step 6
Onboarding Step 2: Select ad accounts
Next.js calls get_ad_accounts via diagnostic-agent
List of available ad accounts is shown
User selects accounts to track
MetaAccountRef records created in Postgres for selected accounts

Step 7
Onboarding Step 3: Confirm and run first health check
Claude calls klayrai-meta-mcp.health_check
If OK: redirect to /dashboard
If error: show error with retry instructions

Step 8
/dashboard loads with first KPI data via diagnostic-agent + MCP

text

---

## Authentication and Authorization Flow

Every request to app.klayrai.com routes:

Request arrives
|
└─> Firebase Auth middleware (verifies ID token via Firebase Admin SDK)
|
├─> No valid token: redirect to /sign-in
|
└─> Valid token: extract uid from Firebase decoded token
|
└─> tRPC context is created
db: Prisma client
userId: string (from Firebase Auth, non-nullable in protectedProcedure)
|
└─> protectedProcedure middleware checks ctx.userId exists
|
└─> Resolver runs DB query scoped to workspaceId
|
└─> Postgres RLS policies enforce workspace isolation
at the database level as a second security layer

text

---

## tRPC Context Setup

```typescript
// src/server/trpc/trpc.ts

import { initTRPC, TRPCError } from "@trpc/server"
import { adminAuth } from "@/lib/firebase-admin"
import superjson from "superjson"
import { ZodError } from "zod"
import { db } from "@/server/db/prisma"

export const createTRPCContext = async (opts: { headers: Headers }) => {
  const token = opts.headers.get("authorization")?.replace("Bearer ", "")
  let userId: string | null = null
  if (token) {
    try {
      const decoded = await adminAuth.verifyIdToken(token)
      userId = decoded.uid
    } catch {
      userId = null
    }
  }
  return {
    db,
    userId,
    headers: opts.headers,
  }
}

const t = initTRPC.context<typeof createTRPCContext>().create({
  transformer: superjson,
  errorFormatter({ shape, error }) {
    return {
      ...shape,
      data: {
        ...shape.data,
        zodError: error.cause instanceof ZodError ? error.cause.flatten() : null,
      },
    }
  },
})

export const createTRPCRouter = t.router
export const publicProcedure = t.procedure

const enforceUserIsAuthed = t.middleware(({ ctx, next }) => {
  if (!ctx.userId) {
    throw new TRPCError({ code: "UNAUTHORIZED" })
  }
  return next({ ctx: { userId: ctx.userId } })
})

export const protectedProcedure = t.procedure.use(enforceUserIsAuthed)
Folder Structure
text
klayrai/
│
├── src/
│   │
│   ├── app/
│   │   │
│   │   ├── (marketing)/                         # www.klayrai.com pages
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx                         # Landing page
│   │   │   └── book-demo/
│   │   │       └── page.tsx
│   │   │
│   │   ├── (auth)/                              # Firebase Auth pages
│   │   │   ├── sign-in/
│   │   │   │   └── page.tsx
│   │   │   └── sign-up/
│   │   │       └── page.tsx
│   │   │
│   │   ├── (app)/                               # app.klayrai.com — protected
│   │   │   ├── layout.tsx                       # Sidebar + header shell
│   │   │   ├── dashboard/
│   │   │   │   └── page.tsx
│   │   │   ├── onboarding/
│   │   │   │   └── page.tsx
│   │   │   ├── campaigns/
│   │   │   │   ├── page.tsx                     # Campaigns list
│   │   │   │   └── [campaignId]/
│   │   │   │       ├── page.tsx                 # Campaign detail + diagnostics
│   │   │   │       └── ad-sets/
│   │   │   │           └── [adSetId]/
│   │   │   │               └── page.tsx
│   │   │   ├── insights/
│   │   │   │   └── page.tsx
│   │   │   ├── reports/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [reportId]/
│   │   │   │       └── page.tsx
│   │   │   └── settings/
│   │   │       └── page.tsx
│   │   │
│   │   ├── api/
│   │   │   ├── trpc/
│   │   │   │   └── [trpc]/
│   │   │   │       └── route.ts                 # tRPC HTTP handler
│   │   │   └── webhooks/
│   │   │       └── stripe/
│   │   │           └── route.ts                 # Stripe billing webhook
│   │   │
│   │   └── layout.tsx                           # Root layout with providers
│   │
│   ├── components/
│   │   ├── ui/                                  # shadcn/ui generated — do not edit manually
│   │   │
│   │   ├── marketing/
│   │   │   ├── hero.tsx
│   │   │   ├── how-it-works.tsx
│   │   │   ├── pricing.tsx
│   │   │   ├── social-proof.tsx
│   │   │   └── book-demo-form.tsx
│   │   │
│   │   ├── dashboard/
│   │   │   ├── kpi-card.tsx
│   │   │   ├── campaign-table.tsx
│   │   │   ├── diagnostic-panel.tsx
│   │   │   ├── recommendation-card.tsx
│   │   │   ├── degradation-panel.tsx
│   │   │   └── performance-chart.tsx
│   │   │
│   │   ├── onboarding/
│   │   │   ├── connect-meta.tsx
│   │   │   └── onboarding-steps.tsx
│   │   │
│   │   └── shared/
│   │       ├── empty-state.tsx
│   │       ├── loading-skeleton.tsx
│   │       └── date-range-picker.tsx
│   │
│   ├── server/
│   │   │
│   │   ├── trpc/
│   │   │   ├── root.ts                          # Merges all routers into appRouter
│   │   │   ├── trpc.ts                          # tRPC init, context, procedures
│   │   │   └── routers/
│   │   │       ├── campaigns.ts
│   │   │       ├── diagnostics.ts
│   │   │       ├── insights.ts
│   │   │       ├── reports.ts
│   │   │       ├── meta-accounts.ts
│   │   │       └── billing.ts
│   │   │
│   │   ├── actions/                             # Next.js Server Actions
│   │   │   ├── book-demo.ts
│   │   │   └── generate-report.ts
│   │   │
│   │   ├── services/
│   │   │   ├── diagnostic.service.ts            # Orchestrates rules engine + Claude agent
│   │   │   ├── llm.service.ts                   # Anthropic SDK wrapper, agent calls
│   │   │   ├── andromeda.service.ts             # User behavior profile computation
│   │   │   └── report.service.ts                # Report generation orchestration
│   │   │
│   │   └── db/
│   │       └── prisma.ts                        # Prisma client singleton
│   │
│   ├── lib/
│   │   ├── claude.ts                            # Anthropic SDK client instance
│   │   ├── stripe.ts                            # Stripe client instance
│   │   ├── rate-limit.ts                        # Upstash ratelimit instances
│   │   ├── resend.ts                            # Resend client instance
│   │   └── validations/
│   │       ├── book-demo.ts                     # Zod schema for demo form
│   │       ├── diagnostic.ts                    # Zod schema for diagnostic output
│   │       └── report.ts                        # Zod schema for report output
│   │
│   ├── hooks/
│   │   ├── use-campaigns.ts
│   │   ├── use-diagnostics.ts
│   │   └── use-date-range.ts
│   │
│   ├── store/
│   │   └── ui-store.ts                          # Zustand: sidebar open, active filters, modals
│   │
│   ├── types/
│   │   ├── diagnostic.ts
│   │   ├── andromeda.ts
│   │   └── meta.ts
│   │
│   └── middleware.ts                            # Firebase Auth middleware + CORS headers
│
├── .claude/
│   ├── skills/
│   │   ├── diagnostic-engine/
│   │   │   └── SKILL.md
│   │   ├── andromeda/
│   │   │   └── SKILL.md
│   │   ├── report-generator/
│   │   │   └── SKILL.md
│   │   ├── stripe/
│   │   │   └── SKILL.md
│   │   └── auth/
│   │       └── SKILL.md
│   └── agents/
│       ├── diagnostic-agent.json
│       ├── report-agent.json
│       └── sync-agent.json
│
├── klayrai-meta-mcp/                           # MCP server — separate deploy to Render.com
│   ├── src/
│   │   ├── index.ts
│   │   ├── tools/
│   │   └── meta-client.ts
│   ├── package.json
│   └── README.md
│
├── prisma/
│   ├── schema.prisma
│   └── migrations/
│
├── public/
│   ├── manifest.json                            # PWA manifest
│   ├── favicon.ico
│   └── og-image.png
│
├── CLAUDE.md
├── .env                                         # NEVER COMMIT — add to .gitignore
├── .env.example                                 # ALWAYS KEEP UPDATED — safe to commit
├── .gitignore
├── .claude.json                                 # MCP server registration for Claude Code
├── next.config.js
├── tailwind.config.ts
└── README.md


Rate Limiting Configuration
Applied via @upstash/ratelimit at the tRPC middleware level and in Server Actions.

Endpoint	Limit	Window	Reason
diagnostics.getDiagnosis	10	per minute per user	Claude API call cost
reports.generate	5	per hour per user	Claude API call cost
book-demo Server Action	3	per hour per IP	Spam protection
generate-report Server Action	5	per hour per user	Claude API call cost
All tRPC read queries	100	per minute per user	General abuse protection
PWA Configuration
manifest.json fields:

name: Klayrai

short_name: Klayrai

description: Meta Ads diagnostics and intelligence

start_url: /dashboard

display: standalone

theme_color: your primary brand color

background_color: your background color

icons: 192x192 and 512x512 PNG

Service worker via next-pwa:

Caches app shell and static assets

Offline fallback page at /offline

Does not cache API responses or MCP data

Install prompt:

Custom banner component triggered after 3 visits to the dashboard

Respects beforeinstallprompt event on Android and Chrome desktop

Shows manual instructions for iOS Safari

Environment Variable Reference
All variables that must exist in Vercel Env Manager for production.
All must be documented in .env.example with placeholder values.
None may appear as literal values anywhere in the codebase.

Group: App
NEXT_PUBLIC_APP_URL
NEXT_PUBLIC_MARKETING_URL
NODE_ENV

Group: Firebase Auth
NEXT_PUBLIC_FIREBASE_API_KEY
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN
NEXT_PUBLIC_FIREBASE_PROJECT_ID
FIREBASE_ADMIN_PROJECT_ID
FIREBASE_ADMIN_CLIENT_EMAIL
FIREBASE_ADMIN_PRIVATE_KEY

Group: Database
DATABASE_URL
DATABASE_URL_UNPOOLED

Group: Anthropic
ANTHROPIC_API_KEY

Group: Stripe
STRIPE_SECRET_KEY
STRIPE_WEBHOOK_SECRET
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY
STRIPE_STARTER_PRICE_ID
STRIPE_PRO_PRICE_ID
STRIPE_AGENCY_PRICE_ID

Group: Resend
RESEND_API_KEY
RESEND_FROM_EMAIL

Group: PostHog
NEXT_PUBLIC_POSTHOG_KEY
NEXT_PUBLIC_POSTHOG_HOST

Group: Sentry
NEXT_PUBLIC_SENTRY_DSN
SENTRY_AUTH_TOKEN
SENTRY_ORG
SENTRY_PROJECT

Group: Upstash Redis
UPSTASH_REDIS_REST_URL
UPSTASH_REDIS_REST_TOKEN

Group: Cloudflare Turnstile
NEXT_PUBLIC_TURNSTILE_SITE_KEY
TURNSTILE_SECRET_KEY

Group: UploadThing (prepare now, activate in v2)
UPLOADTHING_SECRET
UPLOADTHING_APP_ID

Group: MCP Server (set on Render.com, NOT on Vercel)
META_APP_ID
META_APP_SECRET
META_ACCESS_TOKEN
META_API_VERSION
META_BUSINESS_ID
META_AUTO_REFRESH