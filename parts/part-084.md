# Part 084 — Custom GraphQL Directives 🔮

> **ระดับ:** Expert | **เวลาเรียน:** 80 นาที | **ขั้นตอนที่:** 2256–2300

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Schema directives ทำงานอย่างไร
- `@auth` directive
- `@deprecated` และ custom deprecation
- `@rateLimit` directive
- `@cacheControl` directive
- `@formatDate` field transformer
- `@validate` input directive
- Directive composition

---

## 📌 Step 2256: Directive Basics

```graphql
# Directives คือ annotations บน schema elements
# สามารถใช้บน: FIELD_DEFINITION, OBJECT, ARGUMENT, INPUT_FIELD, ENUM_VALUE

# ตัวอย่าง: @auth directive
directive @auth(
  requires: Role = USER
) on FIELD_DEFINITION | OBJECT

enum Role {
  GUEST
  USER
  ADMIN
  SUPERADMIN
}

type Query {
  # Public: ไม่ต้อง auth
  products: [Product!]!
  
  # ต้อง login
  myOrders: [Order!]! @auth(requires: USER)
  
  # ต้องเป็น admin
  adminStats: AdminStats! @auth(requires: ADMIN)
}

type User {
  id: ID!
  email: String!
  
  # Sensitive fields: ต้องเป็น admin
  creditScore: Int @auth(requires: ADMIN)
  internalNotes: String @auth(requires: ADMIN)
}
```

---

## 📌 Step 2257: Implement @auth Directive

```typescript
// src/directives/auth.directive.ts
import { getDirective, MapperKind, mapSchema } from '@graphql-tools/utils';
import { GraphQLSchema, defaultFieldResolver } from 'graphql';

function authDirectiveTransformer(schema: GraphQLSchema, directiveName = 'auth') {
  return mapSchema(schema, {
    // Transform object types
    [MapperKind.OBJECT_FIELD]: (fieldConfig, fieldName, typeName) => {
      const authDirective = getDirective(schema, fieldConfig, directiveName)?.[0];
      
      if (!authDirective) return fieldConfig;
      
      const { resolve = defaultFieldResolver } = fieldConfig;
      const { requires } = authDirective;
      
      return {
        ...fieldConfig,
        resolve: async (source, args, context, info) => {
          // ตรวจ authentication
          if (!context.user) {
            throw new GraphQLError('Unauthenticated', {
              extensions: { code: 'UNAUTHENTICATED' },
            });
          }
          
          // ตรวจ authorization (role hierarchy)
          const roleHierarchy = ['GUEST', 'USER', 'ADMIN', 'SUPERADMIN'];
          const userRoleIndex = roleHierarchy.indexOf(context.user.role);
          const requiredRoleIndex = roleHierarchy.indexOf(requires);
          
          if (userRoleIndex < requiredRoleIndex) {
            throw new GraphQLError(`Requires ${requires} role`, {
              extensions: { code: 'FORBIDDEN' },
            });
          }
          
          return resolve(source, args, context, info);
        },
      };
    },
  });
}

// ใน server setup
import { makeExecutableSchema } from '@graphql-tools/schema';

let schema = makeExecutableSchema({ typeDefs, resolvers });
schema = authDirectiveTransformer(schema, 'auth');
```

---

## 📌 Step 2258: @rateLimit Directive

```typescript
// src/directives/rate-limit.directive.ts
import { getDirective, MapperKind, mapSchema } from '@graphql-tools/utils';
import { defaultFieldResolver } from 'graphql';

// @rateLimit(window: "60s", max: 10) on FIELD_DEFINITION
function rateLimitDirectiveTransformer(schema: GraphQLSchema) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const directive = getDirective(schema, fieldConfig, 'rateLimit')?.[0];
      if (!directive) return fieldConfig;
      
      const { resolve = defaultFieldResolver } = fieldConfig;
      const { window, max } = directive;
      
      // Parse window: "60s" → 60, "1m" → 60, "1h" → 3600
      const windowSeconds = parseWindow(window);
      
      return {
        ...fieldConfig,
        resolve: async (source, args, context, info) => {
          const key = `rateLimit:${context.user?.id ?? context.ip}:${info.fieldName}`;
          
          const [count] = await context.redis.multi()
            .incr(key)
            .expire(key, windowSeconds)
            .exec() as [number, number];
          
          if (count > max) {
            throw new GraphQLError('Rate limit exceeded', {
              extensions: {
                code: 'RATE_LIMITED',
                retryAfter: windowSeconds,
              },
            });
          }
          
          return resolve(source, args, context, info);
        },
      };
    },
  });
}

function parseWindow(window: string): number {
  const units: Record<string, number> = { s: 1, m: 60, h: 3600, d: 86400 };
  const match = window.match(/^(\d+)([smhd])$/);
  if (!match) throw new Error(`Invalid window: ${window}`);
  return parseInt(match[1]) * (units[match[2]] ?? 1);
}
```

```graphql
# Usage ใน schema
type Mutation {
  sendVerificationCode(phone: String!): Boolean!
    @auth(requires: USER)
    @rateLimit(window: "1m", max: 3)
  
  createProduct(input: CreateProductInput!): Product!
    @auth(requires: ADMIN)
    @rateLimit(window: "1h", max: 100)
}
```

---

## 📌 Step 2259: @cacheControl Directive

```typescript
// src/directives/cache-control.directive.ts
// ควบคุม HTTP cache headers ตาม field

function cacheControlDirectiveTransformer(schema: GraphQLSchema) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const directive = getDirective(schema, fieldConfig, 'cacheControl')?.[0];
      if (!directive) return fieldConfig;
      
      const { resolve = defaultFieldResolver } = fieldConfig;
      const { maxAge, scope } = directive;  // scope: PUBLIC | PRIVATE
      
      return {
        ...fieldConfig,
        resolve: async (source, args, context, info) => {
          const result = await resolve(source, args, context, info);
          
          // Set cache hint สำหรับ response
          info.cacheControl?.setCacheHint({ maxAge, scope });
          
          return result;
        },
      };
    },
  });
}

// Plugin: แปลง cache hints เป็น HTTP headers
const cachePlugin: ApolloServerPlugin = {
  async requestDidStart() {
    return {
      async willSendResponse({ response, overallCachePolicy }) {
        if (overallCachePolicy?.maxAge) {
          const scope = overallCachePolicy.scope?.toLowerCase() ?? 'private';
          response.http.headers.set(
            'Cache-Control',
            `${scope}, max-age=${overallCachePolicy.maxAge}`
          );
        }
      },
    };
  },
};
```

```graphql
directive @cacheControl(
  maxAge: Int
  scope: CacheControlScope
) on FIELD_DEFINITION | OBJECT | INTERFACE | UNION

enum CacheControlScope {
  PUBLIC
  PRIVATE
}

type Query {
  products: [Product!]! @cacheControl(maxAge: 60, scope: PUBLIC)
  myProfile: User! @cacheControl(maxAge: 0, scope: PRIVATE)
  publicStats: SiteStats! @cacheControl(maxAge: 3600, scope: PUBLIC)
}
```

---

## 📌 Step 2260: @formatDate Directive (Field Transformer)

```typescript
// src/directives/format-date.directive.ts
// Transform ค่า Date ตาม format ที่กำหนด

function formatDateTransformer(schema: GraphQLSchema) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const directive = getDirective(schema, fieldConfig, 'formatDate')?.[0];
      if (!directive) return fieldConfig;
      
      const { resolve = defaultFieldResolver } = fieldConfig;
      const { format, timezone } = directive;
      
      return {
        ...fieldConfig,
        resolve: async (source, args, context, info) => {
          const date = await resolve(source, args, context, info);
          if (!date) return null;
          
          // ใช้ context locale สำหรับ formatting
          const locale = context.locale ?? 'th';
          const tz = timezone ?? context.timezone ?? 'Asia/Bangkok';
          
          return new Intl.DateTimeFormat(localeMap[locale], {
            ...parseFormat(format),
            timeZone: tz,
          }).format(new Date(date));
        },
      };
    },
  });
}

function parseFormat(format: string) {
  const formats: Record<string, Intl.DateTimeFormatOptions> = {
    DATE: { year: 'numeric', month: '2-digit', day: '2-digit' },
    TIME: { hour: '2-digit', minute: '2-digit' },
    DATETIME: { year: 'numeric', month: '2-digit', day: '2-digit', hour: '2-digit', minute: '2-digit' },
    RELATIVE: undefined, // จัดการแบบพิเศษ
  };
  return formats[format] ?? formats.DATETIME;
}
```

```graphql
directive @formatDate(
  format: DateFormat = DATETIME
  timezone: String
) on FIELD_DEFINITION

enum DateFormat {
  DATE
  TIME
  DATETIME
  RELATIVE
}

type Order {
  id: ID!
  createdAt: String! @formatDate(format: DATETIME)
  deliveredAt: String @formatDate(format: DATE)
}
```

---

## 📌 Step 2261: @validate Input Directive

```typescript
// src/directives/validate.directive.ts
// Validate input arguments ด้วย directive

function validateDirectiveTransformer(schema: GraphQLSchema) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const directive = getDirective(schema, fieldConfig, 'validate')?.[0];
      if (!directive) return fieldConfig;
      
      const { resolve = defaultFieldResolver } = fieldConfig;
      
      return {
        ...fieldConfig,
        resolve: (source, args, context, info) => {
          // Validate input args ตาม directive params
          validateArgs(args, directive);
          return resolve(source, args, context, info);
        },
      };
    },
  });
}

function validateArgs(args: Record<string, unknown>, rules: Record<string, unknown>) {
  for (const [key, value] of Object.entries(args)) {
    if (rules.minLength && typeof value === 'string' && value.length < (rules.minLength as number)) {
      throw new GraphQLError(`${key} must be at least ${rules.minLength} characters`, {
        extensions: { code: 'VALIDATION_ERROR', field: key },
      });
    }
    if (rules.maxLength && typeof value === 'string' && value.length > (rules.maxLength as number)) {
      throw new GraphQLError(`${key} must be at most ${rules.maxLength} characters`, {
        extensions: { code: 'VALIDATION_ERROR', field: key },
      });
    }
    if (rules.pattern && typeof value === 'string' && !new RegExp(rules.pattern as string).test(value)) {
      throw new GraphQLError(`${key} has invalid format`, {
        extensions: { code: 'VALIDATION_ERROR', field: key },
      });
    }
  }
}
```

---

## 📌 Step 2262: Directive Composition

```typescript
// src/server.ts: apply ทุก directives

import { makeExecutableSchema } from '@graphql-tools/schema';

let schema = makeExecutableSchema({ typeDefs, resolvers });

// Apply directives ตามลำดับที่สำคัญ
schema = authDirectiveTransformer(schema, 'auth');
schema = rateLimitDirectiveTransformer(schema);
schema = cacheControlDirectiveTransformer(schema);
schema = formatDateTransformer(schema);
schema = validateDirectiveTransformer(schema);

const server = new ApolloServer({
  schema,
  plugins: [cachePlugin],
});
```

---

## 📌 Step 2263: สรุป Part 084

### เนื้อหาที่เรียนรู้

✅ Directive transformer pattern  
✅ `@auth` สำหรับ authorization  
✅ `@rateLimit` per-field rate limiting  
✅ `@cacheControl` HTTP caching  
✅ `@formatDate` field transformer  
✅ `@validate` input validation  
✅ Directive composition  

### เมื่อไหร่ควรใช้ Directives

```
ใช้ Directives เมื่อ:
✅ Cross-cutting concerns (auth, rate limit, cache)
✅ Declarative → อ่านง่ายกว่าใน resolver
✅ Reusable across many fields

ไม่ใช้ Directives เมื่อ:
❌ Logic ซับซ้อนเฉพาะ field เดียว
❌ ต้องการ access context ที่ complex
❌ Business logic (ควรใส่ใน resolver/service)

Performance Note:
- Directive wrappers เพิ่ม overhead ต่อ call
- สำหรับ @rateLimit: ทุก field call จะ hit Redis
- ใช้ @cacheControl ช่วยลด load ได้มาก
```

### ในส่วนถัดไป

➡️ **[Part 085](./part-085.md)** — GraphQL Plugins & Envelop Ecosystem

---

*Part 084 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
