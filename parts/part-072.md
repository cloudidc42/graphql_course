# Part 072 — GraphQL at Scale: Architecture Patterns 🏗️

> **ระดับ:** World-Class | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1781–1820

---

## 🎯 สิ่งที่จะได้เรียนรู้

- GraphQL at Netflix/GitHub/Shopify scale
- Schema stitching vs Federation
- BFF (Backend for Frontend) pattern
- API Gateway pattern
- Schema versioning at scale
- Cost attribution
- Query whitelisting
- GraphQL persisted operations

---

## 📌 Step 1781: Scale Patterns Overview

```
Netflix (2019): Unified API
- 100s of microservices
- GraphQL Federation เชื่อม services ทั้งหมด
- Client receives unified schema
- Lessons: Domain ownership, no "GraphQL team"

GitHub (2016): GraphQL API v4
- Public REST API → GraphQL
- Strong typing helped clients
- Breaking change: can't change public API
- Lessons: Design for evolution, deprecation is real

Shopify (Storefront API):
- High traffic: millions of requests/sec
- Multi-tenant: thousands of stores
- Rate limiting by "cost" not "count"
- Lessons: Cost-based rate limiting essential
```

---

## 📌 Step 1782: Cost-Based Rate Limiting (Shopify Style)

```typescript
// src/ratelimit/cost-based.ts

interface FieldCost {
  object?: number;
  list?: number;  // cost per returned item
  mutation?: number;
}

// Cost map: ระบุ cost ของแต่ละ field
const FIELD_COSTS: Record<string, FieldCost> = {
  'Query.products': { list: 1 },       // 1 point per product
  'Query.orders': { list: 2 },          // 2 points per order (heavier)
  'Query.me': { object: 1 },
  'Query.search': { list: 3 },          // search แพงกว่า
  'Mutation.createOrder': { mutation: 10 },
  'Mutation.checkout': { mutation: 20 },
  'Product.reviews': { list: 1 },
  'User.orders': { list: 2 },
};

// Calculate query cost
export function calculateQueryCost(
  document: DocumentNode,
  schema: GraphQLSchema,
  variables: Record<string, unknown>
): { cost: number; maxNodes: number } {
  let cost = 0;
  let maxNodes = 0;
  
  function visitField(
    fieldName: string,
    typeName: string,
    args: Record<string, unknown>,
    depth: number
  ) {
    const key = `${typeName}.${fieldName}`;
    const fieldCost = FIELD_COSTS[key];
    
    if (fieldCost?.mutation) {
      cost += fieldCost.mutation;
    } else if (fieldCost?.list) {
      // first/limit argument = max items
      const limit = (args.first ?? args.limit ?? 10) as number;
      cost += fieldCost.list * limit;
      maxNodes = Math.max(maxNodes, limit);
    } else if (fieldCost?.object) {
      cost += fieldCost.object;
    } else {
      cost += 1;  // default: 1 point
    }
  }
  
  // Visit document...
  
  return { cost, maxNodes };
}

// Middleware
export function costLimitMiddleware(options: {
  maxCostPerRequest: number;
  maxCostPerMinute: number;
}) {
  const redis = getRedis();
  
  return async (req: Request, res: Response, next: NextFunction) => {
    const document = parse(req.body.query);
    const { cost } = calculateQueryCost(document, schema, req.body.variables ?? {});
    
    // Check per-request limit
    if (cost > options.maxCostPerRequest) {
      return res.status(429).json({
        errors: [{
          message: `Query cost ${cost} exceeds limit ${options.maxCostPerRequest}`,
          extensions: { code: 'QUERY_TOO_EXPENSIVE', cost, limit: options.maxCostPerRequest },
        }],
      });
    }
    
    // Check rate limit (sliding window)
    const userId = extractUserId(req) ?? req.ip ?? 'anonymous';
    const key = `ratelimit:cost:${userId}`;
    const now = Date.now();
    const window = 60_000; // 1 minute
    
    const pipe = redis.pipeline();
    pipe.zadd(key, now, `${now}-${Math.random()}`);
    pipe.zremrangebyscore(key, 0, now - window);
    pipe.zrangebyscore(key, now - window, now);
    pipe.expire(key, 120);
    
    const results = await pipe.exec();
    const recentRequests = (results?.[2]?.[1] as string[]) ?? [];
    const totalCost = recentRequests.length; // simplified: count as cost
    
    if (totalCost + cost > options.maxCostPerMinute) {
      const retryAfter = Math.ceil(window / 1000);
      res.setHeader('Retry-After', retryAfter);
      res.setHeader('X-RateLimit-Limit', options.maxCostPerMinute);
      res.setHeader('X-RateLimit-Remaining', 0);
      res.setHeader('X-RateLimit-Reset', Math.ceil((now + window) / 1000));
      
      return res.status(429).json({
        errors: [{
          message: 'Rate limit exceeded',
          extensions: { code: 'RATE_LIMITED', retryAfter },
        }],
      });
    }
    
    // Add cost info to response headers
    res.setHeader('X-GraphQL-Cost', cost);
    res.setHeader('X-RateLimit-Remaining', options.maxCostPerMinute - totalCost - cost);
    
    next();
  };
}
```

---

## 📌 Step 1783: BFF (Backend for Frontend) Pattern

```typescript
// ปัญหา: mobile app ต้องการข้อมูลน้อยกว่า web app
// → BFF: GraphQL layer เฉพาะสำหรับแต่ละ client type

// services/bff-mobile/src/schema.ts
// Mobile BFF: schema เล็กกว่า, optimized สำหรับ mobile
const mobileBFFTypeDefs = gql`
  type Query {
    # Simplified product สำหรับ mobile (ไม่มี full description)
    productCard(id: ID!): ProductCard
    
    # Preloaded homepage data ใน 1 query
    homepageData: HomepageData!
    
    # Mobile-specific: ไม่มี cursor pagination (too complex for mobile)
    productsList(page: Int = 1, limit: Int = 20): [ProductCard!]!
  }
  
  type ProductCard {
    id: ID!
    name: String!
    price: Float!
    thumbnailUrl: String!   # ขนาดเล็กสำหรับ mobile
    rating: Float!
    reviewCount: Int!
  }
  
  type HomepageData {
    featured: [ProductCard!]!
    categories: [CategoryCard!]!
    banners: [BannerCard!]!
    userId: ID             # null ถ้าไม่ได้ login
  }
`;

// BFF resolves โดย call internal services/APIs
const mobileBFFResolvers = {
  Query: {
    homepageData: async (_, __, ctx) => {
      // Parallel requests ไปยัง microservices
      const [featured, categories, banners] = await Promise.all([
        productsService.getFeatured(),
        catalogService.getCategories(),
        contentService.getBanners(),
      ]);
      
      return { featured, categories, banners, userId: ctx.user?.id };
    },
  },
};
```

---

## 📌 Step 1784: Query Whitelisting (Production Security)

```typescript
// src/security/query-whitelist.ts
// Allow ONLY registered queries ใน production

import crypto from 'crypto';
import { readFileSync, readdirSync } from 'fs';
import { join } from 'path';

// Build whitelist จาก .graphql files
function buildQueryWhitelist(): Map<string, string> {
  const whitelist = new Map<string, string>();
  
  // โหลด persisted queries จาก directory
  const queryDir = join(process.cwd(), 'queries');
  const files = readdirSync(queryDir).filter(f => f.endsWith('.graphql'));
  
  for (const file of files) {
    const query = readFileSync(join(queryDir, file), 'utf-8');
    const hash = crypto.createHash('sha256').update(query).digest('hex');
    whitelist.set(hash, query);
  }
  
  return whitelist;
}

const queryWhitelist = buildQueryWhitelist();

// Middleware
export function queryWhitelistMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
) {
  // Skip ใน development
  if (process.env.NODE_ENV !== 'production') return next();
  
  const { query, extensions } = req.body;
  
  // Persisted query via hash
  if (extensions?.persistedQuery?.sha256Hash) {
    const hash = extensions.persistedQuery.sha256Hash;
    const whitelistedQuery = queryWhitelist.get(hash);
    
    if (!whitelistedQuery) {
      return res.status(400).json({
        errors: [{ message: 'Query not whitelisted', extensions: { code: 'QUERY_NOT_WHITELISTED' } }],
      });
    }
    
    req.body.query = whitelistedQuery;
    return next();
  }
  
  // Ad-hoc query: check if whitelisted
  if (query) {
    const hash = crypto.createHash('sha256').update(query).digest('hex');
    if (!queryWhitelist.has(hash)) {
      return res.status(400).json({
        errors: [{ message: 'Ad-hoc queries not allowed', extensions: { code: 'QUERY_NOT_WHITELISTED' } }],
      });
    }
  }
  
  next();
}
```

---

## 📌 Step 1785: สรุป Part 072

### เนื้อหาที่เรียนรู้

✅ Scale patterns (Netflix, GitHub, Shopify)  
✅ Cost-based rate limiting  
✅ BFF pattern  
✅ Query whitelisting  

### At-Scale Checklist

```
Schema governance:
□ Schema Registry (Apollo GraphOS หรือ GraphQL Hive)
□ Breaking change detection CI
□ Schema changelog
□ Deprecation policy + sunset dates

Performance:
□ Cost-based rate limiting
□ Query complexity limits
□ Persisted queries / APQ
□ CDN สำหรับ public queries

Security:
□ Query whitelisting ใน production
□ Introspection disabled
□ Field suggestion disabled

Operations:
□ Per-operation metrics
□ Slow operation alerts
□ Cost attribution (ทีมไหนใช้ quota เท่าไหร่)
```

### ในส่วนถัดไป

➡️ **[Part 073](./part-073.md)** — GraphQL Documentation & Developer Experience

---

*Part 072 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
