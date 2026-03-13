# Klayrai — Search Strategy

## V1: Postgres Full-Text Search (zero extra service, zero extra cost)

### What is searchable in V1

| Entity | Searchable Fields | Data Source |
|--------|------------------|-------------|
| Diagnostics | entityName, diagnosisText | Postgres snapshots |
| Saved Reports | title | Postgres |
| Meta campaigns (live) | name, objective | MCP via get_campaigns (not indexed) |

### Implementation pattern (tRPC + Prisma)

```typescript
// src/server/trpc/routers/diagnostics.ts

search: protectedProcedure
  .input(z.object({
    query: z.string().min(1).max(100),
    workspaceId: z.string().cuid(),
    entityType: z.enum(['CAMPAIGN', 'AD_SET', 'AD', 'ACCOUNT']).optional(),
    riskLevel: z.enum(['LOW', 'MEDIUM', 'HIGH', 'CRITICAL']).optional(),
  }))
  .query(async ({ ctx, input }) => {
    return ctx.db.diagnostic.findMany({
      where: {
        workspaceId: input.workspaceId,
        ...(input.entityType && { entityType: input.entityType }),
        ...(input.riskLevel && { riskLevel: input.riskLevel }),
        OR: [
          { entityName: { contains: input.query, mode: 'insensitive' } },
          { diagnosisText: { contains: input.query, mode: 'insensitive' } },
        ],
      },
      include: { recommendations: true },
      orderBy: { createdAt: 'desc' },
      take: 20,
    });
  });
Available Filters (V1 — all via Prisma WHERE clauses)
Filter	Field	Options
Entity type	entityType	CAMPAIGN, AD_SET, AD, ACCOUNT
Risk level	riskLevel	LOW, MEDIUM, HIGH, CRITICAL
Diagnostic flag	flags (JSONB)	learning_stuck, creative_fatigue, pacing_issue, auction_overlap, budget_degradation
Date range	createdAt	gte + lte datetime
Recommendation priority	recommendations.priority	LOW, MEDIUM, HIGH, URGENT
Filtering by JSONB flag example
typescript
// Filter diagnostics where learning_stuck is true
const flagFilter = {
  flags: {
    path: ['learning_stuck'],
    equals: true
  }
}
V2: Typesense Cloud (when Postgres FTS becomes insufficient)
When to upgrade from Postgres FTS to Typesense
Users have more than 500 diagnostic snapshots and search feels slow

Users request typo tolerance when searching campaign names

You need multi-facet filtering (risk level + entity type + date range simultaneously)

You need search ranking by relevance rather than just date

Why Typesense over Algolia
Open source and self-hostable (or Typesense Cloud for managed)

Significantly cheaper than Algolia at scale

Excellent typo tolerance out of the box

Faceted filtering is first-class

Typesense collection schema (V2)
json
{
  "name": "diagnostics",
  "fields": [
    { "name": "id", "type": "string" },
    { "name": "entityName", "type": "string" },
    { "name": "diagnosisText", "type": "string" },
    { "name": "entityType", "type": "string", "facet": true },
    { "name": "riskLevel", "type": "string", "facet": true },
    { "name": "workspaceId", "type": "string", "facet": true },
    { "name": "flagLearningStuck", "type": "bool", "facet": true },
    { "name": "flagCreativeFatigue", "type": "bool", "facet": true },
    { "name": "flagBudgetDegradation", "type": "bool", "facet": true },
    { "name": "createdAt", "type": "int64" }
  ],
  "default_sorting_field": "createdAt"
}
Sync strategy for Typesense (V2)
After every diagnostic saved to Postgres, push to Typesense via background job

Index only diagnostics from the last 90 days to keep index lean

workspaceId facet ensures workspace isolation at the Typesense query level

V3: Semantic Search via RAG
Enable PgVector extension on Neon

Generate embeddings on diagnosisText + recommendation descriptions using text-embedding-3-small

Store in DiagnosticEmbedding table (vector column)

Query: natural language search like "show me campaigns with learning issues in high budgets"

Embed the query text

Run vector similarity search: ORDER BY embedding <=> $queryEmbedding LIMIT 10

Return ranked diagnostics by semantic similarity

Also enables: "find past campaigns similar to this underperformer" for benchmarking