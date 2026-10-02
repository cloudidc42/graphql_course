# Part 094 — GraphQL Interview Preparation 🎯

> **ระดับ:** Expert | **เวลาเรียน:** 75 นาที | **ขั้นตอนที่:** 2706–2750

---

## 🎯 สิ่งที่จะได้เรียนรู้

- คำถามสัมภาษณ์ระดับต่างๆ
- ตอบคำถาม System Design
- Live coding challenges
- คำถามเกี่ยวกับ Architecture
- คำถาม Security
- Behavioral questions
- Take-home project tips

---

## 📌 Step 2706: Junior Level Questions

```
Q: GraphQL คืออะไร ต่างจาก REST อย่างไร?

A: GraphQL เป็น query language สำหรับ API
   ที่ให้ client กำหนดได้ว่าต้องการ data อะไร

   ข้อดีหลักเทียบ REST:
   1. No over-fetching: client ขอแค่ fields ที่ต้องการ
   2. No under-fetching: ได้ data ทุกอย่างใน 1 request
   3. Strongly typed: schema เป็น contract
   4. Introspection: self-documenting

Q: อธิบาย Query, Mutation, Subscription

A: Query    = อ่านข้อมูล (เทียบ GET)
   Mutation  = เขียน/แก้ไขข้อมูล (เทียบ POST/PUT/DELETE)
   Subscription = real-time updates ผ่าน WebSocket

Q: N+1 problem คืออะไร แก้อย่างไร?

A: N+1: query products (1) แล้วสำหรับ each product ดึง seller (N)
   → รวม N+1 queries

   แก้ด้วย DataLoader:
   - Batch keys ทั้งหมดในหนึ่ง event loop tick
   - Execute ใน 1 query
   → รวม 2 queries

Q: อะไรคือ resolver?

A: Function ที่รับผิดชอบ return data สำหรับ field หนึ่งๆ
   Signature: (parent, args, context, info) => data
   - parent: ค่าจาก parent resolver
   - args: arguments ที่ client ส่งมา
   - context: shared state (db, user, loaders)
   - info: query metadata
```

---

## 📌 Step 2707: Mid-Level Questions

```
Q: อธิบาย Apollo Federation

A: Federation = แบ่ง GraphQL schema ออกเป็น subgraphs
   แต่ยังเห็นเป็น unified schema จาก client

   Components:
   - Subgraph: ทีมต่างๆ manage
   - Apollo Router: รวม + route queries
   - Supergraph: composed schema

   Key directives:
   @key(fields: "id")           → entity identifier
   @external                    → field defined elsewhere
   @requires(fields: "price")   → need field from another subgraph
   @provides(fields: "name")    → can provide field

Q: Fragment colocation คืออะไร ทำไมถึงดี?

A: Component กำหนด data requirement ของตัวเองด้วย fragment

   Benefits:
   1. Encapsulation: component เปลี่ยน fragment โดยไม่ affect อื่น
   2. Unused field detection: codegen รู้ว่า field ไม่ได้ใช้
   3. Documentation: fragment = data dependency spec

Q: อธิบาย persisted queries

A: แทนที่ส่ง full query text → ส่งแค่ hash (pre-registered)

   Benefits:
   1. Smaller request payloads
   2. Security: only pre-approved queries allowed
   3. Better APM: operations tracked by hash
   4. Caching: CDN cache บน hash

Q: จะ handle authentication กับ authorization ใน GraphQL อย่างไร?

A: Authentication: ใน context function
   - Parse JWT จาก Authorization header
   - ถ้า invalid → user = null (ไม่ throw)

   Authorization: ใน resolvers
   - Pattern 1: if (!ctx.user) throw
   - Pattern 2: @auth directive
   - Pattern 3: auth scopes (Pothos)
   
   ไม่ทำใน middleware เพราะ GraphQL ใช้ POST /graphql ทุก operation
```

---

## 📌 Step 2708: Senior/Lead Questions

```
Q: ออกแบบ GraphQL API สำหรับ e-commerce platform ที่ scale 1M users/day

A (System Design):

Schema Design:
  - Products, Orders, Users, Reviews แยก subgraphs (Federation)
  - Connection pattern สำหรับ pagination
  - Error unions แทน null returns

Performance:
  - DataLoader ทุก N:1 relationship
  - Redis cache สำหรับ expensive queries
  - APQ (Automatic Persisted Queries)
  - CDN cache สำหรับ public queries (products, categories)

Security:
  - Query depth limit: 10
  - Complexity limit: 5000
  - Rate limiting: per user + per operation
  - Persisted queries ใน production

Scalability:
  - Apollo Router ใน Kubernetes
  - HPA ตาม custom metrics (in-flight requests)
  - Redis สำหรับ subscription PubSub

Observability:
  - OpenTelemetry tracing
  - Prometheus metrics per operation
  - Grafana dashboards
  - PagerDuty alerts

Q: Breaking change ใน GraphQL คืออะไร จัดการอย่างไร?

A:
Breaking = clients ที่ใช้อยู่จะ error:
  - ลบ field/type
  - เปลี่ยน type (String → Int)
  - ลบ enum value
  - เพิ่ม required input field

แนวทาง:
  1. @deprecated แทนการลบทันที
  2. Monitor usage analytics → รู้ว่า clients ยังใช้อยู่
  3. Communicate timeline ผ่าน changelog/email
  4. Remove หลังจาก usage = 0
  5. findBreakingChanges() ใน CI
  6. GraphQL Hive สำหรับ schema registry

Q: Subscription เมื่อ scale ข้าม instances?

A:
ปัญหา: subscription connections distribute กระจายบน instances
         → publish ไปถูก instance เท่านั้น

แก้:
  - PubSub ที่ใช้ Redis/NATS/Kafka (shared message bus)
  - ทุก instance subscribe ต่อ Redis channel
  - Publish ไปที่ Redis → fan-out ไปทุก instance → ส่งไปแต่ละ client

Code:
  - createRedisEventTarget() จาก @graphql-yoga/redis-event-target
  - หรือ NatsPubSub custom implementation
```

---

## 📌 Step 2709: Security Questions

```
⚠️ เนื้อหาด้านล่างเป็นการเรียนรู้เพื่อการป้องกันเท่านั้น
   การใช้เทคนิคเหล่านี้กับระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

Q: Introspection attack คืออะไร ป้องกันอย่างไร?

A:
Attack: Attacker ใช้ __schema query เพื่อ enumerate ทุก type/field
        → รู้ structure ของ API → ง่ายต่อการหา vulnerabilities

ป้องกัน:
  1. Disable introspection ใน production
     introspection: process.env.NODE_ENV !== 'production'
  
  2. ถ้า disable ไม่ได้ → whitelist IPs ที่อนุญาต
  
  3. ลบ "Did you mean X?" suggestions ใน errors
     → ลด information disclosure

Q: Injection attack ใน GraphQL

A:
SQL injection: ถ้า concat user input ใน raw query
  Prevention: parameterized queries หรือ ORM (Prisma)

SSRF: resolver ที่ fetch URL จาก user input
  Prevention: URL allowlist validation

NoSQL injection: ถ้าใช้ MongoDB dynamic queries
  Prevention: validate input types ใน schema (strongly typed)

Q: IDOR ใน GraphQL

A:
ตัวอย่าง: 
  query { order(id: "other-user-order-id") { ... } }

ป้องกัน: WHERE clause ด้วย userId
  prisma.order.findFirst({
    where: { id, userId: ctx.user.id }  // ← ต้องเป็น owner
  })

Common mistake: ตรวจ auth แค่ operation level ไม่ตรวจ resource level
```

---

## 📌 Step 2710: Live Coding Challenge

```typescript
// Challenge: Implement a paginated products query with DataLoader
// Time: 20 นาที

// Given:
// - Prisma client
// - Types: Product, Category
// - Need: products query with cursor pagination and category N+1 fix

// Solution:

const typeDefs = gql`
  type Product {
    id: ID!
    name: String!
    price: Float!
    category: Category!
  }
  
  type Category {
    id: ID!
    name: String!
  }
  
  type ProductEdge {
    node: Product!
    cursor: String!
  }
  
  type ProductConnection {
    edges: [ProductEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }
  
  type PageInfo {
    hasNextPage: Boolean!
    endCursor: String
  }
  
  type Query {
    products(first: Int = 20, after: String): ProductConnection!
  }
`;

// Resolver
const resolvers = {
  Query: {
    products: async (_: unknown, { first = 20, after }: PaginationArgs, ctx: AppContext) => {
      const cursor = after ? Buffer.from(after, 'base64').toString() : undefined;
      
      const [products, totalCount] = await Promise.all([
        ctx.prisma.product.findMany({
          take: first + 1,  // +1 สำหรับ hasNextPage
          cursor: cursor ? { id: cursor } : undefined,
          skip: cursor ? 1 : 0,
          orderBy: { createdAt: 'desc' },
        }),
        ctx.prisma.product.count(),
      ]);
      
      const hasNextPage = products.length > first;
      const items = hasNextPage ? products.slice(0, -1) : products;
      
      return {
        edges: items.map(p => ({
          node: p,
          cursor: Buffer.from(p.id).toString('base64'),
        })),
        pageInfo: {
          hasNextPage,
          endCursor: items[items.length - 1]
            ? Buffer.from(items[items.length - 1].id).toString('base64')
            : null,
        },
        totalCount,
      };
    },
  },
  
  Product: {
    // DataLoader แก้ N+1
    category: (product, _args, ctx) => 
      ctx.loaders.categoryById.load(product.categoryId),
  },
};
```

---

## 📌 Step 2711: สรุป Part 094

### เนื้อหาที่เรียนรู้

✅ คำถาม Junior → Senior  
✅ System design approach  
✅ Security questions  
✅ Live coding pattern  

### Interview Preparation Checklist

```
Concepts ที่ต้องรู้ชัด:
□ N+1 + DataLoader
□ Federation architecture
□ JWT auth pattern
□ Cursor pagination
□ Persisted queries
□ Breaking changes

System Design ที่ต้องฝึก:
□ E-commerce GraphQL API
□ Real-time chat/notifications
□ Multi-tenant SaaS

Code ที่ควรทำได้ใน 20 นาที:
□ DataLoader implementation
□ Custom directive
□ Resolver กับ error handling
□ Pagination

สิ่งที่แสดง seniority:
□ Trade-offs ชัดเจน
□ Operational concerns (monitoring, alerting)
□ Security-first thinking
□ Team considerations
```

### ในส่วนถัดไป

➡️ **[Part 095](./part-095.md)** — Production-Ready Checklist & Certification Study Guide

---

*Part 094 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
