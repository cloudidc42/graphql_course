# Part 099 — GraphQL Performance Deep Dive 🚀

> **ระดับ:** World-Class | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 2931–2975

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Query execution profiling
- Schema execution optimization
- Database query analysis
- Resolver chain optimization
- Memory profiling
- JIT compilation (graphql-jit)
- Deferred execution (@defer/@stream)
- Micro-optimizations ที่ scale

---

## 📌 Step 2931: Profiling GraphQL Operations

```typescript
// src/plugins/profiler.plugin.ts
// Detailed per-field profiling สำหรับ development

interface FieldTiming {
  typeName: string;
  fieldName: string;
  duration: number;
  parentId?: string;
  depth: number;
}

export function useDetailedProfiling(): ApolloServerPlugin {
  return {
    async requestDidStart({ request }) {
      if (process.env.NODE_ENV !== 'development') return {};
      
      const timings: FieldTiming[] = [];
      const fieldStack: string[] = [];
      
      return {
        async executionDidStart() {
          return {
            willResolveField({ info }) {
              const fieldId = `${info.parentType.name}.${info.fieldName}`;
              const start = performance.now();
              fieldStack.push(fieldId);
              
              return () => {
                const duration = performance.now() - start;
                fieldStack.pop();
                
                timings.push({
                  typeName: info.parentType.name,
                  fieldName: info.fieldName,
                  duration,
                  depth: info.path.length,
                });
              };
            },
          };
        },
        
        async willSendResponse({ response }) {
          // Top 10 slowest fields
          const sorted = [...timings].sort((a, b) => b.duration - a.duration);
          
          if (sorted[0]?.duration > 10) {  // ถ้ามี field ที่ช้าเกิน 10ms
            console.log('⚠️ Slow fields:');
            sorted.slice(0, 10).forEach(t => {
              console.log(`  ${t.typeName}.${t.fieldName}: ${t.duration.toFixed(2)}ms`);
            });
          }
          
          // Add to extensions ใน development
          if (response.body.kind === 'single') {
            response.body.singleResult.extensions ??= {};
            response.body.singleResult.extensions.timing = {
              total: sorted.reduce((s, t) => s + t.duration, 0).toFixed(2),
              slowestFields: sorted.slice(0, 5).map(t => ({
                field: `${t.typeName}.${t.fieldName}`,
                ms: parseFloat(t.duration.toFixed(2)),
              })),
            };
          }
        },
      };
    },
  };
}
```

---

## 📌 Step 2932: graphql-jit สำหรับ Performance

```bash
npm install graphql-jit
```

```typescript
// src/execution/jit-executor.ts
// graphql-jit: compiles resolvers ไป optimized JavaScript
// ดีกว่า default interpreter สำหรับ high-traffic

import { compileQuery, isCompiledQuery } from 'graphql-jit';
import { parse, validate } from 'graphql';

class JITExecutor {
  private queryCache = new LRUCache<string, ReturnType<typeof compileQuery>>({
    max: 1000,  // Cache 1000 compiled queries
  });
  
  async execute(schema: GraphQLSchema, query: string, variables: Record<string, unknown>, context: AppContext) {
    // ใช้ query hash เป็น cache key
    const cacheKey = createHash('sha256').update(query).digest('hex');
    
    let compiled = this.queryCache.get(cacheKey);
    
    if (!compiled) {
      const document = parse(query);
      const errors = validate(schema, document);
      if (errors.length) throw new Error(errors[0].message);
      
      compiled = compileQuery(schema, document);
      this.queryCache.set(cacheKey, compiled);
    }
    
    if (!isCompiledQuery(compiled)) {
      throw new Error('Compilation failed');
    }
    
    // Execute compiled query (significantly faster than default)
    return compiled.query(undefined, context, variables);
  }
}

// Benchmark:
// Default graphql-js:  ~15,000 ops/sec
// graphql-jit:         ~50,000-100,000 ops/sec (3-7x faster)
// Benefit เพิ่มขึ้นตาม schema complexity
```

---

## 📌 Step 2933: @defer และ @stream (Incremental Delivery)

```graphql
# @defer: ส่ง critical data ก่อน, defer expensive fields
query GetOrder($id: ID!) {
  order(id: $id) {
    id
    status
    totalAmount
    
    # Expensive: defer ไว้ส่งทีหลัง
    ... on Order @defer(label: "reviews") {
      productReviews {
        rating
        comment
        author { name }
      }
    }
    
    ... on Order @defer(label: "recommendations") {
      similarProducts {
        id
        name
        price
      }
    }
  }
}

# @stream: stream large arrays incrementally
query GetProducts {
  products @stream(initialCount: 3) {
    id
    name
    price
  }
}
```

```typescript
// Server support for @defer (Yoga/Apollo 4+)
import { createYoga } from 'graphql-yoga';
import { schema } from './schema';

const yoga = createYoga({
  schema,
  // Yoga supports @defer/@stream out of the box
  // ส่ง multipart response
});

// Client: handle incremental results
const client = new ApolloClient({
  link: new HttpLink({
    uri: '/graphql',
    // Apollo Client 3.7+ supports @defer
    headers: { 'Accept': 'multipart/mixed; deferSpec=20220824, application/json' },
  }),
  cache: new InMemoryCache(),
});

// Usage ใน React
function OrderDetail({ id }: { id: string }) {
  const { data } = useQuery(GET_ORDER_DEFERRED, {
    variables: { id },
    // Deferred fields มาช้ากว่า แต่ UI ไม่รอ
  });
  
  return (
    <div>
      <h1>Order #{data?.order?.id}</h1>
      <p>Status: {data?.order?.status}</p>
      
      {/* Show when deferred data arrives */}
      {data?.order?.productReviews
        ? <ReviewsList reviews={data.order.productReviews} />
        : <Skeleton />}
    </div>
  );
}
```

---

## 📌 Step 2934: Database Query Optimization

```typescript
// src/utils/prisma-select.ts
// ใช้ info object เพื่อ select เฉพาะ fields ที่ request

import { GraphQLResolveInfo, FieldNode, SelectionSetNode } from 'graphql';

function getRequestedFields(info: GraphQLResolveInfo): Set<string> {
  const fields = new Set<string>();
  
  function traverse(selectionSet: SelectionSetNode) {
    for (const selection of selectionSet.selections) {
      if (selection.kind === 'Field') {
        fields.add(selection.name.value);
        if (selection.selectionSet) {
          traverse(selection.selectionSet);
        }
      }
    }
  }
  
  traverse(info.fieldNodes[0].selectionSet!);
  return fields;
}

// Resolver: select only requested fields
const productResolvers = {
  Query: {
    product: async (_: unknown, { id }: { id: string }, ctx: AppContext, info: GraphQLResolveInfo) => {
      const requestedFields = getRequestedFields(info);
      
      // Build select object
      const select: Record<string, boolean> = {};
      const allFields = ['id', 'name', 'price', 'stock', 'description', 'createdAt'];
      
      for (const field of allFields) {
        select[field] = requestedFields.has(field);
      }
      
      // Always select id (needed for DataLoader)
      select.id = true;
      
      return ctx.prisma.product.findUnique({
        where: { id },
        select,
      });
    },
  },
};

// ดีกว่า: ใช้ Pothos Prisma plugin ที่ทำอัตโนมัติ
// หรือ @graphql-tools/merge กับ @pothos/plugin-prisma
```

---

## 📌 Step 2935: Memory Optimization

```typescript
// src/optimization/memory.ts

// 1. Document caching: parse + validate ครั้งเดียว
const documentCache = new LRUCache<string, DocumentNode>({
  max: 500,
  sizeCalculation: (doc) => JSON.stringify(doc).length,
  maxSize: 10 * 1024 * 1024,  // 10MB
});

function getCachedDocument(query: string): DocumentNode {
  const cached = documentCache.get(query);
  if (cached) return cached;
  
  const doc = parse(query);
  documentCache.set(query, doc);
  return doc;
}

// 2. Response size monitoring
function createResponseSizePlugin(): ApolloServerPlugin {
  return {
    async requestDidStart() {
      return {
        async willSendResponse({ response }) {
          const body = JSON.stringify(response.body);
          const sizeKB = Buffer.byteLength(body, 'utf8') / 1024;
          
          responseSizeHistogram.observe(sizeKB);
          
          if (sizeKB > 1000) {  // > 1MB
            logger.warn({
              type: 'large_response',
              sizeKB,
              operation: (response as Record<string, unknown>).operationName,
            });
          }
        },
      };
    },
  };
}

// 3. Limit list sizes
const MAX_PAGE_SIZE = 100;

const resolvers = {
  Query: {
    products: (_: unknown, { first = 20 }: { first?: number }) => {
      const limit = Math.min(first, MAX_PAGE_SIZE);
      return prisma.product.findMany({ take: limit });
    },
  },
};
```

---

## 📌 Step 2936: Benchmark Results

```
Performance Comparison (1000 concurrent users, 60 seconds):

Baseline (No optimization):
  Throughput:  1,200 req/s
  P95:         850ms
  P99:         2,100ms
  Error rate:  2.1%

+ DataLoader:
  Throughput:  3,500 req/s  (+192%)
  P95:         280ms  (-67%)
  P99:         620ms  (-70%)
  Error rate:  0.2%

+ Redis Cache:
  Throughput:  8,200 req/s  (+134%)
  P95:         85ms   (-70%)
  P99:         210ms  (-66%)
  Error rate:  0.05%

+ APQ + graphql-jit:
  Throughput:  14,000 req/s (+71%)
  P95:         45ms   (-47%)
  P99:         120ms  (-43%)
  Error rate:  0.02%

+ @defer สำหรับ TTFB:
  Time to First Byte: 25ms (critical path only)
  Full response:      180ms (background)
  Perceived speed:    3-4x better UX

Stack:
  Node.js 20, graphql-yoga 3, Prisma 5
  PostgreSQL 15 (RDS r6g.large)
  Redis 7 (ElastiCache r7g.large)
  ECS Fargate 1 vCPU / 2GB RAM × 10 tasks
```

---

## 📌 Step 2937: สรุป Part 099

### เนื้อหาที่เรียนรู้

✅ Per-field profiling  
✅ graphql-jit (3-7x faster execution)  
✅ @defer / @stream  
✅ DB field selection optimization  
✅ Memory optimization  
✅ Benchmark methodology  

### Performance Optimization Stack

```
Layer 1: Client
□ APQ (Automatic Persisted Queries)
□ @defer สำหรับ non-critical fields
□ Fragment colocation (ไม่ over-fetch)

Layer 2: Server
□ graphql-jit compiler
□ Document parse cache
□ Response size limits
□ Per-field profiling ใน dev

Layer 3: Data
□ DataLoader (eliminate N+1)
□ Prisma select (fetch only needed fields)
□ Redis response cache
□ DB query explain analyze

Layer 4: Infrastructure
□ CDN cache สำหรับ public queries
□ Connection pooling (Prisma Accelerate)
□ Horizontal scaling
□ Regional deployment
```

### ในส่วนถัดไป

➡️ **[Part 100](./part-100.md)** — Capstone: Complete Production System

---

*Part 099 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
