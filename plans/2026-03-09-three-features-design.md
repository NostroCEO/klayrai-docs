# Design: Diagnostic Pipeline + Meta OAuth + Landing Page

**Date:** 2026-03-09
**Status:** Approved

---

## Feature 1: Automated Diagnostic Pipeline

### Architecture
```
Vercel Cron → /api/cron/diagnostics → diagnostic.service.ts
  → mcp-client.service.ts (HTTP → klayrai-meta-mcp on Render)
  → llm.service.ts (Claude API with diagnostic-engine SKILL.md as system prompt)
  → Zod validation → Prisma save (Diagnostic + Recommendation)
  → Resend email summary (optional)
```

### Key Decisions
- **Vercel Cron for V1** (not n8n) — zero extra infra
- **6h cache** for Starter, **1h** for Pro
- **Agentic tool_use loop** in llm.service.ts — must handle multiple tool calls

### Files
- `src/app/api/cron/diagnostics/route.ts`
- `src/server/services/diagnostic.service.ts`
- `vercel.json` (cron config)
- Update `llm.service.ts` (agentic loop)

---

## Feature 2: Meta OAuth Connection Flow

### Architecture
```
"Connect Meta" button → /api/auth/meta (redirect to Meta OAuth)
  → Meta redirects to /api/auth/meta/callback
  → Exchange code for long-lived token
  → Encrypt + store in MetaAccountRef (Prisma)
  → MCP receives per-request token (stays stateless)
```

### Key Decisions
- **Per-request tokens** — MCP stays stateless, tokens stored in Prisma only
- **60-day refresh** — store `tokenExpiresAt`, auto-refresh before expiry
- **Meta permissions:** `ads_read`, `ads_management`, `business_management`

### Files
- `src/app/api/auth/meta/route.ts` (OAuth initiation)
- `src/app/api/auth/meta/callback/route.ts` (token exchange)
- `src/app/(app)/onboarding/page.tsx` (3-step flow)
- `src/components/onboarding/connect-meta.tsx`
- `src/components/onboarding/select-accounts.tsx`
- Update `mcp-client.service.ts` (per-request token)
- Update `klayrai-meta-mcp` (accept per-request token override)

---

## Feature 3: Landing Page (klayrai.com)

### Design
- **Font:** Manrope (Google Fonts)
- **Background:** off-white `#f8f8f6`
- **Headings:** 60px, weight 300, letter-spacing -1.5px
- **Buttons:** square (0px border-radius), dark fill `#1a1a1a`
- **Style:** Editorial minimal (ModaFlows-inspired)

### Sections
1. Hero — headline + dual CTA (Start free trial + Book demo)
2. Problem — Why campaigns underperform (educational)
3. How It Works — 3 steps: Connect → Diagnose → Optimize
4. Pricing — Starter EUR49/mo, Pro EUR149/mo
5. Book Demo — Form + Turnstile CAPTCHA
6. Footer

### Files
- `src/app/(marketing)/layout.tsx`
- `src/app/(marketing)/page.tsx`
- `src/components/marketing/{hero,problem-section,how-it-works,pricing,book-demo-form,footer}.tsx`
- `src/server/actions/book-demo.ts`
- `src/lib/validations/book-demo.ts`
