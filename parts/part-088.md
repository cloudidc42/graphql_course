# Part 088 — GraphQL Gateway Patterns 🚪

> **ระดับ:** World-Class | **เวลาเรียน:** 85 นาที | **ขั้นตอนที่:** 2436–2480

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Gateway patterns ต่างๆ
- Schema stitching vs Federation
- Persisted queries ที่ gateway
- Request routing ตาม client
- Rate limiting ที่ gateway
- Circuit breaker pattern
- Observability ที่ gateway
- Self-healing gateway

---

## 📌 Step 2436: Gateway Architecture Overview

```
Gateway Patterns:

1. Single Schema Gateway (Monolith):
   Client → Gateway → One API
   ✅ Simple, fast
   ❌ Scale bottleneck

2. Schema Stitching:
   Client → Gateway → Stitch [API1, API2, API3]
   ✅ Flexible, legacy friendly
   ❌ Gateway handles N+1, harder to maintain

3. Apollo Federation:
   Client → Router → [Subgraph1, Subgraph2, Subgraph3]
   ✅ Distributed, team autonomy
   ✅ Type-safe cross-service references
   ❌ More complex setup

4. GraphQL Proxy (Hasura/PostGraphile style):
   Client → Auto-generated API → Database
   ✅ No code needed
   ❌ Less flexibility, security concerns

5. Hybrid:
   Client → Gateway → [Legacy REST, GraphQL Services, gRPC]
   ✅ Incremental migration
   ✅ Mix old and new services
```

---

## 📌 Step 2437: Schema Stitching Gateway

```typescript
// src/gateway/schema-stitching.ts
import { makeExecutableSchema } from '@graphql-tools/schema';
import { stitchSchemas } from '@graphql-tools/stitch';
import { schemaFromExecutor, wrapSchema } from '@graphql-tools/wrap';
import { buildHTTPExecutor } from '@graphql-tools/executor-http';
import { RenameTypes, FilterObjectFields } from '@graphql-tools/wrap';

// Executor สำหรับ remote schemas
const usersExecutor = buildHTTPExecutor({
  endpoint: 'http://users-service:4001/graphql',
  headers: (executorRequest) => ({
    Authorization: executorRequest.context?.authHeader ?? '',
  }),
});

const productsExecutor = buildHTTPExecutor({
  endpoint: 'http://products-service:4002/graphql',
});

const ordersExecutor = buildHTTPExecutor({
  endpoint: 'http://orders-service:4003/graphql',
});

async function buildGatewaySchema() {
  // ดึง remote schemas
  const usersSchema = wrapSchema({
    schema: await schemaFromExecutor(usersExecutor),
    executor: usersExecutor,
    transforms: [
      // Rename เพื่อหลีกเลี่ยง type conflicts
      new RenameTypes((name) => name === 'User' ? 'UsersServiceUser' : name),
    ],
  });
  
  const productsSchema = wrapSchema({
    schema: await schemaFromExecutor(productsExecutor),
    executor: productsExecutor,
  });
  
  const ordersSchema = wrapSchema({
    schema: await schemaFromExecutor(ordersExecutor),
    executor: ordersExecutor,
  });
  
  // Extension schema: เชื่อม services
  const extensionTypeDefs = /* GraphQL */`
    extend type Order {
      # เพิ่ม product details จาก products service
      product: Product
      # เพิ่ม user details จาก users service
      customer: User
    }
    
    extend type User {
      # เพิ่ม orders จาก orders service
      orders: [Order!]!
    }
  `;
  
  return stitchSchemas({
    subschemas: [usersSchema, productsSchema, ordersSchema],
    typeDefs: extensionTypeDefs,
    resolvers: {
      Order: {
        product: {
          selectionSet: `{ productId }`,
          resolve: async (order, _args, _ctx, info) =>
            delegateToSchema({
              schema: productsSchema,
              operation: 'query',
              fieldName: 'product',
              args: { id: order.productId },
              context: _ctx,
              info,
            }),
        },
      },
    },
  });
}
```

---

## 📌 Step 2438: Gateway-Level Persisted Queries

```typescript
// src/gateway/persisted-queries.ts
// Gateway ตรวจสอบ persisted queries ก่อน forward ไป subgraph

interface PersistedQuery {
  hash: string;
  query: string;
  allowedClients: string[];
  maxComplexity: number;
  rateLimit?: { window: string; max: number };
}

class PersistedQueryStore {
  private cache = new Map<string, PersistedQuery>();
  
  constructor(private redis: Redis) {}
  
  async lookup(hash: string): Promise<PersistedQuery | null> {
    // 1. Memory cache
    if (this.cache.has(hash)) return this.cache.get(hash)!;
    
    // 2. Redis
    const stored = await this.redis.get(`pq:${hash}`);
    if (stored) {
      const pq = JSON.parse(stored);
      this.cache.set(hash, pq);
      return pq;
    }
    
    return null;
  }
  
  async register(pq: PersistedQuery): Promise<void> {
    await this.redis.set(`pq:${pq.hash}`, JSON.stringify(pq));
    this.cache.set(pq.hash, pq);
  }
}

// Gateway middleware
const persistedQueryMiddleware: express.RequestHandler = async (req, res, next) => {
  const { extensions } = req.body ?? {};
  const queryHash = extensions?.persistedQuery?.sha256Hash;
  
  if (queryHash) {
    const pq = await pqStore.lookup(queryHash);
    
    if (!pq) {
      // Unknown hash: ถ้า production → reject
      if (process.env.NODE_ENV === 'production') {
        return res.status(400).json({
          errors: [{ message: 'Unknown persisted query', extensions: { code: 'PERSISTED_QUERY_NOT_FOUND' } }]
        });
      }
      
      // Development: ใช้ query จาก body
    } else {
      // Inject stored query
      req.body.query = pq.query;
      
      // ตรวจสอบ client whitelist
      const clientName = req.headers['apollographql-client-name'];
      if (!pq.allowedClients.includes(clientName as string)) {
        return res.status(403).json({
          errors: [{ message: 'Client not authorized for this query' }]
        });
      }
    }
  }
  
  next();
};
```

---

## 📌 Step 2439: Circuit Breaker ที่ Gateway

```typescript
// src/gateway/circuit-breaker.ts
import CircuitBreaker from 'opossum';

// Circuit breaker per subgraph
const subgraphBreakers = new Map<string, CircuitBreaker>();

function getBreaker(subgraphName: string, fn: (...args: unknown[]) => Promise<unknown>): CircuitBreaker {
  if (!subgraphBreakers.has(subgraphName)) {
    const breaker = new CircuitBreaker(fn, {
      timeout: 5000,                 // 5 วินาที
      errorThresholdPercentage: 50,  // เปิด circuit เมื่อ error > 50%
      resetTimeout: 30000,           // ลอง close ใหม่หลัง 30 วินาที
      volumeThreshold: 10,           // ต้องมี 10+ calls ก่อนเปิด
    });
    
    breaker.on('open', () => {
      logger.error(`Circuit breaker OPEN for ${subgraphName}`);
      alerting.notify(`Subgraph ${subgraphName} is unavailable`);
    });
    
    breaker.on('halfOpen', () => {
      logger.info(`Circuit breaker HALF-OPEN for ${subgraphName}`);
    });
    
    breaker.on('close', () => {
      logger.info(`Circuit breaker CLOSED for ${subgraphName} - recovered`);
    });
    
    subgraphBreakers.set(subgraphName, breaker);
  }
  
  return subgraphBreakers.get(subgraphName)!;
}

// Gateway resolver ด้วย circuit breaker
async function executeSubgraphQuery(subgraphName: string, query: string, variables: Record<string, unknown>, ctx: AppContext) {
  const fetchFn = () => fetchSubgraph(subgraphName, query, variables, ctx);
  const breaker = getBreaker(subgraphName, fetchFn);
  
  try {
    return await breaker.fire();
  } catch (err) {
    if (err.message === 'Breaker is open') {
      // Fallback: return cached data หรือ partial response
      const cached = await getCachedSubgraphResponse(subgraphName, query, variables);
      if (cached) return cached;
      
      throw new GraphQLError(`${subgraphName} is temporarily unavailable`, {
        extensions: { code: 'SERVICE_UNAVAILABLE', subgraph: subgraphName },
      });
    }
    throw err;
  }
}
```

---

## 📌 Step 2440: Gateway Observability

```typescript
// src/gateway/telemetry.ts
// Tracing ทุก subgraph call

const gatewayTracer = trace.getTracer('graphql-gateway');

async function tracedSubgraphCall<T>(
  subgraphName: string,
  operation: string,
  fn: () => Promise<T>
): Promise<T> {
  return gatewayTracer.startActiveSpan(
    `subgraph.${subgraphName}`,
    { attributes: { 'subgraph.name': subgraphName, 'graphql.operation': operation } },
    async (span) => {
      try {
        const result = await fn();
        span.setStatus({ code: SpanStatusCode.OK });
        return result;
      } catch (err) {
        span.recordException(err as Error);
        span.setStatus({ code: SpanStatusCode.ERROR });
        throw err;
      } finally {
        span.end();
      }
    }
  );
}

// Metrics per subgraph
const subgraphDuration = new Histogram({
  name: 'gateway_subgraph_duration_seconds',
  help: 'Duration of subgraph calls',
  labelNames: ['subgraph', 'operation', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
});
```

---

## 📌 Step 2441: สรุป Part 088

### เนื้อหาที่เรียนรู้

✅ Gateway architecture patterns  
✅ Schema stitching  
✅ Gateway-level persisted queries  
✅ Circuit breaker ต่อ subgraph  
✅ Gateway observability  

### Gateway Decision Matrix

```
Schema Stitching:
✅ Legacy REST APIs
✅ Services ที่ team อื่นดูแล
✅ ต้องการ merge types จาก services ต่างๆ
❌ Performance (gateway resolves ทุก connection)

Apollo Federation:
✅ Microservices architecture
✅ Team autonomy (each team owns subgraph)
✅ Type-safe cross-service references
✅ Apollo Router ดูแล routing
❌ Setup ซับซ้อนกว่า

Direct Pass-through:
✅ Simple proxy/authentication only
✅ No type merging needed
✅ Low overhead

Pick ตาม:
- Team structure → Federation
- Legacy services → Stitching
- New greenfield → Federation
- Simple routing → Pass-through
```

### ในส่วนถัดไป

➡️ **[Part 089](./part-089.md)** — GraphQL กับ OAuth 2.0 & OpenID Connect

---

*Part 088 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
