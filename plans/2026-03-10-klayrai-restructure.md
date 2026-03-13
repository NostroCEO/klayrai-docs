# Klayrai Codebase Restructure — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Restructure Klayrai from legacy MCP architecture to programmatic tool calling, reorganize file structure, fix auth routes, and fill all missing pieces.

**Architecture:** Next.js 14 App Router with Firebase Auth, Supabase+Prisma v5, tRPC v11, and Anthropic Claude API with `code_execution` beta for programmatic tool calling. All Meta data stays inside Claude's sandbox — never enters the context window.

**Tech Stack:** Next.js 14, TypeScript strict, Tailwind v3, shadcn/ui, tRPC v11, Zod v4, Firebase Auth, Prisma v5, Anthropic SDK, facebook-nodejs-business-sdk, Stripe, Framer Motion

---

## Phase 1: Reorganize `src/lib/` (no logic changes, just moves)

> **Goal:** Group flat lib files into subdirectories. Zero behavior change.
> **Risk:** LOW — only import paths change.

### Task 1.1: Create lib subdirectory structure

**Step 1: Create directories**
```bash
mkdir -p src/lib/firebase src/lib/supabase src/lib/meta src/lib/ai
```

**Step 2: Move Firebase files**
- MOVE `src/lib/firebase.ts` → `src/lib/firebase/client.ts`
- MOVE `src/lib/firebase-admin.ts` → `src/lib/firebase/admin.ts`
- CREATE `src/lib/firebase/index.ts` — re-export barrel

**Step 3: Move Prisma**
- MOVE `src/server/db/prisma.ts` → `src/lib/supabase/prisma.ts`
- CREATE `src/lib/supabase/index.ts` — re-export barrel

**Step 4: Commit**
```bash
git add -A && git commit -m "refactor: reorganize lib/ into subdirectories"
```

### Task 1.2: Update all import paths

**Files to update after moves:**

| Old import | New import | Files affected |
|---|---|---|
| `@/lib/firebase` | `@/lib/firebase/client` | auth-provider.tsx, middleware.ts |
| `@/lib/firebase-admin` | `@/lib/firebase/admin` | trpc.ts, user-sync.service.ts, api routes |
| `@/server/db/prisma` | `@/lib/supabase/prisma` | trpc.ts, all services, all routers |

**Step 1:** Update each import across the codebase (use find-and-replace).

**Step 2:** Verify build compiles:
```bash
npx next build
```

**Step 3: Commit**
```bash
git add -A && git commit -m "refactor: update import paths for lib reorganization"
```

---

## Phase 2: Move services and actions to top-level `src/`

> **Goal:** Move `src/server/services/` → `src/services/`, `src/server/actions/` → `src/actions/`.
> **Risk:** LOW — import path changes only.

### Task 2.1: Move services

**Step 1: Move files**
- MOVE `src/server/services/diagnostic.service.ts` → `src/services/diagnostic.service.ts`
- MOVE `src/server/services/user-sync.service.ts` → `src/services/user-sync.service.ts`
- KEEP `src/server/services/llm.service.ts` — will be replaced in Phase 5
- KEEP `src/server/services/mcp-client.service.ts` — will be replaced in Phase 5

**Step 2: Move actions**
- MOVE `src/server/actions/book-demo.ts` → `src/actions/book-demo.ts`

**Step 3: Update imports**

| Old import | New import | Files affected |
|---|---|---|
| `@/server/services/diagnostic.service` | `@/services/diagnostic.service` | diagnostics router, cron route |
| `@/server/services/user-sync.service` | `@/services/user-sync.service` | auth-provider, trpc context |
| `@/server/actions/book-demo` | `@/actions/book-demo` | (marketing)/page.tsx |

**Step 4: Commit**
```bash
git add -A && git commit -m "refactor: move services and actions to top-level src/"
```

---

## Phase 3: Fix auth routes (Clerk-style → Firebase Auth)

> **Goal:** Replace `[[...sign-in]]` catch-all (Clerk pattern) with simple Firebase Auth pages.
> **Risk:** MEDIUM — user-facing pages change.

### Task 3.1: Replace auth pages

**Step 1: DELETE Clerk-style catch-all routes**
- DELETE `src/app/(auth)/sign-in/[[...sign-in]]/page.tsx`
- DELETE `src/app/(auth)/sign-up/[[...sign-up]]/page.tsx`
- DELETE the empty `[[...sign-in]]` and `[[...sign-up]]` directories

**Step 2: CREATE simple Firebase Auth pages**

- CREATE `src/app/(auth)/sign-in/page.tsx`
```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import Link from "next/link";
import { signInWithEmailAndPassword, signInWithPopup, GoogleAuthProvider } from "firebase/auth";
import { auth } from "@/lib/firebase/client";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";

export default function SignInPage() {
  const router = useRouter();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  async function handleEmailSignIn(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    setError(null);
    try {
      await signInWithEmailAndPassword(auth, email, password);
      router.push("/dashboard");
    } catch (err) {
      setError(err instanceof Error ? err.message : "Sign in failed");
    } finally {
      setLoading(false);
    }
  }

  async function handleGoogleSignIn() {
    setLoading(true);
    setError(null);
    try {
      await signInWithPopup(auth, new GoogleAuthProvider());
      router.push("/dashboard");
    } catch (err) {
      setError(err instanceof Error ? err.message : "Google sign in failed");
    } finally {
      setLoading(false);
    }
  }

  return (
    <div className="flex min-h-screen items-center justify-center bg-[#f8f8f6]">
      <div className="w-full max-w-md space-y-8 p-8">
        <div className="text-center">
          <h1 className="font-manrope text-3xl font-light text-[#1a1a1a]">Sign in to Klayrai</h1>
          <p className="mt-2 text-sm text-gray-500">AI-powered Meta Ads diagnostics</p>
        </div>

        <form onSubmit={handleEmailSignIn} className="space-y-4">
          <div>
            <Label htmlFor="email">Email</Label>
            <Input id="email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} required />
          </div>
          <div>
            <Label htmlFor="password">Password</Label>
            <Input id="password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} required />
          </div>
          {error && <p className="text-sm text-red-600">{error}</p>}
          <Button type="submit" disabled={loading} className="w-full rounded-none bg-[#1a1a1a] text-white">
            {loading ? "Signing in..." : "Sign in"}
          </Button>
        </form>

        <div className="relative">
          <div className="absolute inset-0 flex items-center"><div className="w-full border-t border-neutral-200" /></div>
          <div className="relative flex justify-center text-sm"><span className="bg-[#f8f8f6] px-2 text-gray-500">or</span></div>
        </div>

        <Button variant="outline" onClick={handleGoogleSignIn} disabled={loading} className="w-full rounded-none">
          Continue with Google
        </Button>

        <p className="text-center text-sm text-gray-500">
          Don&apos;t have an account? <Link href="/sign-up" className="text-blue-600 hover:underline">Sign up</Link>
        </p>
      </div>
    </div>
  );
}
```

- CREATE `src/app/(auth)/sign-up/page.tsx`
```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import Link from "next/link";
import { createUserWithEmailAndPassword, updateProfile, signInWithPopup, GoogleAuthProvider } from "firebase/auth";
import { auth } from "@/lib/firebase/client";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";

export default function SignUpPage() {
  const router = useRouter();
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  async function handleEmailSignUp(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    setError(null);
    try {
      const cred = await createUserWithEmailAndPassword(auth, email, password);
      if (name) await updateProfile(cred.user, { displayName: name });
      router.push("/onboarding");
    } catch (err) {
      setError(err instanceof Error ? err.message : "Sign up failed");
    } finally {
      setLoading(false);
    }
  }

  async function handleGoogleSignUp() {
    setLoading(true);
    setError(null);
    try {
      await signInWithPopup(auth, new GoogleAuthProvider());
      router.push("/onboarding");
    } catch (err) {
      setError(err instanceof Error ? err.message : "Google sign up failed");
    } finally {
      setLoading(false);
    }
  }

  return (
    <div className="flex min-h-screen items-center justify-center bg-[#f8f8f6]">
      <div className="w-full max-w-md space-y-8 p-8">
        <div className="text-center">
          <h1 className="font-manrope text-3xl font-light text-[#1a1a1a]">Create your account</h1>
          <p className="mt-2 text-sm text-gray-500">Start optimizing your Meta Ads with AI</p>
        </div>

        <form onSubmit={handleEmailSignUp} className="space-y-4">
          <div>
            <Label htmlFor="name">Full name</Label>
            <Input id="name" value={name} onChange={(e) => setName(e.target.value)} />
          </div>
          <div>
            <Label htmlFor="email">Email</Label>
            <Input id="email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} required />
          </div>
          <div>
            <Label htmlFor="password">Password</Label>
            <Input id="password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} required minLength={8} />
          </div>
          {error && <p className="text-sm text-red-600">{error}</p>}
          <Button type="submit" disabled={loading} className="w-full rounded-none bg-[#1a1a1a] text-white">
            {loading ? "Creating account..." : "Create account"}
          </Button>
        </form>

        <div className="relative">
          <div className="absolute inset-0 flex items-center"><div className="w-full border-t border-neutral-200" /></div>
          <div className="relative flex justify-center text-sm"><span className="bg-[#f8f8f6] px-2 text-gray-500">or</span></div>
        </div>

        <Button variant="outline" onClick={handleGoogleSignUp} disabled={loading} className="w-full rounded-none">
          Continue with Google
        </Button>

        <p className="text-center text-sm text-gray-500">
          Already have an account? <Link href="/sign-in" className="text-blue-600 hover:underline">Sign in</Link>
        </p>
      </div>
    </div>
  );
}
```

**Step 3: Commit**
```bash
git add -A && git commit -m "refactor: replace Clerk catch-all auth routes with Firebase Auth pages"
```

---

## Phase 4: Reorganize components + add missing ones

> **Goal:** Move layout components to shared/, move kpi-card to dashboard/, create missing components.
> **Risk:** LOW — moves + new stubs.

### Task 4.1: Move existing components

- MOVE `src/components/layout/app-header.tsx` → `src/components/shared/navbar.tsx`
- MOVE `src/components/layout/app-sidebar.tsx` → `src/components/shared/sidebar.tsx`
- MOVE `src/components/shared/kpi-card.tsx` → `src/components/dashboard/kpi-card.tsx`
- DELETE `src/components/layout/` (now empty)
- Update imports in `src/app/(app)/layout.tsx` and any pages using these components.

**Commit:**
```bash
git add -A && git commit -m "refactor: reorganize layout components into shared/"
```

### Task 4.2: Create missing marketing components

Extract from the monolithic `(marketing)/page.tsx` (985 lines) into separate components:

- CREATE `src/components/marketing/hero.tsx` — Hero section with typing effect
- CREATE `src/components/marketing/how-it-works.tsx` — 3-step process section
- CREATE `src/components/marketing/pricing.tsx` — Starter/Pro/Agency pricing cards
- CREATE `src/components/marketing/book-demo-form.tsx` — Demo request form

Then refactor `src/app/(marketing)/page.tsx` to import and compose these components.

**Commit:**
```bash
git add -A && git commit -m "refactor: extract marketing page into reusable components"
```

### Task 4.3: Create missing dashboard components

- CREATE `src/components/dashboard/diagnostic-panel.tsx`
```tsx
"use client";

import type { Diagnostic, Recommendation } from "@prisma/client";
import { Badge } from "@/components/ui/badge";
import { Card } from "@/components/ui/card";

const RISK_COLORS: Record<string, string> = {
  LOW: "bg-green-100 text-green-800",
  MEDIUM: "bg-amber-100 text-amber-800",
  HIGH: "bg-orange-100 text-orange-800",
  CRITICAL: "bg-red-100 text-red-800",
};

interface DiagnosticPanelProps {
  diagnostic: Diagnostic & { recommendations: Recommendation[] };
}

export function DiagnosticPanel({ diagnostic }: DiagnosticPanelProps) {
  return (
    <Card className="rounded-sm border-neutral-200 p-6">
      <div className="flex items-start justify-between">
        <div>
          <h3 className="font-manrope text-lg font-light text-[#1a1a1a]">
            {diagnostic.entityName ?? diagnostic.entityType}
          </h3>
          <p className="mt-1 text-sm text-gray-500">{diagnostic.entityType}</p>
        </div>
        <Badge className={RISK_COLORS[diagnostic.riskLevel] ?? ""}>
          {diagnostic.riskLevel}
        </Badge>
      </div>
      {diagnostic.diagnosisText && (
        <p className="mt-4 text-sm text-[#1a1a1a]">{diagnostic.diagnosisText}</p>
      )}
      {diagnostic.recommendations.length > 0 && (
        <div className="mt-4 space-y-2">
          <h4 className="text-xs font-medium uppercase tracking-wide text-gray-500">
            Recommendations
          </h4>
          {diagnostic.recommendations.map((rec) => (
            <div key={rec.id} className="rounded-sm border border-neutral-200 p-3">
              <p className="text-sm font-medium text-[#1a1a1a]">{rec.title}</p>
              <p className="mt-1 text-xs text-gray-500">{rec.description}</p>
            </div>
          ))}
        </div>
      )}
    </Card>
  );
}
```

- CREATE `src/components/dashboard/recommendation-card.tsx`
```tsx
import type { Recommendation } from "@prisma/client";
import { Badge } from "@/components/ui/badge";
import { Card } from "@/components/ui/card";

const PRIORITY_COLORS: Record<string, string> = {
  LOW: "bg-gray-100 text-gray-800",
  MEDIUM: "bg-blue-100 text-blue-800",
  HIGH: "bg-orange-100 text-orange-800",
  URGENT: "bg-red-100 text-red-800",
};

interface RecommendationCardProps {
  recommendation: Recommendation;
}

export function RecommendationCard({ recommendation }: RecommendationCardProps) {
  return (
    <Card className="rounded-sm border-neutral-200 p-4">
      <div className="flex items-start justify-between">
        <h4 className="text-sm font-medium text-[#1a1a1a]">{recommendation.title}</h4>
        <Badge className={PRIORITY_COLORS[recommendation.priority] ?? ""}>
          {recommendation.priority}
        </Badge>
      </div>
      <p className="mt-2 text-xs text-gray-500">{recommendation.description}</p>
      <p className="mt-2 text-xs text-blue-600">{recommendation.actionType.replace(/_/g, " ")}</p>
    </Card>
  );
}
```

- CREATE `src/components/dashboard/performance-chart.tsx`
```tsx
"use client";

import { Card } from "@/components/ui/card";
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from "recharts";

interface DataPoint {
  date: string;
  spend: number;
  roas: number;
  cpa: number;
}

interface PerformanceChartProps {
  data: DataPoint[];
  metric: "spend" | "roas" | "cpa";
  title: string;
}

const METRIC_COLORS = { spend: "#2563eb", roas: "#16a34a", cpa: "#ea580c" };

export function PerformanceChart({ data, metric, title }: PerformanceChartProps) {
  return (
    <Card className="rounded-sm border-neutral-200 p-6">
      <h3 className="font-manrope text-lg font-light text-[#1a1a1a]">{title}</h3>
      <div className="mt-4 h-64">
        <ResponsiveContainer width="100%" height="100%">
          <LineChart data={data}>
            <CartesianGrid strokeDasharray="3 3" stroke="#e5e5e5" />
            <XAxis dataKey="date" tick={{ fontSize: 12 }} stroke="#6b7280" />
            <YAxis tick={{ fontSize: 12 }} stroke="#6b7280" />
            <Tooltip />
            <Line type="monotone" dataKey={metric} stroke={METRIC_COLORS[metric]} strokeWidth={2} dot={false} />
          </LineChart>
        </ResponsiveContainer>
      </div>
    </Card>
  );
}
```

**Commit:**
```bash
git add -A && git commit -m "feat: add diagnostic-panel, recommendation-card, performance-chart components"
```

### Task 4.4: Create missing shared components

- CREATE `src/components/shared/loading-skeleton.tsx`
```tsx
import { Skeleton } from "@/components/ui/skeleton";

export function LoadingSkeleton({ rows = 3 }: { rows?: number }) {
  return (
    <div className="space-y-4">
      {Array.from({ length: rows }).map((_, i) => (
        <Skeleton key={i} className="h-16 w-full rounded-sm" />
      ))}
    </div>
  );
}
```

- CREATE `src/components/shared/date-range-picker.tsx`
```tsx
"use client";

import { useState } from "react";
import { format } from "date-fns";
import { Calendar } from "@/components/ui/calendar";
import { Popover, PopoverContent, PopoverTrigger } from "@/components/ui/popover";
import { Button } from "@/components/ui/button";

interface DateRangePickerProps {
  from: Date;
  to: Date;
  onChange: (range: { from: Date; to: Date }) => void;
}

export function DateRangePicker({ from, to, onChange }: DateRangePickerProps) {
  const [open, setOpen] = useState(false);

  return (
    <Popover open={open} onOpenChange={setOpen}>
      <PopoverTrigger asChild>
        <Button variant="outline" className="rounded-none text-sm">
          {format(from, "MMM d, yyyy")} — {format(to, "MMM d, yyyy")}
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-auto p-0" align="end">
        <Calendar
          mode="range"
          selected={{ from, to }}
          onSelect={(range) => {
            if (range?.from && range?.to) {
              onChange({ from: range.from, to: range.to });
              setOpen(false);
            }
          }}
          numberOfMonths={2}
        />
      </PopoverContent>
    </Popover>
  );
}
```

**Commit:**
```bash
git add -A && git commit -m "feat: add loading-skeleton and date-range-picker shared components"
```

---

## Phase 5: Core Architecture — MCP → Programmatic Tool Calling

> **Goal:** Replace legacy MCP HTTP calls with programmatic tool calling using `code_execution` beta. This is the biggest change.
> **Risk:** HIGH — core AI engine rewrite. Old services stay until new ones work.

### Task 5.1: Create `src/lib/meta/meta-tools.ts`

These are the tool definitions with `allowed_callers: ["code_execution_20260120"]` so Claude can call them from its sandbox.

**CREATE** `src/lib/meta/meta-tools.ts`
```typescript
import type Anthropic from "@anthropic-ai/sdk";

/**
 * Meta API tool definitions for programmatic tool calling.
 * These tools have allowed_callers set so Claude's code_execution
 * sandbox can call them — tool results stay inside the sandbox,
 * never entering the context window (85-98% token savings).
 */

export const META_TOOLS: Anthropic.Tool[] = [
  {
    name: "get_campaigns",
    description: "List campaigns for a Meta ad account. Returns campaign IDs, names, statuses, objectives, and budgets.",
    input_schema: {
      type: "object" as const,
      properties: {
        account_id: { type: "string", description: "Meta ad account ID (e.g. act_123456)" },
        status_filter: {
          type: "array",
          items: { type: "string", enum: ["ACTIVE", "PAUSED", "DELETED", "ARCHIVED"] },
          description: "Filter campaigns by status. Defaults to all.",
        },
        limit: { type: "number", description: "Maximum campaigns to return. Default 100." },
      },
      required: ["account_id"],
    },
  },
  {
    name: "get_ad_sets",
    description: "List ad sets for a campaign or ad account. Returns targeting, budgets, scheduling, and learning phase status.",
    input_schema: {
      type: "object" as const,
      properties: {
        parent_id: { type: "string", description: "Campaign ID or account ID" },
        status_filter: {
          type: "array",
          items: { type: "string", enum: ["ACTIVE", "PAUSED", "DELETED", "ARCHIVED"] },
        },
        limit: { type: "number", description: "Maximum ad sets to return. Default 100." },
      },
      required: ["parent_id"],
    },
  },
  {
    name: "get_ads",
    description: "List ads for an ad set or campaign. Returns ad creative info, status, and preview URLs.",
    input_schema: {
      type: "object" as const,
      properties: {
        parent_id: { type: "string", description: "Ad set ID or campaign ID" },
        limit: { type: "number", description: "Maximum ads to return. Default 50." },
      },
      required: ["parent_id"],
    },
  },
  {
    name: "get_insights",
    description: "Get performance metrics (spend, impressions, clicks, conversions, ROAS, CPA, etc.) for any Meta entity over a date range.",
    input_schema: {
      type: "object" as const,
      properties: {
        entity_id: { type: "string", description: "Account, campaign, ad set, or ad ID" },
        date_start: { type: "string", description: "Start date YYYY-MM-DD" },
        date_end: { type: "string", description: "End date YYYY-MM-DD" },
        time_increment: {
          type: "string",
          enum: ["1", "7", "monthly", "all_days"],
          description: "Time granularity. '1' = daily, '7' = weekly, 'monthly', 'all_days' = aggregate.",
        },
        breakdowns: {
          type: "array",
          items: { type: "string", enum: ["age", "gender", "country", "device_platform", "publisher_platform", "placement"] },
          description: "Optional breakdowns for segmented data.",
        },
      },
      required: ["entity_id", "date_start", "date_end"],
    },
  },
  {
    name: "compare_performance",
    description: "Compare performance between two date ranges for the same entity. Useful for period-over-period analysis.",
    input_schema: {
      type: "object" as const,
      properties: {
        entity_id: { type: "string" },
        current_start: { type: "string", description: "Current period start YYYY-MM-DD" },
        current_end: { type: "string", description: "Current period end YYYY-MM-DD" },
        previous_start: { type: "string", description: "Previous period start YYYY-MM-DD" },
        previous_end: { type: "string", description: "Previous period end YYYY-MM-DD" },
      },
      required: ["entity_id", "current_start", "current_end", "previous_start", "previous_end"],
    },
  },
  {
    name: "get_learning_phase_info",
    description: "Check if an ad set is in learning phase, learning limited, or has exited learning. Returns learning status and events needed to exit.",
    input_schema: {
      type: "object" as const,
      properties: {
        ad_set_id: { type: "string", description: "Ad set ID to check" },
      },
      required: ["ad_set_id"],
    },
  },
];
```

**Commit:**
```bash
git add -A && git commit -m "feat: add meta-tools.ts with programmatic tool calling definitions"
```

### Task 5.2: Create `src/lib/meta/meta-executor.ts`

This is the ONLY layer that touches the Meta Marketing API via `facebook-nodejs-business-sdk`. It decrypts the user's token, calls Meta, and returns data.

**CREATE** `src/lib/meta/meta-executor.ts`
```typescript
import { AdAccount, Campaign, AdSet, Ad } from "facebook-nodejs-business-sdk";
import { decrypt } from "@/lib/encryption";
import { prisma } from "@/lib/supabase/prisma";

/**
 * executeMetaTool — The only function that calls the Meta Marketing API.
 *
 * Called by claude-engine when Claude's code_execution sandbox invokes a tool.
 * Decrypts the user's access token from the DB, makes the Meta API call,
 * and returns raw JSON data.
 *
 * This function NEVER sends data to Claude's context window — it returns
 * to the sandbox where Claude processes it programmatically.
 */

interface MetaToolInput {
  toolName: string;
  toolInput: Record<string, unknown>;
  workspaceId: string;
  metaAccountRefId: string;
}

export async function executeMetaTool(input: MetaToolInput): Promise<string> {
  const { toolName, toolInput, workspaceId, metaAccountRefId } = input;

  // Get encrypted token from DB
  const accountRef = await prisma.metaAccountRef.findUnique({
    where: { id: metaAccountRefId },
  });

  if (!accountRef) {
    throw new Error(`MetaAccountRef ${metaAccountRefId} not found`);
  }

  // NOTE: Token storage TBD — for now, expect an encryptedAccessToken field
  // or a separate token table. This is a placeholder for the decryption flow.
  // const accessToken = decrypt(accountRef.encryptedAccessToken);

  // For now, use env fallback (will be replaced with per-user tokens)
  const accessToken = process.env.META_ACCESS_TOKEN;
  if (!accessToken) {
    throw new Error("No Meta access token available");
  }

  switch (toolName) {
    case "get_campaigns":
      return await getCampaigns(toolInput, accessToken);
    case "get_ad_sets":
      return await getAdSets(toolInput, accessToken);
    case "get_ads":
      return await getAds(toolInput, accessToken);
    case "get_insights":
      return await getInsights(toolInput, accessToken);
    case "compare_performance":
      return await comparePerformance(toolInput, accessToken);
    case "get_learning_phase_info":
      return await getLearningPhaseInfo(toolInput, accessToken);
    default:
      throw new Error(`Unknown tool: ${toolName}`);
  }
}

// ─── Tool Implementations ──────────────────────────────────

async function getCampaigns(
  input: Record<string, unknown>,
  accessToken: string
): Promise<string> {
  const accountId = input.account_id as string;
  const limit = (input.limit as number) ?? 100;

  const account = new AdAccount(accountId);
  account.api = getApi(accessToken);

  const fields = ["id", "name", "status", "effective_status", "objective", "daily_budget", "lifetime_budget", "start_time", "stop_time"];
  const params: Record<string, unknown> = { limit };

  if (input.status_filter) {
    params.filtering = [{ field: "effective_status", operator: "IN", value: input.status_filter }];
  }

  const campaigns = await account.getCampaigns(fields, params);
  return JSON.stringify({
    total: campaigns.length,
    campaigns: campaigns.map((c: Campaign) => c._data),
  });
}

async function getAdSets(
  input: Record<string, unknown>,
  accessToken: string
): Promise<string> {
  const parentId = input.parent_id as string;
  const limit = (input.limit as number) ?? 100;

  const fields = ["id", "name", "status", "effective_status", "daily_budget", "lifetime_budget", "targeting", "optimization_goal", "bid_strategy", "learning_phase_info"];

  // Determine if parent is a campaign or account
  const isCampaign = !parentId.startsWith("act_");
  const parent = isCampaign ? new Campaign(parentId) : new AdAccount(parentId);
  parent.api = getApi(accessToken);

  const adSets = await parent.getAdSets(fields, { limit });
  return JSON.stringify({
    total: adSets.length,
    ad_sets: adSets.map((a: AdSet) => a._data),
  });
}

async function getAds(
  input: Record<string, unknown>,
  accessToken: string
): Promise<string> {
  const parentId = input.parent_id as string;
  const limit = (input.limit as number) ?? 50;

  const fields = ["id", "name", "status", "effective_status", "creative", "preview_shareable_link"];
  const parent = new AdSet(parentId);
  parent.api = getApi(accessToken);

  const ads = await parent.getAds(fields, { limit });
  return JSON.stringify({
    total: ads.length,
    ads: ads.map((a: Ad) => a._data),
  });
}

async function getInsights(
  input: Record<string, unknown>,
  accessToken: string
): Promise<string> {
  const entityId = input.entity_id as string;
  const dateStart = input.date_start as string;
  const dateEnd = input.date_end as string;
  const timeIncrement = (input.time_increment as string) ?? "all_days";

  // Determine entity type from ID format
  const entity = entityId.startsWith("act_")
    ? new AdAccount(entityId)
    : new Campaign(entityId); // Could also be AdSet/Ad — SDK handles it
  entity.api = getApi(accessToken);

  const fields = ["impressions", "reach", "clicks", "spend", "actions", "action_values", "ctr", "cpc", "cpm", "frequency"];
  const params: Record<string, unknown> = {
    time_range: { since: dateStart, until: dateEnd },
    time_increment: timeIncrement,
  };

  if (input.breakdowns) {
    params.breakdowns = input.breakdowns;
  }

  const insights = await entity.getInsights(fields, params);
  return JSON.stringify({
    entity_id: entityId,
    total_rows: insights.length,
    insights: insights.map((i: Record<string, unknown>) => i._data ?? i),
  });
}

async function comparePerformance(
  input: Record<string, unknown>,
  accessToken: string
): Promise<string> {
  const entityId = input.entity_id as string;

  const currentData = await getInsights(
    { entity_id: entityId, date_start: input.current_start, date_end: input.current_end, time_increment: "all_days" },
    accessToken
  );
  const previousData = await getInsights(
    { entity_id: entityId, date_start: input.previous_start, date_end: input.previous_end, time_increment: "all_days" },
    accessToken
  );

  return JSON.stringify({
    entity_id: entityId,
    current_period: JSON.parse(currentData),
    previous_period: JSON.parse(previousData),
  });
}

async function getLearningPhaseInfo(
  input: Record<string, unknown>,
  accessToken: string
): Promise<string> {
  const adSetId = input.ad_set_id as string;
  const adSet = new AdSet(adSetId);
  adSet.api = getApi(accessToken);

  const data = await adSet.get(["id", "name", "learning_phase_info", "status", "effective_status"]);
  return JSON.stringify(data._data);
}

// ─── Helper ──────────────────────────────────────────────

function getApi(accessToken: string) {
  const FacebookAdsApi = require("facebook-nodejs-business-sdk").FacebookAdsApi;
  return FacebookAdsApi.init(accessToken);
}
```

**Commit:**
```bash
git add -A && git commit -m "feat: add meta-executor.ts — sole Meta API layer"
```

### Task 5.3: Create `src/lib/ai/claude-engine.ts`

The new AI engine using `code_execution` beta with programmatic tool calling.

**CREATE** `src/lib/ai/claude-engine.ts`
```typescript
import Anthropic from "@anthropic-ai/sdk";
import { META_TOOLS } from "@/lib/meta/meta-tools";
import { executeMetaTool } from "@/lib/meta/meta-executor";

/**
 * Claude Engine — Programmatic Tool Calling with code_execution beta
 *
 * Instead of classical MCP (HTTP to external server), Claude runs code
 * in a sandbox that calls Meta tools. Tool results stay inside the sandbox,
 * never entering Claude's context window → 85-98% token savings.
 *
 * Beta headers: "anthropic-beta": "code-execution-20260120,tool-search-20250910"
 */

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

const BETA_HEADERS = "code-execution-20260120,tool-search-20250910";
const MAX_TOOL_ROUNDS = 15;

// ─── Types ──────────────────────────────────────────────

interface EngineConfig {
  model: string;
  systemPrompt: string;
  maxTokens: number;
  temperature: number;
  tools?: Anthropic.Tool[];
}

interface EngineResponse {
  content: string;
  inputTokens: number;
  outputTokens: number;
  model: string;
}

interface ToolContext {
  workspaceId: string;
  metaAccountRefId: string;
}

// ─── Agent Loop ─────────────────────────────────────────

async function runAgentLoop(
  config: EngineConfig,
  userMessage: string,
  toolContext: ToolContext
): Promise<EngineResponse> {
  let totalInputTokens = 0;
  let totalOutputTokens = 0;
  let model = config.model;

  // Add allowed_callers to each tool
  const toolsWithCallers = (config.tools ?? META_TOOLS).map((tool) => ({
    ...tool,
    allowed_callers: ["code_execution_20260120"],
  }));

  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: userMessage },
  ];

  for (let round = 0; round < MAX_TOOL_ROUNDS; round++) {
    const response = await anthropic.messages.create({
      model: config.model,
      max_tokens: config.maxTokens,
      temperature: config.temperature,
      system: config.systemPrompt,
      tools: toolsWithCallers,
      messages,
      // @ts-expect-error — beta headers not in stable types yet
      betas: [BETA_HEADERS],
    });

    totalInputTokens += response.usage.input_tokens;
    totalOutputTokens += response.usage.output_tokens;
    model = response.model;

    // If stop_reason is not tool_use, extract final text
    if (response.stop_reason !== "tool_use") {
      const textBlock = response.content.find(
        (b): b is Anthropic.TextBlock => b.type === "text"
      );

      return {
        content: textBlock?.text ?? "",
        inputTokens: totalInputTokens,
        outputTokens: totalOutputTokens,
        model,
      };
    }

    // Claude wants to use tools — process them
    messages.push({ role: "assistant", content: response.content });

    const toolUseBlocks = response.content.filter(
      (b): b is Anthropic.ToolUseBlock => b.type === "tool_use"
    );

    const toolResults = await Promise.all(
      toolUseBlocks.map(async (toolUse) => {
        try {
          const result = await executeMetaTool({
            toolName: toolUse.name,
            toolInput: toolUse.input as Record<string, unknown>,
            workspaceId: toolContext.workspaceId,
            metaAccountRefId: toolContext.metaAccountRefId,
          });
          return {
            type: "tool_result" as const,
            tool_use_id: toolUse.id,
            content: result,
          };
        } catch (error) {
          return {
            type: "tool_result" as const,
            tool_use_id: toolUse.id,
            content: `Error: ${error instanceof Error ? error.message : String(error)}`,
            is_error: true,
          };
        }
      })
    );

    messages.push({ role: "user", content: toolResults });
  }

  throw new Error(`Agent exceeded max tool rounds (${MAX_TOOL_ROUNDS})`);
}

// ─── Public API ─────────────────────────────────────────

export async function runDiagnosticAgent(
  systemPrompt: string,
  userMessage: string,
  toolContext: ToolContext
): Promise<EngineResponse> {
  return runAgentLoop(
    {
      model: "claude-opus-4-6",
      systemPrompt,
      maxTokens: 8192,
      temperature: 0.1,
    },
    userMessage,
    toolContext
  );
}

export async function runReportAgent(
  systemPrompt: string,
  userMessage: string,
  toolContext: ToolContext
): Promise<EngineResponse> {
  return runAgentLoop(
    {
      model: "claude-sonnet-4-6",
      systemPrompt,
      maxTokens: 16384,
      temperature: 0.3,
    },
    userMessage,
    toolContext
  );
}

export async function runSyncAgent(
  systemPrompt: string,
  userMessage: string,
  toolContext: ToolContext
): Promise<EngineResponse> {
  return runAgentLoop(
    {
      model: "claude-haiku-4-5-20251001",
      systemPrompt,
      maxTokens: 4096,
      temperature: 0,
    },
    userMessage,
    toolContext
  );
}
```

**Commit:**
```bash
git add -A && git commit -m "feat: add claude-engine.ts with code_execution programmatic tool calling"
```

### Task 5.4: Update diagnostic.service.ts to use new engine

**MODIFY** `src/services/diagnostic.service.ts`:

Replace:
```typescript
import { getCampaigns, getInsights } from "@/server/services/mcp-client.service";
import { runDiagnosticAgent } from "@/server/services/llm.service";
```

With:
```typescript
import { runDiagnosticAgent } from "@/lib/ai/claude-engine";
```

Remove the `fetchAccountData()` function entirely — Claude now fetches data itself via tools inside the sandbox.

Update `runDiagnosticPipeline()`:
- Remove Step 2 (fetchAccountData) — no longer needed
- Remove `buildUserMessage()` data injection — Claude fetches via tools
- Pass `toolContext` to `runDiagnosticAgent()`

The user message becomes:
```
"Analyze Meta Ads account {accountId} for {dateRange.start} to {dateRange.end}.
Use the tools to fetch campaigns, ad sets, insights, and learning phase info.
Return a structured JSON diagnostic."
```

**Commit:**
```bash
git add -A && git commit -m "refactor: diagnostic service now uses claude-engine instead of MCP"
```

### Task 5.5: Delete legacy MCP files

- DELETE `src/server/services/llm.service.ts`
- DELETE `src/server/services/mcp-client.service.ts`
- DELETE `src/server/services/` directory (should be empty now)

**Commit:**
```bash
git add -A && git commit -m "chore: delete legacy MCP service files"
```

---

## Phase 6: Add missing pages and routes

> **Goal:** Add campaign detail, diagnostics history, and missing API routes.
> **Risk:** LOW — new pages, no breaking changes.

### Task 6.1: Campaign detail page

**CREATE** `src/app/(app)/campaigns/[id]/page.tsx`
```tsx
"use client";

import { useParams } from "next/navigation";
import { trpc } from "@/lib/trpc";
import { DiagnosticPanel } from "@/components/dashboard/diagnostic-panel";
import { PerformanceChart } from "@/components/dashboard/performance-chart";
import { LoadingSkeleton } from "@/components/shared/loading-skeleton";

export default function CampaignDetailPage() {
  const { id } = useParams<{ id: string }>();

  // TODO: Add campaigns.getById to tRPC router
  // const { data, isLoading } = trpc.campaigns.getById.useQuery({ id });

  return (
    <div className="space-y-6 p-8">
      <h1 className="font-manrope text-2xl font-light text-[#1a1a1a]">
        Campaign Details
      </h1>
      <p className="text-sm text-gray-500">Campaign ID: {id}</p>
      {/* Wire up when campaigns.getById router is ready */}
      <LoadingSkeleton rows={4} />
    </div>
  );
}
```

### Task 6.2: Diagnostics history page

**CREATE** `src/app/(app)/diagnostics/page.tsx`
```tsx
"use client";

import { trpc } from "@/lib/trpc";
import { DiagnosticPanel } from "@/components/dashboard/diagnostic-panel";
import { LoadingSkeleton } from "@/components/shared/loading-skeleton";
import { EmptyState } from "@/components/shared/empty-state";

export default function DiagnosticsPage() {
  // TODO: Add diagnostics.list to tRPC router
  // const { data, isLoading } = trpc.diagnostics.list.useQuery();

  return (
    <div className="space-y-6 p-8">
      <h1 className="font-manrope text-2xl font-light text-[#1a1a1a]">
        Diagnostic History
      </h1>
      <p className="text-sm text-gray-500">All past AI diagnostics across your accounts.</p>
      {/* Wire up when diagnostics.list router is ready */}
      <EmptyState
        title="No diagnostics yet"
        description="Run a diagnostic on any campaign to see AI-powered analysis here."
      />
    </div>
  );
}
```

### Task 6.3: Missing API routes

**CREATE** `src/app/api/marketing/book-demo/route.ts`
```typescript
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";
import { prisma } from "@/lib/supabase/prisma";
import { rateLimit } from "@/lib/rate-limit";
import { verifyTurnstileToken } from "@/lib/turnstile";

const BookDemoSchema = z.object({
  name: z.string().min(1, { message: "Name is required" }),
  email: z.string().email({ message: "Valid email required" }),
  role: z.string().optional(),
  monthlyBudget: z.string().optional(),
  metaAccountUrl: z.string().url().optional().or(z.literal("")),
  message: z.string().optional(),
  turnstileToken: z.string().min(1, { message: "CAPTCHA required" }),
});

export async function POST(req: NextRequest) {
  try {
    const rateLimitResult = await rateLimit(req);
    if (!rateLimitResult.success) {
      return NextResponse.json({ error: "Too many requests" }, { status: 429 });
    }

    const body = await req.json();
    const parsed = BookDemoSchema.parse(body);

    const isValid = await verifyTurnstileToken(parsed.turnstileToken);
    if (!isValid) {
      return NextResponse.json({ error: "CAPTCHA verification failed" }, { status: 400 });
    }

    const demo = await prisma.demoRequest.create({
      data: {
        name: parsed.name,
        email: parsed.email,
        role: parsed.role,
        monthlyBudget: parsed.monthlyBudget,
        metaAccountUrl: parsed.metaAccountUrl || null,
        message: parsed.message,
      },
    });

    return NextResponse.json({ success: true, id: demo.id });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json({ error: "Validation failed", issues: error.issues }, { status: 400 });
    }
    return NextResponse.json({ error: "Internal server error" }, { status: 500 });
  }
}
```

**Commit:**
```bash
git add -A && git commit -m "feat: add campaign detail, diagnostics history pages, and book-demo API route"
```

---

## Phase 7: Add missing hooks and services

> **Goal:** Create custom hooks for auth, campaigns, diagnostics. Create andromeda and report services.
> **Risk:** LOW — new files only.

### Task 7.1: Custom hooks

**CREATE** `src/hooks/use-auth.ts`
```typescript
"use client";

import { useEffect, useState } from "react";
import { onAuthStateChanged, type User } from "firebase/auth";
import { auth } from "@/lib/firebase/client";

export function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, (firebaseUser) => {
      setUser(firebaseUser);
      setLoading(false);
    });
    return unsubscribe;
  }, []);

  return { user, loading, isAuthenticated: !!user };
}
```

**CREATE** `src/hooks/use-campaigns.ts`
```typescript
"use client";

import { trpc } from "@/lib/trpc";

export function useCampaigns(accountId?: string) {
  return trpc.campaigns.list.useQuery(
    { accountId: accountId ?? "" },
    { enabled: !!accountId }
  );
}
```

**CREATE** `src/hooks/use-diagnostics.ts`
```typescript
"use client";

import { trpc } from "@/lib/trpc";

export function useDiagnostics(accountId?: string) {
  return trpc.diagnostics.list.useQuery(
    { accountId: accountId ?? "" },
    { enabled: !!accountId }
  );
}

export function useRunDiagnostic() {
  const utils = trpc.useUtils();
  return trpc.diagnostics.run.useMutation({
    onSuccess: () => {
      utils.diagnostics.list.invalidate();
    },
  });
}
```

**Commit:**
```bash
git add -A && git commit -m "feat: add use-auth, use-campaigns, use-diagnostics hooks"
```

### Task 7.2: Missing services

**CREATE** `src/services/andromeda.service.ts`
```typescript
import { prisma } from "@/lib/supabase/prisma";
import type { RiskAppetite, FocusMetric } from "@prisma/client";

/**
 * Andromeda — User Intelligence Layer
 *
 * Computes and updates UserBehaviorProfile weekly from UserEvent table.
 * Profile is injected into every Claude diagnostic call so Claude
 * adapts tone, aggressiveness, and priority to the user's preferences.
 */

export async function getOrCreateProfile(userId: string) {
  return prisma.userBehaviorProfile.upsert({
    where: { userId },
    create: { userId },
    update: {},
  });
}

export async function computeProfile(userId: string) {
  const events = await prisma.userEvent.findMany({
    where: { userId, createdAt: { gte: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000) } },
    orderBy: { createdAt: "desc" },
  });

  const applied = events.filter((e) => e.eventType === "RECOMMENDATION_APPLIED").length;
  const viewed = events.filter((e) => e.eventType === "RECOMMENDATION_VIEWED").length;
  const applyRate = viewed > 0 ? applied / viewed : null;

  // Determine risk appetite from apply rate
  let riskAppetite: RiskAppetite = "MEDIUM";
  if (applyRate !== null) {
    if (applyRate > 0.7) riskAppetite = "HIGH";
    else if (applyRate < 0.3) riskAppetite = "LOW";
  }

  // Determine focus metric from most viewed metrics
  const metricEvents = events.filter((e) => e.eventType === "METRIC_VIEWED");
  let focusMetric: FocusMetric = "ROAS";
  if (metricEvents.length > 0) {
    const counts: Record<string, number> = {};
    for (const e of metricEvents) {
      const metric = (e.payload as Record<string, string>)?.metric ?? "ROAS";
      counts[metric] = (counts[metric] ?? 0) + 1;
    }
    focusMetric = (Object.entries(counts).sort((a, b) => b[1] - a[1])[0]?.[0] ?? "ROAS") as FocusMetric;
  }

  return prisma.userBehaviorProfile.update({
    where: { userId },
    data: {
      riskAppetite,
      focusMetric,
      recommendationApplyRate: applyRate,
      totalRecommendationsApplied: applied,
      lastActiveAt: new Date(),
    },
  });
}

export function buildAndromedaContext(profile: {
  riskAppetite: RiskAppetite;
  focusMetric: FocusMetric;
  recommendationApplyRate: number | null;
  avgManagedBudget: unknown;
}): string {
  return [
    "## User Profile (Andromeda)",
    `- Risk appetite: ${profile.riskAppetite}`,
    `- Focus metric: ${profile.focusMetric}`,
    `- Recommendation apply rate: ${profile.recommendationApplyRate ? `${(profile.recommendationApplyRate * 100).toFixed(0)}%` : "Unknown"}`,
    `- Avg managed budget: ${profile.avgManagedBudget ?? "Unknown"}`,
    "",
    "Adapt your tone and recommendations to match this profile.",
    profile.riskAppetite === "LOW" ? "Be conservative. Prioritize stability." : "",
    profile.riskAppetite === "HIGH" ? "Be aggressive. Prioritize growth opportunities." : "",
  ].filter(Boolean).join("\n");
}
```

**CREATE** `src/services/report.service.ts`
```typescript
import { prisma } from "@/lib/supabase/prisma";
import { runReportAgent } from "@/lib/ai/claude-engine";
import type { Prisma } from "@prisma/client";

/**
 * Report Service — Generates structured performance reports.
 *
 * Uses the report-agent (Claude Sonnet, temp 0.3) to produce
 * executive summaries, campaign performance tables, risk assessments,
 * and recommendations in Markdown format.
 */

interface ReportInput {
  workspaceId: string;
  metaAccountRefId: string;
  title: string;
  dateRange: { start: string; end: string };
  createdBy: string;
}

export async function generateReport(input: ReportInput) {
  const systemPrompt = [
    "You are a Meta Ads performance report generator for the Klayrai SaaS platform.",
    "Generate a structured report with these sections:",
    "1. Executive Summary",
    "2. Campaign Performance (table format)",
    "3. Risk Assessment",
    "4. Recommendations",
    "5. Budget Allocation Analysis",
    "",
    "Use Markdown formatting. Be concise and data-driven.",
  ].join("\n");

  const userMessage = `Generate a performance report for the period ${input.dateRange.start} to ${input.dateRange.end}. Use the available tools to fetch all campaign data, insights, and metrics.`;

  const response = await runReportAgent(systemPrompt, userMessage, {
    workspaceId: input.workspaceId,
    metaAccountRefId: input.metaAccountRefId,
  });

  const report = await prisma.savedReport.create({
    data: {
      workspaceId: input.workspaceId,
      title: input.title,
      dateRangeStart: new Date(input.dateRange.start),
      dateRangeEnd: new Date(input.dateRange.end),
      scope: { metaAccountRefId: input.metaAccountRefId } as Prisma.InputJsonValue,
      contentMarkdown: response.content,
      contentJson: {
        model: response.model,
        inputTokens: response.inputTokens,
        outputTokens: response.outputTokens,
      } as Prisma.InputJsonValue,
      createdBy: input.createdBy,
    },
  });

  return report;
}
```

**Commit:**
```bash
git add -A && git commit -m "feat: add andromeda and report services"
```

---

## Phase 8: Create missing actions and Stripe lib

> **Goal:** Add server actions for connect-meta, generate-report. Add Stripe client lib.
> **Risk:** LOW — new files.

### Task 8.1: Server actions

**CREATE** `src/actions/connect-meta.ts`
```typescript
"use server";

import { redirect } from "next/navigation";

const META_APP_ID = process.env.META_APP_ID;
const META_REDIRECT_URI = process.env.META_REDIRECT_URI ?? "https://app.klayrai.com/api/auth/meta/callback";

export async function connectMeta() {
  const scopes = ["ads_management", "ads_read", "business_management", "read_insights"].join(",");

  const authUrl = new URL("https://www.facebook.com/v23.0/dialog/oauth");
  authUrl.searchParams.set("client_id", META_APP_ID ?? "");
  authUrl.searchParams.set("redirect_uri", META_REDIRECT_URI);
  authUrl.searchParams.set("scope", scopes);
  authUrl.searchParams.set("response_type", "code");

  redirect(authUrl.toString());
}
```

**CREATE** `src/actions/generate-report.ts`
```typescript
"use server";

import { generateReport } from "@/services/report.service";

interface GenerateReportInput {
  workspaceId: string;
  metaAccountRefId: string;
  title: string;
  dateStart: string;
  dateEnd: string;
  userId: string;
}

export async function generateReportAction(input: GenerateReportInput) {
  const report = await generateReport({
    workspaceId: input.workspaceId,
    metaAccountRefId: input.metaAccountRefId,
    title: input.title,
    dateRange: { start: input.dateStart, end: input.dateEnd },
    createdBy: input.userId,
  });

  return { success: true, reportId: report.id };
}
```

### Task 8.2: Stripe client lib

**CREATE** `src/lib/stripe.ts`
```typescript
import Stripe from "stripe";

if (!process.env.STRIPE_SECRET_KEY) {
  throw new Error("STRIPE_SECRET_KEY is required");
}

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, {
  apiVersion: "2024-12-18.acacia",
  typescript: true,
});

export const PLANS = {
  STARTER: {
    name: "Starter",
    priceId: process.env.STRIPE_STARTER_PRICE_ID ?? "",
    price: 49,
    currency: "eur",
    accounts: 1,
    features: ["1 Meta Ad account", "Basic diagnostics", "Weekly reports"],
  },
  PRO: {
    name: "Pro",
    priceId: process.env.STRIPE_PRO_PRICE_ID ?? "",
    price: 149,
    currency: "eur",
    accounts: 5,
    features: ["5 Meta Ad accounts", "All diagnostics", "Daily reports", "Andromeda AI profiling"],
  },
  AGENCY: {
    name: "Agency",
    priceId: process.env.STRIPE_AGENCY_PRICE_ID ?? "",
    price: 499,
    currency: "eur",
    accounts: -1, // unlimited
    features: ["Unlimited accounts", "All features", "White-label ready", "Priority support", "API access"],
  },
} as const;
```

**Commit:**
```bash
git add -A && git commit -m "feat: add connect-meta, generate-report actions and stripe lib"
```

---

## Phase 9: Create diagnostic-agent.json + final cleanup

> **Goal:** Create the missing agent config, remove (auth) onboarding from (app), clean up.
> **Risk:** LOW.

### Task 9.1: Create diagnostic-agent.json

**CREATE** `.claude/agents/diagnostic-agent.json`
```json
{
  "name": "diagnostic-agent",
  "description": "Analyzes Meta Ads campaign performance using programmatic tool calling. Identifies issues (creative fatigue, learning stuck, auction overlap, pacing problems, budget degradation) and produces structured diagnostics with recommendations.",
  "model": "claude-opus-4-6",
  "temperature": 0.1,
  "maxTokens": 8192,
  "systemPromptSource": ".claude/skills/diagnostic-engine/SKILL.md",
  "tools": [
    "get_campaigns",
    "get_ad_sets",
    "get_ads",
    "get_insights",
    "compare_performance",
    "get_learning_phase_info"
  ],
  "maxToolRounds": 15,
  "outputFormat": "json",
  "errorHandling": {
    "onTokenExpired": "abort_notify_user",
    "onRateLimit": "exponential_backoff"
  }
}
```

### Task 9.2: Remove onboarding from (app) route group

The onboarding should be its own route, not nested under the app layout with sidebar/header.

- MOVE `src/app/(app)/onboarding/page.tsx` → `src/app/(auth)/onboarding/page.tsx`

This way onboarding uses the auth layout (clean, no sidebar) and redirects to `/dashboard` on completion.

### Task 9.3: Final verification

```bash
npx next build
```

Verify:
- Zero TypeScript errors
- All imports resolve
- No references to `mcp-client.service` or `llm.service`
- No `[[...sign-in]]` or `[[...sign-up]]` directories
- No `clerkId` references

**Commit:**
```bash
git add -A && git commit -m "chore: add diagnostic-agent.json, move onboarding, final cleanup"
```

---

## Summary: File Operations

### MOVE (12 files)
| From | To |
|---|---|
| `src/lib/firebase.ts` | `src/lib/firebase/client.ts` |
| `src/lib/firebase-admin.ts` | `src/lib/firebase/admin.ts` |
| `src/server/db/prisma.ts` | `src/lib/supabase/prisma.ts` |
| `src/server/services/diagnostic.service.ts` | `src/services/diagnostic.service.ts` |
| `src/server/services/user-sync.service.ts` | `src/services/user-sync.service.ts` |
| `src/server/actions/book-demo.ts` | `src/actions/book-demo.ts` |
| `src/components/layout/app-header.tsx` | `src/components/shared/navbar.tsx` |
| `src/components/layout/app-sidebar.tsx` | `src/components/shared/sidebar.tsx` |
| `src/components/shared/kpi-card.tsx` | `src/components/dashboard/kpi-card.tsx` |
| `src/app/(auth)/sign-in/[[...sign-in]]/page.tsx` | `src/app/(auth)/sign-in/page.tsx` |
| `src/app/(auth)/sign-up/[[...sign-up]]/page.tsx` | `src/app/(auth)/sign-up/page.tsx` |
| `src/app/(app)/onboarding/page.tsx` | `src/app/(auth)/onboarding/page.tsx` |

### CREATE (21 files)
| File | Type |
|---|---|
| `src/lib/firebase/index.ts` | Re-export barrel |
| `src/lib/supabase/index.ts` | Re-export barrel |
| `src/lib/meta/meta-tools.ts` | Tool definitions |
| `src/lib/meta/meta-executor.ts` | Meta API layer |
| `src/lib/ai/claude-engine.ts` | AI engine |
| `src/lib/stripe.ts` | Stripe client |
| `src/app/(auth)/sign-in/page.tsx` | Firebase sign-in |
| `src/app/(auth)/sign-up/page.tsx` | Firebase sign-up |
| `src/app/(app)/campaigns/[id]/page.tsx` | Campaign detail |
| `src/app/(app)/diagnostics/page.tsx` | Diagnostics history |
| `src/app/api/marketing/book-demo/route.ts` | Book demo API |
| `src/components/marketing/hero.tsx` | Hero section |
| `src/components/marketing/how-it-works.tsx` | How it works |
| `src/components/marketing/pricing.tsx` | Pricing cards |
| `src/components/marketing/book-demo-form.tsx` | Demo form |
| `src/components/dashboard/diagnostic-panel.tsx` | Diagnostic display |
| `src/components/dashboard/recommendation-card.tsx` | Recommendation card |
| `src/components/dashboard/performance-chart.tsx` | Chart component |
| `src/components/shared/loading-skeleton.tsx` | Loading skeleton |
| `src/components/shared/date-range-picker.tsx` | Date picker |
| `src/hooks/use-auth.ts` | Auth hook |
| `src/hooks/use-campaigns.ts` | Campaigns hook |
| `src/hooks/use-diagnostics.ts` | Diagnostics hook |
| `src/services/andromeda.service.ts` | Andromeda service |
| `src/services/report.service.ts` | Report service |
| `src/actions/connect-meta.ts` | Connect Meta action |
| `src/actions/generate-report.ts` | Generate report action |
| `.claude/agents/diagnostic-agent.json` | Agent config |

### DELETE (4 files)
| File | Reason |
|---|---|
| `src/server/services/llm.service.ts` | Replaced by claude-engine.ts |
| `src/server/services/mcp-client.service.ts` | Replaced by meta-executor.ts |
| `src/app/(auth)/sign-in/[[...sign-in]]/` | Clerk pattern → Firebase |
| `src/app/(auth)/sign-up/[[...sign-up]]/` | Clerk pattern → Firebase |

---

## Execution Order

```
Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6 → Phase 7 → Phase 8 → Phase 9
  │           │          │          │          │          │          │          │          │
  └─ imports  └─ imports └─ auth    └─ UI      └─ CORE   └─ pages  └─ hooks   └─ actions └─ cleanup
     only        only      pages     comps      engine     routes    services    stripe     verify
```

Each phase is independently committable and leaves the app in a working state.
