# Part 096 — Case Study: World-Class Architecture 🌟

> **ระดับ:** World-Class | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 2796–2840

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Netflix DGS (Domain Graph Service)
- GitHub's GraphQL architecture
- Shopify's API evolution
- Twitter/X API considerations
- Airbnb's schema design
- Lessons learned จาก scale
- Patterns ที่ world-class companies ใช้

---

## 📌 Step 2796: Netflix DGS Framework

```
Netflix DGS (Domain Graph Service):
- Kotlin/Java framework สำหรับ GraphQL
- ใช้ Spring Boot
- Schema-first
- Modular federation

Architecture:
  Client → Studio Edge (Gateway) → DGS Services
  
  Studio Edge:
  - Routes queries
  - Caches responses
  - Auth enforcement
  - Rate limiting

Key Patterns Netflix ใช้:
1. Operation Tracing: ทุก operation มี trace ID
2. Schema Registry: Federated schema management
3. Field Isolation: each team owns fields
4. Safe Defaults: nullable by default
5. Backward Compatibility: 6-month deprecation window

Netflix Scale:
- 200M+ subscribers
- 1000s of engineers
- Hundreds of DGS services
- Billions of requests/day

Lessons:
□ GraphQL จัดการ complexity ได้ดีกว่า REST ที่ scale นี้
□ Federation ให้ team autonomy โดยไม่ต้องรอ API gateway team
□ Schema evolution เป็น product feature ไม่ใช่ technical debt
```

---

## 📌 Step 2797: GitHub's GraphQL API

```
GitHub launched GraphQL API v4 (2016):
- แทน REST API v3
- Reason: clients ต้องการ very different data combinations

Architecture:
  GitHub REST API (legacy) → still maintained
  GitHub GraphQL API v4    → primary API สำหรับ integrations

Key Design Decisions:
1. Node Interface: ทุก object implement Node
   interface Node { id: ID! }
   → ทุก ID globally unique
   → Relay compatibility

2. Connection Pattern everywhere:
   repository.refs(first: 10, after: cursor)
   → Consistent pagination สำหรับทุก lists

3. Strict Rate Limiting:
   - Point system (mutations มีค่ามากกว่า queries)
   - 5000 points per hour per token
   - X-RateLimit-* headers

4. Mutations return the changed resource:
   mutation { createIssue(input: {...}) {
     issue { id title }  # ← return changed object
   }}

5. Deprecation with date:
   fieldName: Type @deprecated(reason: "Use X. Removing 2025-06-01")

GitHub Scale:
- 100M+ developers
- 10M+ events/hour
- GraphQL handles significant portion of API traffic
```

---

## 📌 Step 2798: Shopify's GraphQL Evolution

```
Shopify's Story:
- REST API (2007) → GraphQL API (2019)
- Merchant-facing + Partner ecosystem

Key Innovations:
1. Cost-Based Rate Limiting:
   - ทุก field มี "cost"
   - Lists: cost = requested_count × field_cost
   - 1000 points per second, refills continuously
   - X-GraphQL-Cost header

2. Throttle on Complexity:
   เหมือน rate limit แต่ต่อ query cost
   ไม่ใช่แค่ requests per second

3. Bulk Operations:
   mutation { bulkOperationRunQuery(query: "...") {
     bulkOperation { id status }
   }}
   → สำหรับ large data exports (async, file download)

4. Versioning:
   - GraphQL API มี version (2024-01, 2024-04, etc.)
   - 12 months support per version
   - Quarterly releases
   → แตกต่างจาก Facebook/GitHub ที่ versionless

5. API Design Principles:
   - Clear ownership: mutation ชื่อ productCreate ไม่ใช่ createProduct
   - Input objects สำหรับ mutations
   - Error handling consistent
   - Pagination สำหรับ ทุก lists

Implementation Detail: Cost-based limiting
  const FIELD_COSTS = {
    'Query.products': 1,
    'products.edges': (requested) => requested,
    'Product.metafields': 5,
  };
```

---

## 📌 Step 2799: Architectural Patterns สรุป

```typescript
// Pattern 1: Error Union (Facebook style)
type CreateOrderResult = Order | ValidationError | InsufficientStockError;

// Pattern 2: Relay Spec (GitHub style)
interface Node {
  id: ID!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

// Pattern 3: Versioned mutations (Shopify style)
type Mutation {
  productCreate(input: ProductCreateInput!): ProductCreatePayload!
  productUpdate(id: ID!, input: ProductUpdateInput!): ProductUpdatePayload!
}

type ProductCreatePayload {
  product: Product
  userErrors: [UserError!]!  # ← Shopify pattern: return errors here
}

type UserError {
  field: [String!]
  message: String!
}

// Pattern 4: Bulk operations (Shopify style)
type Mutation {
  bulkProductExport(filter: ProductFilter): BulkOperation!
}

type BulkOperation {
  id: ID!
  status: BulkOperationStatus!
  url: URL  # เมื่อ complete: URL ไปยัง file
  errorCode: BulkOperationErrorCode
}

enum BulkOperationStatus {
  CREATED
  RUNNING
  COMPLETED
  CANCELING
  CANCELED
  FAILED
  EXPIRED
}

// Pattern 5: Connection pattern (GitHub/Relay)
type ProductConnection {
  edges: [ProductEdge!]!
  nodes: [Product!]!  # shortcut ไม่ต้องผ่าน edges
  pageInfo: PageInfo!
  totalCount: Int!
}
```

---

## 📌 Step 2800: Production Architecture ตัวอย่างสมบูรณ์

```
Client (React/Mobile)
  │
  ▼
CDN (Cloudflare)
  │ Cache public queries
  │ WAF protection
  │
  ▼
Load Balancer (AWS ALB / nginx)
  │ SSL termination
  │ Health checks
  │
  ▼
Apollo Router (Kubernetes, 3-5 replicas)
  │ JWT validation
  │ Rate limiting
  │ APQ cache (Redis)
  │ Query planning
  │ Telemetry (OTLP → Jaeger)
  │
  ├──► Users Subgraph (2-3 replicas)
  │     └── PostgreSQL + Redis cache
  │
  ├──► Products Subgraph (3-5 replicas)
  │     └── PostgreSQL + Elasticsearch + Redis
  │
  ├──► Orders Subgraph (2-3 replicas)
  │     └── PostgreSQL + RabbitMQ
  │
  └──► Subscriptions Subgraph (2 replicas)
        └── Redis PubSub + WebSocket

Supporting Services:
  - Prometheus + Grafana (metrics)
  - Jaeger (traces)
  - ELK Stack (logs)
  - PagerDuty (alerts)
  - Apollo GraphOS (schema registry)
  - Sentry (error tracking)
  
CI/CD:
  - GitHub Actions: test → build → schema check → deploy
  - Rover CLI: schema publish → compose supergraph
  - ArgoCD: GitOps deployment
```

---

## 📌 Step 2801: สรุป Part 096

### เนื้อหาที่เรียนรู้

✅ Netflix DGS Federation approach  
✅ GitHub Node interface + Relay spec  
✅ Shopify cost-based rate limiting  
✅ Error union patterns  
✅ Production architecture ครบถ้วน  

### Key Takeaways จาก World-Class Companies

```
Netflix:
□ DGS + Federation สำหรับ 1000+ engineers
□ Schema as product, not implementation detail

GitHub:
□ Node interface + globally unique IDs
□ Relay spec ให้ consistent UX สำหรับ clients
□ Point-based rate limiting

Shopify:
□ Cost-based limiting (complexity-aware)
□ Versioned API สำหรับ merchant stability
□ Payload error pattern (userErrors)
□ Bulk operations สำหรับ large datasets

Universal Principles:
□ Schema เป็น contract → backward compatibility
□ Observability first → trace ทุก operation
□ Security by default → อย่า trust clients
□ Team autonomy → Federation
□ Deprecate gracefully → analytics + communication
```

### ในส่วนถัดไป

➡️ **[Part 097](./part-097.md)** — Real-Time Systems: Chat & Collaborative Apps

---

*Part 096 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
