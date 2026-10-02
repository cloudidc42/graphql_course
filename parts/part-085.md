# Part 085 — GraphQL Plugins & Envelop Ecosystem 🔌

> **ระดับ:** Expert | **เวลาเรียน:** 75 นาที | **ขั้นตอนที่:** 2301–2345

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Envelop คืออะไร
- useDepthLimit plugin
- useResponseCache plugin
- useDataLoader plugin
- useArmor (security)
- สร้าง custom Envelop plugin
- Apollo Server plugins vs Envelop plugins
- Plugin composition patterns

---

## 📌 Step 2301: Envelop คืออะไร

```
Envelop = Middleware layer สำหรับ GraphQL execution pipeline

ทำงานตรงไหน:
- ก่อน/หลัง parse
- ก่อน/หลัง validate
- ก่อน/หลัง execute
- ก่อน/หลัง subscribe

เหมาะสำหรับ:
- Framework-agnostic plugins
- ใช้กับ Apollo Server, Yoga, Mercurius, Fastify

ความสัมพันธ์:
Apollo Server plugins ← ทำงานกับ HTTP layer
Envelop plugins      ← ทำงานกับ GraphQL execution layer
```

```bash
npm install graphql-yoga @envelop/core @envelop/depth-limit @envelop/response-cache @envelop/dataloader @escape.tech/graphql-armor
```

---

## 📌 Step 2302: GraphQL Yoga + Envelop Setup

```typescript
// src/server.ts
import { createYoga } from 'graphql-yoga';
import { useDepthLimit } from '@envelop/depth-limit';
import { useResponseCache } from '@envelop/response-cache';
import { createRedisCache } from '@envelop/response-cache-redis';
import { useDataLoader } from '@envelop/dataloader';
import { createGraphQLArmor } from '@escape.tech/graphql-armor';

const armor = createGraphQLArmor({
  // Security hardening defaults
  maxDepth: { n: 10 },
  maxDirectives: { n: 50 },
  maxAliases: { n: 15 },
  costLimit: { maxCost: 5000 },
  blockFieldSuggestion: { enabled: process.env.NODE_ENV === 'production' },
});

const yoga = createYoga({
  schema,
  plugins: [
    // Security
    ...armor.protect(),
    
    // Depth limit (ซ้อนกันได้ไม่เกิน 10 ชั้น)
    useDepthLimit({ maxDepth: 10 }),
    
    // Response cache ด้วย Redis
    useResponseCache({
      cache: createRedisCache({ redis }),
      ttl: 60000,  // 60 วินาที
      ttlPerType: {
        // Custom TTL per type
        Product: 300000,    // 5 นาที
        UserProfile: 0,     // ไม่ cache
        PublicStats: 3600000, // 1 ชั่วโมง
      },
      ttlPerSchemaCoordinate: {
        'Query.products': 60000,
        'Query.myProfile': 0,
      },
      // ไม่ cache requests ที่มี Authorization
      enabled: ({ request }) => !request.headers.get('authorization'),
    }),
    
    // DataLoader per request
    useDataLoader('userById', (ctx) => createUserByIdLoader(ctx.prisma)),
    useDataLoader('productById', (ctx) => createProductByIdLoader(ctx.prisma)),
  ],
  
  context: async ({ request }) => ({
    user: await verifyToken(request),
    prisma,
    redis,
  }),
});
```

---

## 📌 Step 2303: สร้าง Custom Envelop Plugin

```typescript
// src/plugins/operation-complexity.plugin.ts
import { Plugin } from '@envelop/core';
import { getComplexity, simpleEstimator, fieldExtensionsEstimator } from 'graphql-query-complexity';

export function useOperationComplexity(options: {
  maxComplexity: number;
  onComplexity?: (complexity: number, context: unknown) => void;
}): Plugin {
  return {
    onExecute({ args, extendContext }) {
      const complexity = getComplexity({
        schema: args.schema,
        query: args.document,
        variables: args.variableValues ?? {},
        estimators: [
          fieldExtensionsEstimator(),
          simpleEstimator({ defaultComplexity: 1 }),
        ],
      });
      
      options.onComplexity?.(complexity, args.contextValue);
      
      if (complexity > options.maxComplexity) {
        throw new GraphQLError(
          `Query complexity ${complexity} exceeds maximum ${options.maxComplexity}`,
          { extensions: { code: 'COMPLEXITY_LIMIT_EXCEEDED', complexity } }
        );
      }
      
      // Expose complexity ใน context
      extendContext({ queryComplexity: complexity });
    },
  };
}

// src/plugins/request-logging.plugin.ts
export function useRequestLogging(): Plugin {
  return {
    onExecute({ args }) {
      const start = Date.now();
      const operationName = args.operationName ?? 'anonymous';
      
      return {
        onExecuteDone({ result }) {
          const duration = Date.now() - start;
          const hasErrors = Array.isArray(result) 
            ? result.some(r => r.errors)
            : !!result.errors;
          
          logger.info({
            operation: operationName,
            duration,
            hasErrors,
            userId: (args.contextValue as AppContext).user?.id,
          });
          
          // Metrics
          operationHistogram.observe(
            { operation: operationName, status: hasErrors ? 'error' : 'success' },
            duration / 1000
          );
        },
      };
    },
  };
}
```

---

## 📌 Step 2304: Apollo Server Plugin Pattern

```typescript
// src/plugins/apollo-plugins.ts
// สำหรับผู้ที่ใช้ Apollo Server แทน Yoga

import { ApolloServerPlugin, GraphQLRequestListener } from '@apollo/server';

// Plugin lifecycle:
// serverWillStart
//   └── requestDidStart
//         ├── parsingDidStart
//         ├── validationDidStart
//         ├── executionDidStart
//         │     └── willResolveField  (per field)
//         └── willSendResponse

export function createMetricsPlugin(): ApolloServerPlugin {
  return {
    async serverWillStart() {
      logger.info('GraphQL server starting');
    },
    
    async requestDidStart({ request, contextValue }): Promise<GraphQLRequestListener> {
      const start = Date.now();
      const operationName = request.operationName ?? 'anonymous';
      
      return {
        async parsingDidStart() {
          return async (err) => {
            if (err) logger.error({ type: 'parse_error', error: err });
          };
        },
        
        async validationDidStart() {
          return async (errors) => {
            if (errors?.length) {
              logger.warn({ type: 'validation_errors', count: errors.length });
            }
          };
        },
        
        async executionDidStart() {
          return {
            willResolveField({ info }) {
              const fieldStart = performance.now();
              
              return (error, result) => {
                const duration = performance.now() - fieldStart;
                
                // Track slow fields
                if (duration > 100) {
                  logger.warn({
                    type: 'slow_field',
                    field: `${info.parentType.name}.${info.fieldName}`,
                    duration,
                  });
                }
              };
            },
          };
        },
        
        async willSendResponse({ response }) {
          const duration = Date.now() - start;
          const hasErrors = !!response.body.singleResult?.errors?.length;
          
          operationDuration.observe(
            { operation: operationName, status: hasErrors ? 'error' : 'ok' },
            duration / 1000
          );
          
          // Add timing header
          response.http.headers.set('X-Response-Time', `${duration}ms`);
        },
      };
    },
  };
}
```

---

## 📌 Step 2305: Plugin Composition Pattern

```typescript
// src/plugins/index.ts
// รวม plugins ทั้งหมดในที่เดียว

import { Plugin } from '@envelop/core';

interface PluginConfig {
  env: 'development' | 'production' | 'test';
  redis: Redis;
  enableCache: boolean;
  maxComplexity: number;
  maxDepth: number;
}

export function createPlugins(config: PluginConfig): Plugin[] {
  const plugins: Plugin[] = [
    // Security (always on)
    ...createGraphQLArmor({
      maxDepth: { n: config.maxDepth },
      blockFieldSuggestion: { enabled: config.env === 'production' },
    }).protect(),
    
    // Logging
    useRequestLogging(),
    
    // Complexity
    useOperationComplexity({
      maxComplexity: config.maxComplexity,
      onComplexity: (complexity) => {
        complexityHistogram.observe(complexity);
      },
    }),
  ];
  
  // Cache: production only
  if (config.enableCache && config.env === 'production') {
    plugins.push(
      useResponseCache({
        cache: createRedisCache({ redis: config.redis }),
        ttl: 60000,
      })
    );
  }
  
  // Dev tools
  if (config.env === 'development') {
    plugins.push(useLogger());
  }
  
  return plugins;
}
```

---

## 📌 Step 2306: สรุป Part 085

### เนื้อหาที่เรียนรู้

✅ Envelop plugin system  
✅ GraphQL Yoga setup  
✅ useResponseCache ด้วย Redis  
✅ useOperationComplexity custom plugin  
✅ Apollo Server plugin lifecycle  
✅ Plugin composition  

### Plugin Ecosystem Map

```
Security:
  @escape.tech/graphql-armor → depth, cost, aliases, directives
  
Performance:
  @envelop/response-cache → operation-level caching
  @envelop/dataloader     → automatic DataLoader per request
  
Observability:
  @envelop/opentelemetry  → OTLP tracing
  @envelop/prometheus     → Prometheus metrics
  
DX:
  @envelop/parser-cache   → parse + validate result caching
  @envelop/validation-cache → schema validation caching
  
Use Cases:
  Apollo Server Plugins   → HTTP layer, usage reporting, lifecycle hooks
  Envelop Plugins         → Framework-agnostic GraphQL execution hooks
```

### ในส่วนถัดไป

➡️ **[Part 086](./part-086.md)** — GraphQL กับ Edge Computing & WebAssembly

---

*Part 085 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
