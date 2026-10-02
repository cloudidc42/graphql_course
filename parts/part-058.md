# Part 058 — Multi-Tenant GraphQL Architecture 🏢

> **ระดับ:** Expert | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1231–1270

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Multi-tenancy patterns
- Tenant isolation strategies
- Row-level security (RLS)
- Schema-per-tenant
- Database-per-tenant
- Tenant context propagation
- Cross-tenant queries (admin)
- Tenant provisioning
- White-label GraphQL APIs

---

## 📌 Step 1231: Multi-Tenancy Patterns

```
3 patterns หลัก:

1. Shared Database, Shared Schema (Row-Level Security)
   - tenantId column ใน tables
   - RLS policy ที่ database level
   - ✅ ง่าย, ถูก
   - ❌ data isolation น้อยกว่า

2. Shared Database, Separate Schema
   - แต่ละ tenant มี PostgreSQL schema ของตัวเอง
   - users_tenant_a, users_tenant_b
   - ✅ isolation ดีขึ้น
   - ❌ migration ซับซ้อน

3. Database-per-Tenant
   - แต่ละ tenant มี database เป็นของตัวเอง
   - ✅ isolation สูงสุด
   - ❌ แพง, scale ยาก
   - เหมาะสำหรับ enterprise customers

ส่วนใหญ่ใช้ Row-Level Security ก่อน แล้วค่อย scale
```

---

## 📌 Step 1232: Tenant Context

```typescript
// src/multitenancy/tenant.ts

export interface Tenant {
  id: string;
  slug: string;
  name: string;
  plan: 'starter' | 'professional' | 'enterprise';
  features: Set<string>;
  dbConnectionString?: string; // สำหรับ DB-per-tenant
  rateLimits: {
    requestsPerMin: number;
    queryComplexity: number;
  };
}

// Tenant resolution — จาก request
export async function resolveTenant(req: Request): Promise<Tenant | null> {
  // Option 1: Subdomain (tenant1.app.com)
  const host = req.headers.get('host') ?? '';
  const subdomain = host.split('.')[0];
  if (subdomain && subdomain !== 'www') {
    return findTenantBySlug(subdomain);
  }
  
  // Option 2: Custom domain (customer.com → app.com)
  const customDomain = await findTenantByCustomDomain(host);
  if (customDomain) return customDomain;
  
  // Option 3: API Key (for API clients)
  const apiKey = req.headers.get('x-api-key');
  if (apiKey) return findTenantByApiKey(apiKey);
  
  // Option 4: JWT claim
  const token = req.headers.get('authorization')?.split(' ')[1];
  if (token) {
    const decoded = verifyToken(token);
    if (decoded?.tenantId) return findTenantById(decoded.tenantId);
  }
  
  return null;
}

// Feature flags per tenant
export function tenantHasFeature(tenant: Tenant, feature: string): boolean {
  return tenant.features.has(feature);
}

// Tenant-specific rate limits
export function getTenantRateLimit(tenant: Tenant) {
  const planLimits = {
    starter: { requestsPerMin: 60, queryComplexity: 500 },
    professional: { requestsPerMin: 300, queryComplexity: 2000 },
    enterprise: { requestsPerMin: 1000, queryComplexity: 10000 },
  };
  
  return planLimits[tenant.plan];
}
```

---

## 📌 Step 1233: Prisma กับ Row-Level Security

```typescript
// src/multitenancy/prisma-tenant.ts
// Prisma middleware สำหรับ automatic tenant filtering

import { PrismaClient, Prisma } from '@prisma/client';

// Models ที่ต้องการ tenant isolation
const TENANT_MODELS = new Set([
  'User', 'Product', 'Order', 'Category', 'Review',
]);

// สร้าง Prisma client ที่ filter tenantId อัตโนมัติ
export function createTenantPrisma(tenantId: string): PrismaClient {
  const prisma = new PrismaClient();
  
  // Middleware: inject tenantId ใน queries
  prisma.$use(async (params, next) => {
    if (!TENANT_MODELS.has(params.model ?? '')) {
      return next(params);
    }
    
    // Inject tenantId ใน WHERE clauses
    if (['findFirst', 'findMany', 'count', 'aggregate'].includes(params.action)) {
      params.args.where = {
        ...params.args.where,
        tenantId,
      };
    }
    
    if (['findUnique', 'findUniqueOrThrow'].includes(params.action)) {
      // แปลง findUnique → findFirst เพื่อ add tenantId
      params.action = 'findFirst';
      params.args.where = {
        ...params.args.where,
        tenantId,
      };
    }
    
    // Inject tenantId ใน create/createMany
    if (params.action === 'create') {
      params.args.data = {
        ...params.args.data,
        tenantId,
      };
    }
    
    if (params.action === 'createMany') {
      params.args.data = params.args.data.map((d: Record<string, unknown>) => ({
        ...d,
        tenantId,
      }));
    }
    
    // Update: ตรวจสอบ tenantId
    if (['update', 'delete'].includes(params.action)) {
      params.args.where = {
        ...params.args.where,
        tenantId,
      };
    }
    
    return next(params);
  });
  
  return prisma;
}

// Context factory
export async function createContext({ req }) {
  const tenant = await resolveTenant(req);
  
  if (!tenant) {
    return { prisma: null, tenant: null, user: null };
  }
  
  // ถ้า DB-per-tenant: สร้าง connection ไปยัง tenant's DB
  const prisma = tenant.dbConnectionString
    ? new PrismaClient({ datasources: { db: { url: tenant.dbConnectionString } } })
    : createTenantPrisma(tenant.id);
  
  // Resolve user (ต้องอยู่ใน tenant นี้ด้วย)
  const user = await resolveUser(req, prisma);
  
  return { prisma, tenant, user };
}
```

---

## 📌 Step 1234: PostgreSQL Row-Level Security

```sql
-- Database-level RLS (ปลอดภัยกว่า Application-level)

-- Enable RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- สร้าง policy
CREATE POLICY tenant_isolation ON users
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

CREATE POLICY tenant_isolation ON products
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- Admin ข้าม RLS ได้
CREATE POLICY admin_bypass ON users
  USING (current_setting('app.is_admin', true) = 'true');
```

```typescript
// Set tenant context ก่อน query
async function setTenantContext(prisma: PrismaClient, tenantId: string) {
  await prisma.$executeRaw`
    SELECT set_config('app.tenant_id', ${tenantId}, true)
  `;
}

// ใช้ใน middleware
prisma.$use(async (params, next) => {
  // Set config สำหรับ RLS
  await prisma.$executeRaw`
    SELECT set_config('app.tenant_id', ${tenantId}::text, true)
  `;
  return next(params);
});
```

---

## 📌 Step 1235: Tenant-Aware GraphQL Schema

```graphql
# Tenant-specific schema extensions
# แต่ละ tenant อาจมี custom fields

# Base schema
type Product {
  id: ID!
  name: String!
  price: Float!
  # Dynamic fields จาก tenant configuration
  customAttributes: JSON
}

# Tenant provisioning
type Mutation {
  # Admin: สร้าง tenant ใหม่
  createTenant(input: CreateTenantInput!): Tenant!
  
  # Tenant admin: configure schema
  configureTenantSchema(input: TenantSchemaInput!): TenantSchema!
  
  # Feature flags
  enableFeature(tenantId: ID!, feature: String!): Tenant!
  disableFeature(tenantId: ID!, feature: String!): Tenant!
}

type Query {
  # Tenant info
  currentTenant: Tenant!
  
  # Admin: cross-tenant queries (requires admin role)
  tenants(first: Int, after: String): TenantConnection!
  tenantStats(tenantId: ID!): TenantStats!
}

type Tenant {
  id: ID!
  name: String!
  slug: String!
  plan: TenantPlan!
  createdAt: DateTime!
  userCount: Int!
  storageUsed: Float! # bytes
  monthlyApiCalls: Int!
}
```

---

## 📌 Step 1236: White-Label GraphQL API

```typescript
// src/multitenancy/whitelabel.ts
// Tenant-specific branding + customization

interface TenantBranding {
  tenantId: string;
  primaryColor: string;
  logoUrl: string;
  appName: string;
  supportEmail: string;
  customDomain?: string;
}

// GraphQL endpoint per tenant
// tenant1.api.example.com/graphql → tenant1
// tenant2.api.example.com/graphql → tenant2

// Tenant-specific rate limits ใน Apollo Server plugin
const tenantRateLimitPlugin = {
  async requestDidStart(context) {
    const tenant = context.contextValue?.tenant as Tenant;
    if (!tenant) return;
    
    const limits = getTenantRateLimit(tenant);
    
    return {
      async executionDidStart() {
        // Apply tenant-specific complexity limit
        return {
          willResolveField() {
            // Track complexity per tenant
          },
        };
      },
    };
  },
};

// Tenant feature gating in resolvers
function requireFeature(feature: string) {
  return function (resolver: GraphQLResolver) {
    return async (parent, args, ctx, info) => {
      const tenant = ctx.tenant as Tenant;
      
      if (!tenantHasFeature(tenant, feature)) {
        throw new GraphQLError(`Feature "${feature}" is not available on your plan`, {
          extensions: {
            code: 'FEATURE_NOT_AVAILABLE',
            currentPlan: tenant.plan,
            upgradeRequired: true,
          }
        });
      }
      
      return resolver(parent, args, ctx, info);
    };
  };
}

// Usage
const advancedAnalyticsResolver = requireFeature('advanced-analytics')(
  async (_, args, ctx) => {
    return ctx.prisma.analytics.findMany({ /* ... */ });
  }
);
```

---

## 📌 Step 1237: สรุป Part 058

### เนื้อหาที่เรียนรู้

✅ Multi-tenancy patterns  
✅ Tenant resolution strategies  
✅ Prisma middleware สำหรับ tenant isolation  
✅ PostgreSQL RLS  
✅ Feature flags per tenant  
✅ White-label API patterns  

### Migration Path

```
Phase 1 (Start): Shared DB, Row-Level Security
- tenantId column ในทุก table
- Prisma middleware inject tenantId
- เร็ว, ถูก

Phase 2 (Growth): PostgreSQL RLS
- เพิ่ม RLS policies ที่ DB level
- ป้องกัน application bugs

Phase 3 (Enterprise): Schema-per-tenant
- PostgreSQL schemas แยกกัน
- Migration tools per schema

Phase 4 (Scale): Database-per-tenant
- เฉพาะ enterprise customers ที่ต้องการ
- Infrastructure cost สูงขึ้น
```

### ในส่วนถัดไป

➡️ **[Part 059](./part-059.md)** — Real-Time Collaboration

---

*Part 058 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
