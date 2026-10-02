# Part 089 — GraphQL กับ OAuth 2.0 & OpenID Connect 🔐

> **ระดับ:** Expert | **เวลาเรียน:** 85 นาที | **ขั้นตอนที่:** 2481–2525

---

## 🎯 สิ่งที่จะได้เรียนรู้

- OAuth 2.0 flows ที่เหมาะกับ GraphQL
- Authorization Code + PKCE
- JWT validation ใน resolvers
- Refresh token rotation
- Social login (Google, LINE)
- Scope-based authorization
- Service-to-service auth
- Multi-tenant auth

---

## 📌 Step 2481: OAuth 2.0 Overview สำหรับ GraphQL

```
OAuth 2.0 Flows:

1. Authorization Code + PKCE (SPA/Mobile):
   User → App → Auth Server (Login) → Code → App → Token
   ✅ Most secure สำหรับ user-facing apps
   ✅ ไม่มี client_secret ใน frontend

2. Client Credentials (Service-to-Service):
   Service → Auth Server → Access Token
   ✅ M2M communication
   ✅ ไม่มี user involved

3. Device Flow (CLI/TV):
   Device → Auth Server → User approves on phone → Token
   ✅ สำหรับ devices ที่ไม่มี browser

สิ่งที่ GraphQL ต้องการ:
- Validate JWT ใน context()
- ตรวจ scopes ใน resolvers
- Token refresh ก่อน expire
- Propagate token ไปยัง subgraphs
```

---

## 📌 Step 2482: JWT Validation ใน Context

```typescript
// src/auth/jwt.ts
import { jwtVerify, createRemoteJWKSet } from 'jose';

// JWKS: ดึง public keys จาก auth server
const JWKS = createRemoteJWKSet(
  new URL(`${process.env.AUTH_SERVER_URL}/.well-known/jwks.json`)
);

export interface JWTPayload {
  sub: string;        // User ID
  email: string;
  roles: string[];
  scope: string;      // space-separated: "read:products write:orders"
  iat: number;
  exp: number;
  iss: string;
  aud: string | string[];
}

export async function verifyJWT(token: string): Promise<JWTPayload> {
  const { payload } = await jwtVerify(token, JWKS, {
    issuer: process.env.AUTH_SERVER_URL,
    audience: process.env.API_AUDIENCE,
    algorithms: ['RS256'],
  });
  
  return payload as unknown as JWTPayload;
}

// Context builder
export async function buildContext({ req }: { req: Request }): Promise<AppContext> {
  const authHeader = req.headers.get('authorization');
  
  let user: JWTPayload | null = null;
  
  if (authHeader?.startsWith('Bearer ')) {
    const token = authHeader.slice(7);
    
    try {
      user = await verifyJWT(token);
    } catch (err) {
      // Invalid token: ไม่ throw ที่นี่
      // → resolver ที่ require auth จะ throw เอง
      if (process.env.NODE_ENV === 'development') {
        console.warn('JWT verification failed:', (err as Error).message);
      }
    }
  }
  
  return {
    user,
    prisma,
    redis,
    loaders: createLoaders(prisma),
  };
}
```

---

## 📌 Step 2483: Scope-Based Authorization

```typescript
// src/auth/scopes.ts

// Scopes: fine-grained permissions
const SCOPES = {
  READ_PRODUCTS: 'read:products',
  WRITE_PRODUCTS: 'write:products',
  READ_ORDERS: 'read:orders',
  WRITE_ORDERS: 'write:orders',
  READ_USERS: 'read:users',
  ADMIN: 'admin',
} as const;

type Scope = typeof SCOPES[keyof typeof SCOPES];

function hasScope(user: JWTPayload | null, scope: Scope): boolean {
  if (!user) return false;
  const userScopes = user.scope.split(' ');
  return userScopes.includes(scope) || userScopes.includes(SCOPES.ADMIN);
}

function requireScope(user: JWTPayload | null, scope: Scope): asserts user is JWTPayload {
  if (!user) {
    throw new GraphQLError('Authentication required', {
      extensions: { code: 'UNAUTHENTICATED' },
    });
  }
  
  if (!hasScope(user, scope)) {
    throw new GraphQLError(`Insufficient permissions. Required: ${scope}`, {
      extensions: { code: 'FORBIDDEN', requiredScope: scope },
    });
  }
}

// Resolvers ใช้ scope check
const resolvers = {
  Query: {
    products: async (_: unknown, args: unknown, { user }: AppContext) => {
      requireScope(user, SCOPES.READ_PRODUCTS);
      return prisma.product.findMany();
    },
    
    adminUsers: async (_: unknown, args: unknown, { user }: AppContext) => {
      requireScope(user, SCOPES.ADMIN);
      return prisma.user.findMany();
    },
  },
  
  Mutation: {
    createProduct: async (_: unknown, { input }: CreateProductArgs, { user }: AppContext) => {
      requireScope(user, SCOPES.WRITE_PRODUCTS);
      return prisma.product.create({ data: input });
    },
  },
};
```

---

## 📌 Step 2484: Social Login Mutations

```typescript
// src/resolvers/auth.resolver.ts
import { OAuth2Client } from 'google-auth-library';

const googleClient = new OAuth2Client(process.env.GOOGLE_CLIENT_ID);

const authResolvers = {
  Mutation: {
    // Google Sign-In
    loginWithGoogle: async (_: unknown, { idToken }: { idToken: string }, { prisma }: AppContext) => {
      // Verify Google ID token
      const ticket = await googleClient.verifyIdToken({
        idToken,
        audience: process.env.GOOGLE_CLIENT_ID,
      });
      
      const payload = ticket.getPayload();
      if (!payload?.email) throw new AppError('Invalid Google token', ErrorCode.UNAUTHENTICATED);
      
      // Upsert user
      const user = await prisma.user.upsert({
        where: { email: payload.email },
        create: {
          email: payload.email,
          name: payload.name ?? '',
          avatarUrl: payload.picture,
          provider: 'GOOGLE',
          providerId: payload.sub,
          emailVerified: true,
        },
        update: {
          name: payload.name ?? undefined,
          avatarUrl: payload.picture,
        },
      });
      
      // Issue JWT tokens
      return issueTokens(user);
    },
    
    // Refresh token
    refreshAccessToken: async (_: unknown, { refreshToken }: { refreshToken: string }) => {
      // Verify refresh token
      const payload = await verifyRefreshToken(refreshToken);
      
      // Rotate refresh token (invalidate old)
      await invalidateRefreshToken(refreshToken);
      
      const user = await prisma.user.findUnique({ where: { id: payload.sub } });
      if (!user) throw new AppError('User not found', ErrorCode.NOT_FOUND);
      
      return issueTokens(user);
    },
    
    // Logout
    logout: async (_: unknown, { refreshToken }: { refreshToken: string }, { user }: AppContext) => {
      if (!user) return { success: false };
      
      await invalidateRefreshToken(refreshToken);
      return { success: true };
    },
  },
};

async function issueTokens(user: User) {
  const accessToken = await signJWT({
    sub: user.id,
    email: user.email,
    roles: [user.role],
    scope: getScopesForRole(user.role),
  }, { expiresIn: '15m' });
  
  const refreshToken = await signRefreshToken(user.id, { expiresIn: '30d' });
  
  // Store refresh token (for rotation/invalidation)
  await redis.setex(
    `refresh:${user.id}:${hashToken(refreshToken)}`,
    30 * 24 * 60 * 60,
    '1'
  );
  
  return { accessToken, refreshToken, user };
}
```

---

## 📌 Step 2485: Service-to-Service Auth (M2M)

```typescript
// src/auth/m2m.ts
// Service credentials: ใช้ Client Credentials flow

class M2MTokenManager {
  private tokenCache: { token: string; expiresAt: number } | null = null;
  
  async getToken(): Promise<string> {
    // ใช้ cached token ถ้ายังไม่หมดอายุ (ลบ 60 วินาที เผื่อ clock skew)
    if (this.tokenCache && Date.now() < this.tokenCache.expiresAt - 60000) {
      return this.tokenCache.token;
    }
    
    // Request new token
    const response = await fetch(`${process.env.AUTH_SERVER_URL}/oauth/token`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        grant_type: 'client_credentials',
        client_id: process.env.SERVICE_CLIENT_ID,
        client_secret: process.env.SERVICE_CLIENT_SECRET,
        audience: process.env.API_AUDIENCE,
        scope: 'read:users read:products',
      }),
    });
    
    const { access_token, expires_in } = await response.json();
    
    this.tokenCache = {
      token: access_token,
      expiresAt: Date.now() + expires_in * 1000,
    };
    
    return access_token;
  }
}

const m2mTokenManager = new M2MTokenManager();

// ใช้ใน subgraph calls
async function callSubgraph(subgraphUrl: string, query: string) {
  const token = await m2mTokenManager.getToken();
  
  return fetch(subgraphUrl, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({ query }),
  }).then(r => r.json());
}
```

---

## 📌 Step 2486: Multi-Tenant Auth

```typescript
// src/auth/multi-tenant.ts
// SaaS: user อยู่ใน tenant ต่างๆ มี permissions ต่างกัน

interface TenantJWTPayload extends JWTPayload {
  tenantId: string;
  tenantRoles: Record<string, string>;  // { "tenant-123": "admin", "tenant-456": "member" }
}

// Context เพิ่ม tenantId จาก request
async function buildMultiTenantContext({ req }: { req: Request }): Promise<MultiTenantContext> {
  const baseCtx = await buildContext({ req });
  const tenantId = req.headers.get('x-tenant-id');
  
  if (!tenantId) {
    return { ...baseCtx, tenant: null, tenantRole: null };
  }
  
  // ตรวจสอบว่า user มี access ต่อ tenant
  const tenantPayload = baseCtx.user as TenantJWTPayload | null;
  const tenantRole = tenantPayload?.tenantRoles[tenantId] ?? null;
  
  if (!tenantRole) {
    return { ...baseCtx, tenant: null, tenantRole: null };
  }
  
  const tenant = await prisma.tenant.findUnique({ where: { id: tenantId } });
  
  return { ...baseCtx, tenant, tenantRole };
}

// Row-level security: ทุก query filter ด้วย tenantId
function requireTenant(ctx: MultiTenantContext): asserts ctx is MultiTenantContext & { tenant: Tenant } {
  if (!ctx.tenant) {
    throw new GraphQLError('Tenant access required', {
      extensions: { code: 'TENANT_REQUIRED' },
    });
  }
}

const multiTenantResolvers = {
  Query: {
    tenantProducts: async (_: unknown, _args: unknown, ctx: MultiTenantContext) => {
      requireTenant(ctx);
      // Automatically scoped to tenant
      return prisma.product.findMany({ where: { tenantId: ctx.tenant.id } });
    },
  },
};
```

---

## 📌 Step 2487: สรุป Part 089

### เนื้อหาที่เรียนรู้

✅ JWT validation ด้วย JWKS  
✅ Scope-based authorization  
✅ Social login (Google)  
✅ Refresh token rotation  
✅ Service-to-service M2M auth  
✅ Multi-tenant auth patterns  

### Auth Architecture Checklist

```
Token Handling:
□ Short-lived access tokens (15 min)
□ Refresh token rotation
□ Refresh token blacklisting
□ JWKS endpoint สำหรับ public keys

Authorization:
□ Scope-based (not role-based only)
□ Resource ownership check
□ Row-level security (tenants)
□ Field-level masking สำหรับ sensitive data

Security:
□ HTTPS only
□ httpOnly cookies สำหรับ refresh tokens
□ CSRF protection
□ Token binding (optional)
□ Revocation mechanism
```

### ในส่วนถัดไป

➡️ **[Part 090](./part-090.md)** — Advanced DataLoader Patterns

---

*Part 089 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
