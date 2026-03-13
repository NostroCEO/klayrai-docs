# FILE 2 — `/docs/roadmap.md`

```markdown
# Klayrai — Roadmap

## Versioning Philosophy
- V1: Ship fast, validate core value, learn from real users. MCP-first architecture.
- V2: Deepen intelligence. Multi-agent with LangGraph. RAG. Multi-account agency features.
- V3: Platform play. Write-back to Meta. Multi-channel. White-label for agencies.

---

## V1 — MVP (8 Weeks)

### Week 1 — Foundation
- [ ] Init monorepo: Next.js 14 App Router + TypeScript strict mode
- [ ] Install and configure Tailwind CSS v3 and shadcn/ui
- [ ] Install and configure Firebase Auth (email/password + Google OAuth)
- [ ] Create Supabase Postgres project and initialize Prisma v5
- [ ] Apply v1 lean schema migrations
- [ ] Configure tRPC v11 (server router + client + Next.js App Router adapter)
- [ ] Set up Zustand store skeleton (ui-store.ts)
- [ ] Initialize Sentry (client DSN + server DSN)
- [ ] Initialize PostHog
- [ ] Link Vercel project and set all env vars in Vercel Env Manager
- [ ] Fork brijr/meta-mcp into klayrai-meta-mcp repo
- [ ] Rename MCP, update package.json and README
- [ ] Configure MCP env vars for Klayrai Meta App
- [ ] Register MCP in Claude Code CLI (claude mcp add klayrai-meta-mcp)
- [ ] Confirm MCP health_check returns OK
- [ ] Create all .claude/skills/ SKILL.md skeletons and commit
- [ ] Write .env.example with all variable names (no values)
- [ ] Write README.md with setup instructions

### Week 2 — Landing Page (www.klayrai.com)
- [ ] Hero section: headline, subcopy, dual CTA (Start free trial + Book free demo)
- [ ] Why your Meta campaigns underperform section (educational: learning phase, fatigue, overlap)
- [ ] How it works: 3-step visual section
- [ ] Pricing section: Starter EUR 49/mo, Pro EUR 149/mo, and Agency EUR 499/mo
- [ ] Book free demo section:
  - [ ] Form fields: name, email, role, monthly budget, Meta account URL, message
  - [ ] Cloudflare Turnstile CAPTCHA integration
  - [ ] Server Action with Zod validation on every field
  - [ ] Save submission to DemoRequest table in Postgres
  - [ ] Send confirmation email via Resend + React Email template
- [ ] Footer: Privacy Policy, Terms of Service, social links
- [ ] SEO: meta tags, OG image, favicon, sitemap.xml
- [ ] Lighthouse score above 90 confirmed

### Week 3 — klayrai-meta-mcp Build and Validation
- [ ] Clone and configure forked MCP with production Meta App credentials
- [ ] Test all 25 MCP tools via Claude Code interactive prompts
- [ ] Validate core tools in detail:
  - [ ] health_check returns OK with valid credentials
  - [ ] get_ad_accounts returns correct ad accounts list
  - [ ] get_campaigns works with status filter (ACTIVE, PAUSED, ALL)
  - [ ] get_ad_sets returns learning_status and bid_strategy
  - [ ] get_insights returns: spend, impressions, clicks, CTR, CPC, CPA, ROAS, frequency, reach, quality_ranking
  - [ ] compare_performance works across 2 or more campaigns
  - [ ] export_insights returns valid JSON
- [ ] Document all tool I/O schemas in .claude/skills/diagnostic-engine/SKILL.md
- [ ] Deploy MCP server to Render.com (always-on Node process)
- [ ] Confirm MCP reachable from Vercel environment

### Week 4 — App Shell and Onboarding (app.klayrai.com)
- [ ] Configure PWA manifest.json (name, icons, theme_color, display: standalone, start_url: /dashboard)
- [ ] Install and configure next-pwa for service worker and offline shell
- [ ] Set up Next.js App Router route groups: (auth), (app)
- [ ] Configure Firebase Auth middleware to protect all (app) routes
- [ ] Set up tRPC context to resolve userId and workspaceId from Firebase Auth ID token
- [ ] Build onboarding flow (3 steps):
  - [ ] Step 1: Connect Meta account (OAuth flow via Meta App)
  - [ ] Step 2: Select ad accounts to track (stored as MetaAccountRef in DB)
  - [ ] Step 3: Confirm setup and run MCP health_check live
  - [ ] Empty state screen with clear CTA on every step
- [ ] Build app shell: sidebar navigation + header with user menu and workspace switcher
- [ ] Dashboard home page: KPI cards (total spend, avg ROAS, avg CPA, active campaigns count)
- [ ] Campaigns list page: table with status filter, date range filter, text search
- [ ] Ad Sets page per campaign (triggered via MCP get_ad_sets)
- [ ] Ads page per ad set (triggered via MCP get_ads)
- [ ] Global date range picker component (last 7d, 14d, 30d, custom)
- [ ] All data fetched via diagnostic-agent → MCP (zero direct Meta calls from Next.js)

### Week 5 — Diagnostic Engine
- [ ] Build deterministic rules engine in diagnostic.service.ts:
  - [ ] Learning stuck: learning_status === LEARNING_LIMITED AND conversions_7d < 50
  - [ ] Creative fatigue: frequency > 3.0 AND CTR trending down more than 20% over 7 days
  - [ ] Auction overlap: multiple ad sets in same campaign with overlapping audience and placement
  - [ ] Pacing issue: (spend_today / daily_budget) / (elapsed_hours / 24) outside range 0.75 to 1.25
  - [ ] Budget degradation: ad set spend_7d > account average * 1.5 AND ROAS < account average * 0.8
- [ ] Build LLM diagnostic layer in llm.service.ts:
  - [ ] Input: mcp_insights_json + flags + user_behavior_profile
  - [ ] Output: diagnosis_text + causes[] + recommendations[] + risk_level
  - [ ] Model: claude-opus-4-5, temperature: 0.3, max_tokens: 2000
- [ ] Finalize .claude/skills/diagnostic-engine/SKILL.md
- [ ] Build DiagnosticPanel component: shows flags, diagnosis text, causes
- [ ] Build RecommendationCard component: shows title, description, priority, predicted effect
- [ ] Build DegradationPanel component: explains why Meta is still spending on underperformer
- [ ] Show diagnostics inline on Campaign and Ad Set detail pages
- [ ] Save diagnostic snapshots to diagnostics and recommendations tables in Postgres

### Week 6 — Andromeda (User Behavior Profiling)
- [ ] Implement internal event tracking (UserEvent table):
  - [ ] FILTER_APPLIED, METRIC_VIEWED
  - [ ] RECOMMENDATION_VIEWED, RECOMMENDATION_APPLIED, RECOMMENDATION_DISMISSED
  - [ ] REPORT_GENERATED, REPORT_EXPORTED
  - [ ] CAMPAIGN_OPENED, AD_SET_OPENED
  - [ ] META_ACCOUNT_CONNECTED, ONBOARDING_STEP_COMPLETED
- [ ] Build andromeda.service.ts:
  - [ ] Weekly Vercel Cron job to recompute UserBehaviorProfile from events
  - [ ] Compute: risk_appetite, focus_metric, preferred_actions, avg_budget_managed, recommendation_apply_rate
- [ ] Inject user_behavior_profile into every LLM and agent call as context
- [ ] Finalize .claude/skills/andromeda/SKILL.md
- [ ] Verify per-client tone adaptation works: conservative language for LOW risk, decisive for HIGH risk
- [ ] Verify focus metric changes recommendation priority ordering

### Week 7 — Reports and Insights Page
- [ ] Build Insights page: cross-campaign performance comparison, current period vs previous
- [ ] Build report-agent integration in report.service.ts
- [ ] Generate report structure via Claude report-agent:
  - [ ] Executive summary
  - [ ] Performance overview table (spend, ROAS, CPA, CTR vs previous period)
  - [ ] Diagnostics section (all flagged entities)
  - [ ] Recommendations section (prioritized list)
  - [ ] 7-day next steps plan
- [ ] Export to Markdown: copy to clipboard button
- [ ] Export to PDF: @react-pdf/renderer implementation
- [ ] Save reports to saved_reports table in Postgres
- [ ] Build Report history page with list of past reports
- [ ] Finalize .claude/skills/report-generator/SKILL.md

### Week 8 — Stripe, Polish, and Launch
- [ ] Create Stripe products: Starter EUR 49/mo, Pro EUR 149/mo, Agency EUR 499/mo
- [ ] Implement Stripe Billing with subscription flow
- [ ] Handle Stripe webhooks at /api/webhooks/stripe:
  - [ ] customer.subscription.created
  - [ ] customer.subscription.updated
  - [ ] customer.subscription.deleted
- [ ] Gate features by plan in tRPC middleware:
  - [ ] Starter: 1 Meta ad account, 6h sync, basic reports
  - [ ] Pro: 10 Meta ad accounts, 1h sync, full reports + PDF export
  - [ ] Agency: unlimited Meta ad accounts, 30min sync, full reports + PDF export + multi-workspace
- [ ] Add Stripe Customer Portal for self-serve plan management
- [ ] Finalize .claude/skills/stripe/SKILL.md
- [ ] Add onboarding tooltips on first 5 user interactions
- [ ] Add empty states on every page and every route
- [ ] Run full Lighthouse audit: landing above 90, app above 80
- [ ] Run full security checklist from CLAUDE.md
- [ ] Run npm audit with zero critical vulnerabilities
- [ ] Finalize README.md
- [ ] LAUNCH

---

## V2 — Intelligence Layer (target: 8 weeks post-launch)
- [ ] LangGraph multi-agent: diagnostic-agent + strategy-agent + report-agent as separate graph nodes
- [ ] PgVector extension on Supabase: embeddings on historical diagnostic snapshots
- [ ] Best Ads Matches engine: vector similarity search on top-performing campaigns
- [ ] Multi-user workspace: invite members with roles (Owner, Analyst, Viewer)
- [ ] Automated alerts: email and in-app notification when a diagnostic flag fires
- [ ] Typesense Cloud: full-text search with typo tolerance on campaigns and diagnostics
- [ ] Multi-account aggregated view: agency-level dashboard across all workspaces
- [ ] Bulk actions: pause all underperformers in one click via MCP write tools
- [ ] Slack notifications integration
- [ ] GitHub Actions CI: npm audit + TypeScript strict check on every PR

## V3 — Platform and Automation (target: V2 + 12 weeks)
- [ ] Meta API write-back via MCP create and update tools (user-approved actions)
- [ ] TikTok Ads API integration via new klayrai-tiktok-mcp
- [ ] Google Ads integration via new klayrai-google-mcp
- [ ] AI-driven campaign brief generator (structure, copy, audience, budget recommendations)
- [ ] White-label for agencies (custom domain, custom branding per workspace)
- [ ] React Native Expo mobile app
- [ ] ClickHouse for time-series analytics at scale (separate from Postgres)
