# Klayrai — Full Database Schema

## Design Principles
- V1 schema is lean: MCP is the source of truth for live Meta data
- Postgres stores: users, workspaces, billing, diagnostic snapshots, Andromeda profiles, events, reports, demo requests
- Campaign and AdSet hierarchy is NOT stored in V1 — they live in Meta via MCP
- V2 adds full Campaign/AdSet/Ad/PerformanceStat tables for historical analytics

## schema.prisma (copy this entire file to /prisma/schema.prisma)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DATABASE_URL_UNPOOLED")
}

// ─────────────────────────────────────────────────────
// USERS AND WORKSPACES
// ─────────────────────────────────────────────────────

model User {
  id               String    @id @default(cuid())
  firebaseUid          String    @unique
  email            String    @unique
  name             String?
  avatarUrl        String?
  createdAt        DateTime  @default(now())
  updatedAt        DateTime  @updatedAt

  workspaceMembers WorkspaceMember[]
  behaviorProfile  UserBehaviorProfile?
  events           UserEvent[]

  @@index([firebaseUid])
  @@index([email])
}

model Workspace {
  id                   String      @id @default(cuid())
  name                 String
  slug                 String      @unique
  stripeCustomerId     String?     @unique
  stripeSubscriptionId String?     @unique
  plan                 Plan        @default(STARTER)
  planStatus           PlanStatus  @default(TRIALING)
  trialEndsAt          DateTime?
  createdAt            DateTime    @default(now())
  updatedAt            DateTime    @updatedAt

  members              WorkspaceMember[]
  metaAccountRefs      MetaAccountRef[]
  diagnostics          Diagnostic[]
  savedReports         SavedReport[]
  demoRequests         DemoRequest[]

  @@index([slug])
  @@index([stripeCustomerId])
}

model WorkspaceMember {
  id          String         @id @default(cuid())
  workspaceId String
  userId      String
  role        WorkspaceRole  @default(MEMBER)
  createdAt   DateTime       @default(now())

  workspace   Workspace      @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  user        User           @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([workspaceId, userId])
  @@index([workspaceId])
  @@index([userId])
}

enum WorkspaceRole {
  OWNER
  ADMIN
  ANALYST
  VIEWER
}

enum Plan {
  STARTER
  PRO
  AGENCY
}

enum PlanStatus {
  TRIALING
  ACTIVE
  PAST_DUE
  CANCELLED
}

// ─────────────────────────────────────────────────────
// META ACCOUNT REFERENCES
// No tokens here — MCP handles all Meta auth
// ─────────────────────────────────────────────────────

model MetaAccountRef {
  id              String    @id @default(cuid())
  workspaceId     String
  metaAccountId   String
  name            String
  currency        String    @default("USD")
  timezone        String    @default("UTC")
  isActive        Boolean   @default(true)
  lastDiagnosedAt DateTime?
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  workspace       Workspace    @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  diagnostics     Diagnostic[]

  @@unique([workspaceId, metaAccountId])
  @@index([workspaceId])
}

// ─────────────────────────────────────────────────────
// LEAD MAGNET
// ─────────────────────────────────────────────────────

model DemoRequest {
  id             String     @id @default(cuid())
  workspaceId    String?
  name           String
  email          String
  role           String?
  monthlyBudget  String?
  metaAccountUrl String?
  message        String?
  status         DemoStatus @default(PENDING)
  createdAt      DateTime   @default(now())

  workspace      Workspace? @relation(fields: [workspaceId], references: [id])

  @@index([email])
  @@index([status])
}

enum DemoStatus {
  PENDING
  CONTACTED
  CONVERTED
  REJECTED
}

// ─────────────────────────────────────────────────────
// DIAGNOSTICS AND RECOMMENDATIONS
// Snapshots from Claude agent processing MCP data
// ─────────────────────────────────────────────────────

model Diagnostic {
  id               String         @id @default(cuid())
  workspaceId      String
  metaAccountRefId String
  entityType       EntityType
  metaCampaignId   String?
  metaAdSetId      String?
  metaAdId         String?
  entityName       String?
  dateRangeStart   DateTime       @db.Date
  dateRangeEnd     DateTime       @db.Date
  flags            Json
  diagnosisText    String?        @db.Text
  causesJson       Json?
  riskLevel        RiskLevel      @default(MEDIUM)
  llmModel         String?
  llmInputTokens   Int?
  llmOutputTokens  Int?
  rawContextJson   Json?
  createdAt        DateTime       @default(now())

  workspace        Workspace      @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  metaAccountRef   MetaAccountRef @relation(fields: [metaAccountRefId], references: [id])
  recommendations  Recommendation[]

  @@index([workspaceId])
  @@index([metaAccountRefId])
  @@index([entityType, metaCampaignId])
  @@index([createdAt])
}

model Recommendation {
  id              String               @id @default(cuid())
  diagnosticId    String
  title           String
  description     String               @db.Text
  actionType      RecommendationAction
  priority        Priority             @default(MEDIUM)
  predictedEffect Json?
  isApplied       Boolean              @default(false)
  appliedAt       DateTime?
  createdAt       DateTime             @default(now())

  diagnostic      Diagnostic           @relation(fields: [diagnosticId], references: [id], onDelete: Cascade)

  @@index([diagnosticId])
  @@index([priority])
}

enum EntityType {
  CAMPAIGN
  AD_SET
  AD
  ACCOUNT
}

enum RiskLevel {
  LOW
  MEDIUM
  HIGH
  CRITICAL
}

enum Priority {
  LOW
  MEDIUM
  HIGH
  URGENT
}

enum RecommendationAction {
  INCREASE_BUDGET
  DECREASE_BUDGET
  PAUSE_AD_SET
  PAUSE_CAMPAIGN
  REFRESH_CREATIVE
  EXPAND_AUDIENCE
  NARROW_AUDIENCE
  CONSOLIDATE_AD_SETS
  RESET_LEARNING
  ADJUST_BID
  TEST_NEW_CREATIVE
  CHANGE_OPTIMIZATION_GOAL
  DUPLICATE_WINNER
  OTHER
}

// ─────────────────────────────────────────────────────
// ANDROMEDA — USER BEHAVIOR PROFILES
// ─────────────────────────────────────────────────────

model UserBehaviorProfile {
  id                          String       @id @default(cuid())
  userId                      String       @unique
  riskAppetite                RiskAppetite @default(MEDIUM)
  focusMetric                 FocusMetric  @default(ROAS)
  avgManagedBudget            Decimal?     @db.Decimal(12, 2)
  preferredActions            Json?
  totalReportsGenerated       Int          @default(0)
  totalRecommendationsApplied Int          @default(0)
  recommendationApplyRate     Float?
  mostUsedFilters             Json?
  lastActiveAt                DateTime?
  createdAt                   DateTime     @default(now())
  updatedAt                   DateTime     @updatedAt

  user                        User         @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
}

enum RiskAppetite {
  LOW
  MEDIUM
  HIGH
}

enum FocusMetric {
  ROAS
  CPA
  CPM
  CTR
  VOLUME
  REACH
}

// ─────────────────────────────────────────────────────
// USER EVENTS — feeds Andromeda computation
// ─────────────────────────────────────────────────────

model UserEvent {
  id        String    @id @default(cuid())
  userId    String
  eventType EventType
  payload   Json?
  sessionId String?
  createdAt DateTime  @default(now())

  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId, eventType])
  @@index([createdAt])
}

enum EventType {
  FILTER_APPLIED
  METRIC_VIEWED
  RECOMMENDATION_VIEWED
  RECOMMENDATION_APPLIED
  RECOMMENDATION_DISMISSED
  REPORT_GENERATED
  REPORT_EXPORTED
  CAMPAIGN_OPENED
  AD_SET_OPENED
  SYNC_TRIGGERED
  META_ACCOUNT_CONNECTED
  ONBOARDING_STEP_COMPLETED
  MCP_HEALTH_CHECKED
}

// ─────────────────────────────────────────────────────
// SAVED REPORTS
// ─────────────────────────────────────────────────────

model SavedReport {
  id              String    @id @default(cuid())
  workspaceId     String
  title           String
  dateRangeStart  DateTime  @db.Date
  dateRangeEnd    DateTime  @db.Date
  scope           Json
  contentMarkdown String    @db.Text
  contentJson     Json
  createdBy       String
  createdAt       DateTime  @default(now())

  workspace       Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)

  @@index([workspaceId])
  @@index([createdAt])
}
```

## Row Level Security (run these SQL statements after migration)

```sql
ALTER TABLE "Diagnostic" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "Recommendation" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "SavedReport" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "MetaAccountRef" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "UserBehaviorProfile" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "UserEvent" ENABLE ROW LEVEL SECURITY;

CREATE POLICY workspace_isolation_diagnostics ON "Diagnostic"
  USING (
    "workspaceId" IN (
      SELECT "workspaceId" FROM "WorkspaceMember"
      WHERE "userId" = current_setting('app.current_user_id')
    )
  );

CREATE POLICY workspace_isolation_meta_accounts ON "MetaAccountRef"
  USING (
    "workspaceId" IN (
      SELECT "workspaceId" FROM "WorkspaceMember"
      WHERE "userId" = current_setting('app.current_user_id')
    )
  );

CREATE POLICY workspace_isolation_reports ON "SavedReport"
  USING (
    "workspaceId" IN (
      SELECT "workspaceId" FROM "WorkspaceMember"
      WHERE "userId" = current_setting('app.current_user_id')
    )
  );

CREATE POLICY user_isolation_behavior_profile ON "UserBehaviorProfile"
  USING ("userId" = current_setting('app.current_user_id'));

CREATE POLICY user_isolation_events ON "UserEvent"
  USING ("userId" = current_setting('app.current_user_id'));
```

## V2 Schema Additions (do not build yet — plan ahead)

```prisma
// V2: Full Meta hierarchy for historical analytics
model Campaign {
  id             String     @id @default(cuid())
  workspaceId    String
  metaAccountId  String
  metaCampaignId String
  name           String
  status         MetaStatus
  objective      String?
  buyingType     String?
  dailyBudget    Decimal?
  createdAt      DateTime   @default(now())
  updatedAt      DateTime   @updatedAt
}

model AdSet {
  id             String         @id @default(cuid())
  campaignId     String
  metaAdSetId    String
  name           String
  status         MetaStatus
  dailyBudget    Decimal?
  bidStrategy    String?
  learningStatus LearningStatus?
  targetingJson  Json?
  createdAt      DateTime       @default(now())
  updatedAt      DateTime       @updatedAt
}

model Ad {
  id           String     @id @default(cuid())
  adSetId      String
  metaAdId     String
  name         String
  status       MetaStatus
  creativeType String?
  createdAt    DateTime   @default(now())
  updatedAt    DateTime   @updatedAt
}

model PerformanceStat {
  id              String     @id @default(cuid())
  entityType      EntityType
  campaignId      String?
  adSetId         String?
  adId            String?
  date            DateTime   @db.Date
  impressions     BigInt     @default(0)
  reach           BigInt     @default(0)
  clicks          BigInt     @default(0)
  spend           Decimal    @default(0) @db.Decimal(12, 2)
  conversions     Int        @default(0)
  conversionValue Decimal    @default(0) @db.Decimal(12, 2)
  frequency       Decimal    @default(0) @db.Decimal(6, 4)
  ctr             Decimal    @default(0) @db.Decimal(8, 6)
  cpc             Decimal    @default(0) @db.Decimal(10, 4)
  cpm             Decimal    @default(0) @db.Decimal(10, 4)
  cpa             Decimal    @default(0) @db.Decimal(10, 4)
  roas            Decimal    @default(0) @db.Decimal(10, 4)
  createdAt       DateTime   @default(now())
}

// V2: RAG embeddings
model DiagnosticEmbedding {
  id           String   @id @default(cuid())
  diagnosticId String   @unique
  embedding    Unsupported("vector(1536)")
  createdAt    DateTime @default(now())
}
```
