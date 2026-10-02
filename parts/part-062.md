# Part 062 — Performance Optimization: Advanced Strategies ⚡

> **ระดับ:** Expert | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1386–1425

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Response caching (operation-level)
- Field-level caching กับ @cacheControl
- CDN caching สำหรับ GraphQL
- Database query optimization
- Connection pooling
- Query planning & cost estimation
- Deferred queries (@defer)
- Partial query caching

---

## 📌 Step 1386: Response Caching

```typescript
// Apollo Server Response Cache
// npm install @apollo/server-plugin-response-cache

import responseCachePlugin from '@apollo/server-plugin-response-cache';
import { InMemoryLRUCache } from '@apollo/utils.keyvaluecache';
import { KeyvAdapter } from '@apollo/utils.keyvadapter';
import Keyv from 'keyv';

// Redis-backed cache
const redisCache = new KeyvAdapter(
  new Keyv(process.env.REDIS_URL, { namespace: 'gql-cache' })
);

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    responseCachePlugin({
      sessionId: ({ contextValue }) => {
        // Cache key includes user role
        const ctx = contextValue as AppContext;
        return ctx.user?.role ?? 'anonymous';
      },
    }),
  ],
  cache: redisCache,
});

// Schema-level cache hints
const typeDefs = gql`
  type Product @cacheControl(maxAge: 300) {  # Cache 5 นาที
    id: ID!
    name: String!
    price: Float! @cacheControl(maxAge: 60)  # Price เปลี่ยนบ่อยกว่า
    description: String @cacheControl(maxAge: 3600)  # Description เปลี่ยนน้อย
    inventory: ProductInventory @cacheControl(maxAge: 30, scope: PRIVATE)
  }
  
  type User @cacheControl(maxAge: 0, scope: PRIVATE) {  # ไม่ cache user data
    id: ID!
    name: String!
    email: String!
  }
  
  type Query {
    product(id: ID!): Product @cacheControl(maxAge: 300)
    products: [Product!]! @cacheControl(maxAge: 60)
    me: User @cacheControl(maxAge: 0, scope: PRIVATE)
  }
`;

// Programmatic cache hints ใน resolver
const resolvers = {
  Query: {
    product: async (_, { id }, ctx, info) => {
      // Dynamic cache hint
      const product = await ctx.prisma.product.findUnique({ where: { id } });
      
      if (product?.featured) {
        // Featured products cached longer
        info.cacheControl.setCacheHint({ maxAge: 3600 });
      } else {
        info.cacheControl.setCacheHint({ maxAge: 300 });
      }
      
      return product;
    },
  },
};
```

---

## 📌 Step 1387: CDN Caching สำหรับ GraphQL

```typescript
// GET request สำหรับ cacheable queries
// CDN (Cloudflare, Fastly) cache GET requests ได้

// Client: ส่ง query ผ่าน GET
const GET_PRODUCT = gql`
  query GetProduct($id: ID!) {
    product(id: $id) {
      id name price description
      images { url alt }
    }
  }
`;

// Apollo Client: persisted queries + GET
const link = createPersistedQueryLink({
  useGETForHashedQueries: true,  // ส่ง GET สำหรับ hashed queries
});

// Server: รองรับ GET requests
app.get('/graphql', async (req, res) => {
  const { query, variables, extensions } = req.query;
  
  // Validate: GET ต้องเป็น query เท่านั้น (ไม่ใช่ mutation)
  const parsed = parse(decodeURIComponent(query as string));
  const operation = parsed.definitions[0];
  if (operation?.kind === 'OperationDefinition' && operation.operation !== 'query') {
    return res.status(405).json({ error: 'Only queries allowed via GET' });
  }
  
  // Set cache headers สำหรับ CDN
  res.setHeader('Cache-Control', 'public, max-age=300, s-maxage=3600');
  res.setHeader('Vary', 'Accept-Encoding');  // สำหรับ gzip
  
  // Process query...
});

// APQ (Automatic Persisted Queries) — ลดขนาด request
// Client ส่ง hash แทน query string ทั้งหมด
// Server ดู hash ใน cache → ถ้าไม่มี → client ส่ง full query
```

---

## 📌 Step 1388: Database Optimization

```typescript
// src/optimization/query-optimizer.ts

// 1. Select only needed fields (avoid SELECT *)
async function getProductForResolver(id: string, info: GraphQLResolveInfo) {
  // Analyze GraphQL info เพื่อรู้ว่า fields ไหน requested
  const requestedFields = getRequestedFields(info);
  
  // Build Prisma select dynamically
  const select: Record<string, boolean> = { id: true };
  
  if (requestedFields.has('name')) select.name = true;
  if (requestedFields.has('price')) select.price = true;
  if (requestedFields.has('description')) select.description = true;
  if (requestedFields.has('createdAt')) select.createdAt = true;
  
  // Include relations only if requested
  if (requestedFields.has('category')) {
    (select as Record<string, unknown>).category = {
      select: { id: true, name: true }
    };
  }
  
  return prisma.product.findUnique({ where: { id }, select });
}

// Helper: extract requested fields from GraphQL info
function getRequestedFields(info: GraphQLResolveInfo): Set<string> {
  const fields = new Set<string>();
  
  function traverse(selections: SelectionSetNode['selections']) {
    for (const selection of selections) {
      if (selection.kind === 'Field') {
        fields.add(selection.name.value);
        if (selection.selectionSet) {
          traverse(selection.selectionSet.selections);
        }
      }
    }
  }
  
  if (info.fieldNodes[0]?.selectionSet) {
    traverse(info.fieldNodes[0].selectionSet.selections);
  }
  
  return fields;
}

// 2. Avoid N+1 with DataLoader (already covered in Part 037)
// 3. Use indexes effectively

// Ensure indexes exist สำหรับ common query patterns
// prisma/migrations/add_indexes.sql:
// CREATE INDEX idx_products_category ON products(category_id);
// CREATE INDEX idx_products_status_created ON products(status, created_at DESC);
// CREATE INDEX idx_orders_customer ON orders(customer_id, created_at DESC);
// CREATE INDEX idx_order_items_product ON order_items(product_id);

// 4. Connection pooling
// DATABASE_URL="postgresql://...?connection_limit=10&pool_timeout=10"

// Prisma connection pooling config
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL,
    },
  },
  log: process.env.NODE_ENV === 'development' 
    ? ['query', 'warn', 'error'] 
    : ['warn', 'error'],
});

// PgBouncer สำหรับ production
// DATABASE_URL="...?pgbouncer=true&connection_limit=1"
```

---

## 📌 Step 1389: @defer Directive

```graphql
# @defer: ส่ง partial response ก่อน รอ expensive fields ทีหลัง
# รองรับใน Apollo Server 4+ และ Yoga

query GetProduct($id: ID!) {
  product(id: $id) {
    id
    name
    price
    
    # ส่งหลัง — expensive operation
    ... on Product @defer(label: "reviews") {
      reviews(first: 10) {
        id rating comment
        author { name }
      }
    }
    
    # ส่งหลัง — slow related data
    ... on Product @defer(label: "analytics") {
      viewCount
      salesCount
      conversionRate
    }
  }
}

# Response จะมาเป็น multipart:
# 1. ข้อมูล main fields ทันที
# 2. reviews chunk เมื่อพร้อม
# 3. analytics chunk เมื่อพร้อม
```

```typescript
// Server support สำหรับ @defer
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';

// Apollo Server 4 รองรับ @defer โดยธรรมชาติ
// ต้องใช้ HTTP transport (ไม่ใช่ WebSocket)

// Client-side handling
import { ApolloClient, InMemoryCache, gql } from '@apollo/client';

// Apollo Client >= 3.7 รองรับ @defer
const { data } = await client.query({
  query: GET_PRODUCT_WITH_DEFER,
  variables: { id },
});

// data จะ update incrementally:
// 1. { product: { id, name, price } }
// 2. { product: { id, name, price, reviews: [...] } }
// 3. { product: { id, name, price, reviews: [...], viewCount: 100 } }
```

---

## 📌 Step 1390: Query Analysis & Cost Estimation

```typescript
// src/optimization/query-analyzer.ts
// Analyze query complexity + estimated cost ก่อน execute

import { parse, validate, DocumentNode } from 'graphql';

interface QueryCost {
  depth: number;
  complexity: number;
  fieldCount: number;
  estimatedDbQueries: number;
  estimatedCost: number; // 0-100 score
}

function analyzeQuery(query: string, schema: GraphQLSchema): QueryCost {
  let depth = 0;
  let fieldCount = 0;
  let listFields = 0;
  let currentDepth = 0;
  
  function traverse(node: ASTNode) {
    if (node.kind === 'Field') {
      fieldCount++;
      currentDepth++;
      depth = Math.max(depth, currentDepth);
      
      // Detect list fields (expensive)
      const field = schema.getType('Query'); // simplified
      
      if (node.selectionSet) {
        traverse(node.selectionSet);
      }
      
      currentDepth--;
    } else if (node.kind === 'SelectionSet') {
      for (const selection of node.selections) {
        traverse(selection);
      }
    }
  }
  
  const doc = parse(query);
  traverse(doc);
  
  const complexity = fieldCount * 1 + listFields * 10 + depth * 2;
  
  return {
    depth,
    complexity,
    fieldCount,
    estimatedDbQueries: Math.ceil(fieldCount / 3),
    estimatedCost: Math.min(100, complexity),
  };
}

// Log slow queries
const slowQueryPlugin = {
  async requestDidStart(ctx) {
    const start = Date.now();
    return {
      async willSendResponse(ctx) {
        const duration = Date.now() - start;
        if (duration > 1000) {
          console.warn('Slow query detected', {
            duration,
            query: ctx.request.query?.substring(0, 200),
            variables: ctx.request.variables,
          });
        }
      },
    };
  },
};
```

---

## 📌 Step 1391: สรุป Part 062

### เนื้อหาที่เรียนรู้

✅ Response caching กับ @cacheControl  
✅ CDN caching ด้วย GET + APQ  
✅ Database query optimization  
✅ Connection pooling  
✅ @defer directive  
✅ Query cost analysis  

### Performance Checklist

```
Database:
□ Indexes สำหรับ common filters/sorts
□ Connection pooling configured
□ PgBouncer สำหรับ production
□ DataLoader ทุก N+1 ที่เป็นไปได้
□ Select only requested fields

Caching:
□ @cacheControl annotations
□ Response cache plugin
□ APQ enabled
□ CDN สำหรับ public queries
□ Redis cache สำหรับ expensive computations

Network:
□ gzip compression
□ HTTP/2 (multiplexing)
□ @defer สำหรับ slow fields

Monitoring:
□ Slow query logging
□ N+1 detection
□ Cache hit/miss metrics
□ P95/P99 latency tracking
```

### ในส่วนถัดไป

➡️ **[Part 063](./part-063.md)** — Load Testing & Performance Benchmarking

---

*Part 062 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
