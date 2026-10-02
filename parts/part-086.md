# Part 086 — GraphQL กับ Edge Computing & Serverless ⚡

> **ระดับ:** World-Class | **เวลาเรียน:** 80 นาที | **ขั้นตอนที่:** 2346–2390

---

## 🎯 สิ่งที่จะได้เรียนรู้

- GraphQL บน Cloudflare Workers
- Vercel Edge Functions
- AWS Lambda กับ GraphQL
- Cold start optimization
- Edge caching strategies
- Distributed execution patterns
- Durable Objects สำหรับ subscriptions
- Cost & performance trade-offs

---

## 📌 Step 2346: GraphQL บน Cloudflare Workers

```typescript
// src/worker.ts
// Cloudflare Workers: V8 isolates, ไม่มี Node.js
// ใช้ graphql-yoga ที่ support Web API

import { createYoga } from 'graphql-yoga';
import { schema } from './schema';

const yoga = createYoga({
  schema,
  // Workers: ไม่มี file system, ไม่มี TCP connections (ใช้ Hyperdrive สำหรับ DB)
  context: async ({ request, env }) => ({
    db: env.DB,       // D1 database (SQLite at edge)
    kv: env.KV,       // KV store for caching
    r2: env.R2,       // R2 for file storage
    user: await verifyJWT(request, env.JWT_SECRET),
  }),
  
  // Workers: no introspection in production
  maskedErrors: true,
  graphiql: false,
});

// Worker entry point
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext) {
    return yoga.handleRequest(request, { env, ctx });
  },
};

// wrangler.toml
/*
name = "graphql-api"
main = "src/worker.ts"
compatibility_date = "2024-01-01"
compatibility_flags = ["nodejs_compat"]

[[d1_databases]]
binding = "DB"
database_name = "myapp"
database_id = "..."

[[kv_namespaces]]
binding = "KV"
id = "..."

[vars]
JWT_SECRET = "..."
*/
```

---

## 📌 Step 2347: D1 Database Integration

```typescript
// src/db/d1.ts
// D1: SQLite-compatible, globally replicated read replicas

interface Env {
  DB: D1Database;
}

// Resolver ใช้ D1
const productResolvers = {
  Query: {
    products: async (_: unknown, { limit = 20, offset = 0 }, { db }: AppContext) => {
      const { results } = await db.prepare(
        `SELECT id, name, price, stock 
         FROM products 
         WHERE is_active = 1 
         ORDER BY created_at DESC 
         LIMIT ? OFFSET ?`
      ).bind(limit, offset).all();
      
      return results;
    },
    
    product: async (_: unknown, { id }: { id: string }, { db }: AppContext) => {
      return db.prepare('SELECT * FROM products WHERE id = ?')
        .bind(id)
        .first();
    },
  },
  
  Mutation: {
    createProduct: async (_: unknown, { input }: CreateProductArgs, { db, user }: AppContext) => {
      requireRole(user, 'ADMIN');
      
      const id = crypto.randomUUID();
      
      await db.prepare(
        `INSERT INTO products (id, name, price, stock, created_by)
         VALUES (?, ?, ?, ?, ?)`
      ).bind(id, input.name, input.price, input.stock, user.id).run();
      
      return db.prepare('SELECT * FROM products WHERE id = ?')
        .bind(id)
        .first();
    },
  },
};
```

---

## 📌 Step 2348: Edge Caching ด้วย KV

```typescript
// src/cache/edge-cache.ts
// Workers KV: eventual consistency, ดีสำหรับ read-heavy public data

class EdgeCache {
  constructor(private kv: KVNamespace) {}
  
  async get<T>(key: string): Promise<T | null> {
    const value = await this.kv.get(key, 'json');
    return value as T | null;
  }
  
  async set<T>(key: string, value: T, ttlSeconds: number): Promise<void> {
    await this.kv.put(key, JSON.stringify(value), {
      expirationTtl: ttlSeconds,
    });
  }
  
  async getOrFetch<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttlSeconds: number
  ): Promise<T> {
    const cached = await this.get<T>(key);
    if (cached) return cached;
    
    const fresh = await fetcher();
    await this.set(key, fresh, ttlSeconds);
    return fresh;
  }
}

// Resolver กับ edge cache
const publicResolvers = {
  Query: {
    featuredProducts: async (_: unknown, _args: unknown, { db, kv }: AppContext) => {
      const cache = new EdgeCache(kv);
      
      return cache.getOrFetch(
        'featured_products',
        async () => {
          const { results } = await db.prepare(
            `SELECT * FROM products WHERE is_featured = 1 LIMIT 10`
          ).all();
          return results;
        },
        300  // cache 5 นาที
      );
    },
  },
};
```

---

## 📌 Step 2349: Vercel Edge Functions

```typescript
// pages/api/graphql.ts (Next.js)
// หรือ app/api/graphql/route.ts (App Router)

import { createYoga } from 'graphql-yoga';
import { schema } from '@/lib/schema';

export const config = {
  runtime: 'edge',  // ← ใช้ Edge Runtime
};

const yoga = createYoga({
  schema,
  graphqlEndpoint: '/api/graphql',
  fetchAPI: { Response, Request, fetch },
});

export default yoga;

// Edge Middleware สำหรับ caching
// middleware.ts
import { NextResponse } from 'next/server';

export async function middleware(request: Request) {
  // Cache GET requests (introspection, etc.) ที่ edge
  if (request.method === 'GET') {
    const response = await fetch(request);
    const newResponse = new NextResponse(response.body, response);
    newResponse.headers.set('Cache-Control', 's-maxage=60, stale-while-revalidate');
    return newResponse;
  }
  
  return NextResponse.next();
}
```

---

## 📌 Step 2350: AWS Lambda สำหรับ GraphQL

```typescript
// src/lambda.ts
import serverlessHttp from 'serverless-http';
import express from 'express';
import { expressMiddleware } from '@apollo/server/express4';
import { ApolloServer } from '@apollo/server';
import { schema } from './schema';

const app = express();
app.use(express.json());

// Cold start optimization: สร้าง server นอก handler
let server: ApolloServer | null = null;

async function getServer() {
  if (!server) {
    server = new ApolloServer({ schema });
    await server.start();
  }
  return server;
}

app.use('/graphql', async (req, res, next) => {
  const srv = await getServer();
  return expressMiddleware(srv, {
    context: async ({ req }) => ({
      user: await verifyToken(req.headers.authorization),
      // Lambda: ใช้ RDS Proxy สำหรับ DB connections
      // (Lambda instances หลาย ≠ connection pool)
      db: createDbClient(),
    }),
  })(req, res, next);
});

// Lambda handler
export const handler = serverlessHttp(app);

// serverless.yml
/*
functions:
  graphql:
    handler: src/lambda.handler
    events:
      - http:
          path: /graphql
          method: ANY
          cors: true
    environment:
      DATABASE_URL: ${env:DATABASE_URL}
    vpc:
      securityGroupIds:
        - ${env:SG_ID}
      subnetIds:
        - ${env:SUBNET_ID}
    memorySize: 1024   # เพิ่ม memory → เพิ่ม CPU
    timeout: 29        # API Gateway max
*/
```

---

## 📌 Step 2351: Cold Start Optimization

```typescript
// Cold start เกิดเมื่อ Lambda/Worker เริ่มใหม่
// GraphQL cold start: parse + compile schema ใช้เวลานาน

// 1. Pre-compile schema นอก handler
const schema = buildSchema(typeDefs);
const compiledSchema = buildASTSchema(parse(typeDefs));  // ทำครั้งเดียว

// 2. Prisma: ใช้ connection pooling
// Lambda + Prisma → Prisma Accelerate หรือ RDS Proxy
import { PrismaClient } from '@prisma/client';

let prisma: PrismaClient;

function getPrismaClient() {
  if (!prisma) {
    prisma = new PrismaClient({
      // ลด connections ใน Lambda
      datasources: { db: { url: process.env.DATABASE_URL } },
    });
  }
  return prisma;
}

// 3. Warm-up: EventBridge rule ที่ ping Lambda ทุก 5 นาที
// 4. Provisioned concurrency: Lambda stays warm (มีค่าใช้จ่าย)

// 5. Bundle size: ลด cold start time
// webpack.config.js
module.exports = {
  target: 'node',
  externals: [
    // ไม่ bundle dependencies ที่ใหญ่
    // ใช้ Lambda Layers แทน
    'prisma',
    '@prisma/client',
  ],
};
```

---

## 📌 Step 2352: สรุป Part 086

### เนื้อหาที่เรียนรู้

✅ Cloudflare Workers + D1 + KV  
✅ Vercel Edge Functions  
✅ AWS Lambda + serverless-http  
✅ Cold start optimization  
✅ Edge caching patterns  

### Edge vs Serverless vs Traditional

```
Cloudflare Workers (Edge):
✅ 0ms cold start (V8 isolates)
✅ 200+ PoPs globally
✅ D1 (SQLite), KV, R2
❌ Limited runtime (no Node.js TCP)
❌ No stateful connections

Vercel Edge:
✅ Near-zero cold start
✅ Integrated กับ Next.js
✅ Global distribution
❌ No TCP, no fs

AWS Lambda:
✅ Full Node.js runtime
✅ VPC access → RDS
✅ Flexible memory/CPU
❌ Cold start 200-500ms
❌ Connection pool management

Traditional Server (ECS/K8s):
✅ Full control
✅ WebSocket subscriptions
✅ Long connections
❌ Always-on cost
❌ Manual scaling

GraphQL + Edge: เหมาะสำหรับ read-heavy, cacheable APIs
GraphQL + Lambda: เหมาะสำหรับ mutations, private APIs
GraphQL + K8s: เหมาะสำหรับ subscriptions, complex workloads
```

### ในส่วนถัดไป

➡️ **[Part 087](./part-087.md)** — GraphQL Contract Testing

---

*Part 086 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
