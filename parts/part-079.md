# Part 079 — Enterprise GraphQL: Multi-Team Governance 🏢

> **ระดับ:** World-Class | **เวลาเรียน:** 85 นาที | **ขั้นตอนที่:** 2051–2090

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Schema ownership model
- GraphQL Guild model
- RFC process สำหรับ schema changes
- Schema review process
- Cross-team coordination
- Client registry
- Operation analytics
- Schema governance tooling

---

## 📌 Step 2051: Schema Ownership Model

```
Organization Structure:

Platform Team (owns Router & Gateway):
- Apollo Router configuration
- Authentication/Authorization policies
- Performance monitoring
- Schema registry

Domain Teams (own subgraphs):
- Users Team → users subgraph
- Products Team → products subgraph
- Orders Team → orders subgraph
- Reviews Team → reviews subgraph

Rules:
1. Team เพิ่ม field ใน subgraph ตัวเองได้ (no review needed)
2. Breaking changes → RFC process
3. Cross-team field additions → peer review
4. Security-sensitive changes → Platform Team sign-off
```

---

## 📌 Step 2052: RFC Process

```markdown
# RFC-0042: Add Seller Performance Metrics

## Summary
เพิ่ม `sellerMetrics` field ใน User type เพื่อแสดงข้อมูล performance ของ sellers

## Motivation
Seller dashboard ต้องการ metrics: average rating, response rate, fulfillment rate

## Proposed Changes

```graphql
type User {
  # เพิ่ม field ใหม่ (non-breaking)
  sellerMetrics: SellerMetrics  # nullable: ไม่ใช่ทุก user เป็น seller
}

type SellerMetrics {
  averageRating: Float!
  responseRate: Float!      # 0.0 - 1.0
  fulfillmentRate: Float!
  totalSales: Int!
  reviewCount: Int!
}
```

## Impact Analysis
- Clients affected: none (new field)
- Performance: requires join กับ orders table (N+1 protected by DataLoader)
- Cache: maxAge: 3600 (recalculate hourly)

## Implementation Plan
1. Users team: เพิ่ม sellerMetrics field
2. Orders team: expose metrics endpoint
3. Reviews team: expose aggregate ratings

## Reviewers
- [x] Platform Team: @platform-team
- [x] Orders Team: @orders-team
- [x] Reviews Team: @reviews-team

## Status: APPROVED 2025-01-15
```

---

## 📌 Step 2053: Client Registry

```typescript
// src/analytics/client-registry.ts
// Track ว่า clients ไหนใช้ operations ไหนบ้าง

interface ClientOperation {
  clientName: string;
  clientVersion: string;
  operationName: string;
  operationHash: string;
  fieldsUsed: string[];
  lastSeen: Date;
  callCount: number;
}

// Middleware: extract client info
app.use('/graphql', (req, res, next) => {
  const clientName = req.headers['apollographql-client-name'] as string ?? 'unknown';
  const clientVersion = req.headers['apollographql-client-version'] as string ?? 'unknown';
  
  req.clientInfo = { clientName, clientVersion };
  next();
});

// Plugin: track operations
const clientRegistryPlugin = (): ApolloServerPlugin => ({
  async requestDidStart({ request }) {
    const { clientName, clientVersion } = (request.http?.headers as Record<string, string>) ?? {};
    
    return {
      async executionDidStart() {
        const fieldsUsed: string[] = [];
        
        return {
          willResolveField({ info }) {
            fieldsUsed.push(`${info.parentType.name}.${info.fieldName}`);
          },
          
          async executionDidEnd() {
            if (request.operationName) {
              // Store in analytics DB
              await analyticsDb.trackOperation({
                clientName: clientName ?? 'unknown',
                clientVersion: clientVersion ?? 'unknown',
                operationName: request.operationName,
                operationHash: hashOperation(request.query ?? ''),
                fieldsUsed: [...new Set(fieldsUsed)],
                timestamp: new Date(),
              });
            }
          },
        };
      },
    };
  },
});

// API: ดู field usage
app.get('/admin/field-usage/:field', requireAdmin, async (req, res) => {
  const { field } = req.params;  // e.g., "Product.price"
  
  const usage = await analyticsDb.getFieldUsage(field, {
    since: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000), // last 30 days
  });
  
  res.json({
    field,
    totalCalls: usage.totalCalls,
    uniqueClients: usage.uniqueClients,
    clients: usage.clientBreakdown,
    safeToRemove: usage.totalCalls === 0,
  });
});
```

---

## 📌 Step 2054: Schema Review Automation

```typescript
// .github/schema-review/rules.ts
// Automated schema review rules

interface SchemaChange {
  type: string;
  description: string;
  path: string;
}

interface ReviewResult {
  approved: boolean;
  comments: string[];
  requiresManualReview: boolean;
}

const rules = [
  // Rule 1: เพิ่ม field ต้องมี description
  {
    name: 'require-field-description',
    check: (change: SchemaChange) => {
      if (change.type === 'FIELD_ADDED' && !change.description.includes('description')) {
        return {
          approved: false,
          comment: `Field ${change.path} requires a description`,
        };
      }
      return { approved: true };
    },
  },
  
  // Rule 2: Breaking changes ต้องมี manual review
  {
    name: 'breaking-change-requires-review',
    check: (change: SchemaChange) => {
      const breakingTypes = ['FIELD_REMOVED', 'TYPE_REMOVED', 'ENUM_VALUE_REMOVED'];
      if (breakingTypes.includes(change.type)) {
        return {
          approved: false,
          requiresManualReview: true,
          comment: `Breaking change: ${change.description} requires manual approval`,
        };
      }
      return { approved: true };
    },
  },
  
  // Rule 3: ตรวจสอบ naming conventions
  {
    name: 'naming-conventions',
    check: (change: SchemaChange) => {
      if (change.type === 'FIELD_ADDED') {
        const fieldName = change.path.split('.').pop() ?? '';
        if (/[A-Z]/.test(fieldName[0] ?? '')) {
          return {
            approved: false,
            comment: `Field names must be camelCase: ${fieldName}`,
          };
        }
      }
      return { approved: true };
    },
  },
];
```

---

## 📌 Step 2055: สรุป Part 079

### เนื้อหาที่เรียนรู้

✅ Schema ownership model  
✅ RFC process สำหรับ schema changes  
✅ Client registry  
✅ Field usage tracking  
✅ Automated review rules  

### Governance Maturity Model

```
Level 1 (Ad-hoc):
- ไม่มี process
- Breaking changes เกิดขึ้นบ่อย
- ไม่รู้ว่า clients ใช้อะไรบ้าง

Level 2 (Defined):
- Team owners ชัดเจน
- Breaking changes ต้องมี review
- Basic analytics

Level 3 (Managed):
- RFC process
- Client registry
- Schema registry (Hive/Studio)
- Automated breaking change detection

Level 4 (Optimizing):
- Field-level analytics
- Proactive client migration
- SLO-driven schema design
- Schema as product
```

### ในส่วนถัดไป

➡️ **[Part 080](./part-080.md)** — GraphQL Client Patterns: React & Beyond

---

*Part 079 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
