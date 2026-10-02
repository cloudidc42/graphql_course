# Part 083 — Advanced TypeScript กับ GraphQL 🔷

> **ระดับ:** Expert | **เวลาเรียน:** 85 นาที | **ขั้นตอนที่:** 2211–2255

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Conditional types ใน resolver generics
- Branded types สำหรับ IDs
- Discriminated unions กับ error handling
- Type-safe DataLoaders
- Mapped types สำหรับ resolvers
- DeepPartial ใน tests
- Infer ใน codegen types
- Strict context typing

---

## 📌 Step 2211: Branded Types สำหรับ IDs

```typescript
// src/types/branded.ts
// ป้องกัน mixing ProductId กับ UserId

declare const __brand: unique symbol;
type Brand<T, B> = T & { [__brand]: B };

export type UserId = Brand<string, 'UserId'>;
export type ProductId = Brand<string, 'ProductId'>;
export type OrderId = Brand<string, 'OrderId'>;

// ฟังก์ชัน type-safe constructors
export const asUserId = (id: string): UserId => id as UserId;
export const asProductId = (id: string): ProductId => id as ProductId;
export const asOrderId = (id: string): OrderId => id as OrderId;

// Resolvers ใช้ branded types
const userResolvers = {
  Query: {
    user: async (_: unknown, { id }: { id: UserId }, ctx: AppContext) => {
      return ctx.prisma.user.findUnique({ where: { id } });
    },
    
    // TypeScript error ถ้าส่ง ProductId ที่นี่
    // user(id: productId) ← compile error
  },
};
```

---

## 📌 Step 2212: Discriminated Unions สำหรับ Error Handling

```typescript
// src/types/result.ts
// Type-safe result type ที่ไม่ throw

type Success<T> = { success: true; data: T };
type Failure<E = string> = { success: false; error: E };
type Result<T, E = string> = Success<T> | Failure<E>;

// ใน resolvers
async function getUserById(id: UserId, ctx: AppContext): Promise<Result<User, 'NOT_FOUND' | 'FORBIDDEN'>> {
  const user = await ctx.prisma.user.findUnique({ where: { id } });
  
  if (!user) return { success: false, error: 'NOT_FOUND' };
  if (user.id !== ctx.user?.id && ctx.user?.role !== 'ADMIN') {
    return { success: false, error: 'FORBIDDEN' };
  }
  
  return { success: true, data: user };
}

// Resolver: แปลง Result เป็น GraphQL error union
const userResolvers = {
  Query: {
    user: async (_: unknown, { id }: { id: string }, ctx: AppContext) => {
      const result = await getUserById(asUserId(id), ctx);
      
      if (!result.success) {
        // TypeScript รู้ว่า error เป็น 'NOT_FOUND' หรือ 'FORBIDDEN'
        switch (result.error) {
          case 'NOT_FOUND':
            throw new AppError('User not found', ErrorCode.NOT_FOUND);
          case 'FORBIDDEN':
            throw new AppError('Access denied', ErrorCode.FORBIDDEN);
        }
      }
      
      return result.data;  // TypeScript รู้ว่าเป็น User
    },
  },
};
```

---

## 📌 Step 2213: Type-Safe DataLoaders

```typescript
// src/loaders/typed-loaders.ts
import DataLoader from 'dataloader';

// Generic loader factory
function createLoader<K, V>(
  batchFn: (keys: readonly K[]) => Promise<(V | Error)[]>
): DataLoader<K, V> {
  return new DataLoader(batchFn, {
    maxBatchSize: 100,
    cacheKeyFn: (key) => String(key),
  });
}

// Typed loaders
export type AppLoaders = {
  userById: DataLoader<UserId, User | null>;
  productById: DataLoader<ProductId, Product | null>;
  ordersByUserId: DataLoader<UserId, Order[]>;
};

export function createLoaders(prisma: PrismaClient): AppLoaders {
  return {
    userById: createLoader(async (ids) => {
      const users = await prisma.user.findMany({
        where: { id: { in: [...ids] } },
      });
      const map = new Map(users.map(u => [u.id as UserId, u]));
      return ids.map(id => map.get(id) ?? null);
    }),
    
    productById: createLoader(async (ids) => {
      const products = await prisma.product.findMany({
        where: { id: { in: [...ids] } },
      });
      const map = new Map(products.map(p => [p.id as ProductId, p]));
      return ids.map(id => map.get(id) ?? null);
    }),
    
    ordersByUserId: createLoader(async (userIds) => {
      const orders = await prisma.order.findMany({
        where: { userId: { in: [...userIds] } },
      });
      const map = new Map<UserId, Order[]>();
      for (const order of orders) {
        const userId = order.userId as UserId;
        const arr = map.get(userId) ?? [];
        arr.push(order);
        map.set(userId, arr);
      }
      return userIds.map(id => map.get(id) ?? []);
    }),
  };
}
```

---

## 📌 Step 2214: Mapped Types สำหรับ Resolvers

```typescript
// src/types/resolver-map.ts
// สร้าง resolver types จาก schema types อัตโนมัติ

import type { Resolvers } from './generated/graphql';

// ตรวจสอบว่า resolver ครบทุก field
type RequiredResolvers = Required<Resolvers>;

// Ensure ว่า resolver object type-safe
export function makeResolvers(resolvers: Resolvers): Resolvers {
  return resolvers;
}

// Type-safe context extraction
type ResolverFn<TResult, TParent, TContext, TArgs> = (
  parent: TParent,
  args: TArgs,
  context: TContext,
  info: GraphQLResolveInfo
) => Promise<TResult> | TResult;

// Helper: type-safe resolver ที่ต้องการ auth
type AuthenticatedContext = AppContext & { user: UserPayload };

function requireAuth<TResult, TParent, TArgs>(
  resolver: ResolverFn<TResult, TParent, AuthenticatedContext, TArgs>
): ResolverFn<TResult, TParent, AppContext, TArgs> {
  return (parent, args, ctx, info) => {
    if (!ctx.user) throw new AppError('Unauthenticated', ErrorCode.UNAUTHENTICATED);
    return resolver(parent, args, ctx as AuthenticatedContext, info);
  };
}

// Usage
const userResolvers: Resolvers = {
  Mutation: {
    updateProfile: requireAuth(async (_, { input }, ctx) => {
      // ctx.user รับประกันว่าไม่ใช่ null ที่นี่
      return ctx.prisma.user.update({
        where: { id: ctx.user.id },
        data: input,
      });
    }),
  },
};
```

---

## 📌 Step 2215: Conditional Types ใน GraphQL

```typescript
// src/types/conditional.ts

// Type-safe pagination
type PaginationArgs = {
  first?: number;
  after?: string;
  last?: number;
  before?: string;
};

type ConnectionType<T> = {
  edges: Array<{ node: T; cursor: string }>;
  pageInfo: {
    hasNextPage: boolean;
    hasPreviousPage: boolean;
    startCursor: string | null;
    endCursor: string | null;
  };
  totalCount: number;
};

// Generic connection builder
async function buildConnection<T extends { id: string }>(
  items: T[],
  args: PaginationArgs,
  totalCount: number
): Promise<ConnectionType<T>> {
  return {
    edges: items.map((node, i) => ({
      node,
      cursor: Buffer.from(`cursor:${i}`).toString('base64'),
    })),
    pageInfo: {
      hasNextPage: args.first ? items.length === args.first : false,
      hasPreviousPage: !!args.after,
      startCursor: items[0] ? Buffer.from(`cursor:0`).toString('base64') : null,
      endCursor: items.length > 0
        ? Buffer.from(`cursor:${items.length - 1}`).toString('base64')
        : null,
    },
    totalCount,
  };
}

// DeepPartial helper สำหรับ tests
type DeepPartial<T> = T extends object
  ? { [P in keyof T]?: DeepPartial<T[P]> }
  : T;

// Mock data ใน tests
function mockProduct(overrides: DeepPartial<Product> = {}): Product {
  return {
    id: asProductId('product-1'),
    name: 'Test Product',
    price: 1000,
    stock: 10,
    createdAt: new Date(),
    updatedAt: new Date(),
    ...overrides,
  } as Product;
}
```

---

## 📌 Step 2216: Strict Schema Types ด้วย GraphQL Codegen

```yaml
# codegen.yml
generates:
  src/generated/graphql.ts:
    schema: ./schema.graphql
    plugins:
      - typescript
      - typescript-resolvers
    config:
      # Strict null checks
      strictScalars: true
      
      # Branded IDs
      scalars:
        ID: string
        DateTime: Date
        
      # Custom mappers (DB model → GraphQL type)
      mappers:
        User: '@prisma/client#User as PrismaUser'
        Product: '@prisma/client#Product as PrismaProduct'
        
      # Context type
      contextType: '../context#AppContext'
      
      # Default resolver ที่ use deep partial
      defaultMapper: 'Partial<{T}>'
      
      # Type-safe enums
      enumsAsTypes: false
      
      # Avoid any
      avoidOptionals: true
```

```typescript
// src/generated/graphql.ts (example output)
export type UserResolvers<
  ContextType = AppContext,
  ParentType extends ResolversParentTypes['User'] = ResolversParentTypes['User']
> = {
  id?: Resolver<ResolversTypes['ID'], ParentType, ContextType>;
  email?: Resolver<ResolversTypes['String'], ParentType, ContextType>;
  orders?: Resolver<
    Array<ResolversTypes['Order']>,
    ParentType,
    ContextType,
    RequireFields<UserOrdersArgs, 'status'>
  >;
  __isTypeOf?: IsTypeOfResolverFn<ParentType, ContextType>;
};
```

---

## 📌 Step 2217: สรุป Part 083

### เนื้อหาที่เรียนรู้

✅ Branded types สำหรับ IDs  
✅ Discriminated unions กับ errors  
✅ Type-safe DataLoaders  
✅ Mapped types สำหรับ resolvers  
✅ requireAuth helper  
✅ DeepPartial สำหรับ tests  
✅ codegen config  

### TypeScript Best Practices

```
Type Safety:
□ Branded types สำหรับ IDs (UserId ≠ ProductId)
□ Result<T, E> แทน try/catch
□ Discriminated unions สำหรับ union types
□ Never use 'any'

Code Generation:
□ ใช้ codegen สำหรับ resolver types
□ mappers: DB types → GraphQL types
□ contextType ใน codegen
□ strictScalars: true

Testing:
□ DeepPartial<T> สำหรับ mock data
□ Type-safe mock context
□ TypeScript ตรวจ test coverage
```

### ในส่วนถัดไป

➡️ **[Part 084](./part-084.md)** — Custom GraphQL Directives

---

*Part 083 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
