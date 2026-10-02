# Part 047 — Advanced TypeScript with GraphQL 🔷

> **ระดับ:** Advanced | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 801–840

---

## 🎯 สิ่งที่จะได้เรียนรู้

- TypeScript strict mode สำหรับ GraphQL projects
- Generic types และ utility types
- Mapped types สำหรับ resolvers
- Discriminated unions ใน TypeScript
- Type guards สำหรับ GraphQL results
- Conditional types สำหรับ schema mapping
- Template literal types
- Branded types สำหรับ IDs

---

## 📌 Step 801: TypeScript Strict Config

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022"],
    "strict": true,                    // เปิด strict mode ทั้งหมด
    "strictNullChecks": true,          // null/undefined ต้อง handle
    "noImplicitAny": true,             // ไม่ยอมรับ implicit any
    "noUncheckedIndexedAccess": true,  // array access อาจ undefined
    "exactOptionalPropertyTypes": true,// optional ≠ undefined
    "noImplicitReturns": true,         // ต้อง return ทุก path
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*", "test/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

## 📌 Step 802: Branded Types สำหรับ IDs

```typescript
// src/types/branded.ts
// ป้องกัน mixing IDs ต่างชนิด

declare const __brand: unique symbol;
type Brand<T, B> = T & { [__brand]: B };

// Branded ID types
export type UserId = Brand<string, 'UserId'>;
export type ProductId = Brand<string, 'ProductId'>;
export type OrderId = Brand<string, 'OrderId'>;
export type PostId = Brand<string, 'PostId'>;

// Constructor functions
export const UserId = (id: string): UserId => id as UserId;
export const ProductId = (id: string): ProductId => id as ProductId;
export const OrderId = (id: string): OrderId => id as OrderId;

// Usage
function getUser(id: UserId) { /* ... */ }
function getProduct(id: ProductId) { /* ... */ }

const userId = UserId('user-123');
const productId = ProductId('product-456');

getUser(userId);     // ✅
getUser(productId);  // ❌ Type error! ProductId ≠ UserId
getUser('raw-string'); // ❌ Type error!

// แปลง GraphQL ID → branded type
const userResolver = {
  Query: {
    user: (_, { id }: { id: string }, ctx) => {
      return ctx.prisma.user.findUnique({
        where: { id: UserId(id) } // Cast ที่ boundary เท่านั้น
      });
    }
  }
};
```

---

## 📌 Step 803: Discriminated Unions สำหรับ GraphQL Unions

```typescript
// src/types/search.ts

// GraphQL schema:
// union SearchResult = User | Product | Post

type UserResult = {
  __typename: 'User';
  id: UserId;
  name: string;
  email: string;
};

type ProductResult = {
  __typename: 'Product';
  id: ProductId;
  name: string;
  price: number;
};

type PostResult = {
  __typename: 'Post';
  id: PostId;
  title: string;
  content: string;
};

type SearchResult = UserResult | ProductResult | PostResult;

// TypeScript จะ narrow type อัตโนมัติ
function processSearchResult(result: SearchResult) {
  switch (result.__typename) {
    case 'User':
      // result: UserResult
      console.log(result.email);  // ✅
      break;
    case 'Product':
      // result: ProductResult  
      console.log(result.price);  // ✅
      break;
    case 'Post':
      // result: PostResult
      console.log(result.title);  // ✅
      break;
    default:
      // exhaustive check
      const _exhaustive: never = result;
  }
}

// Type guard functions
function isUser(result: SearchResult): result is UserResult {
  return result.__typename === 'User';
}

function isProduct(result: SearchResult): result is ProductResult {
  return result.__typename === 'Product';
}
```

---

## 📌 Step 804: Resolver Type Helpers

```typescript
// src/types/resolver-helpers.ts
import type { GraphQLResolveInfo } from 'graphql';
import type { AppContext } from './context.js';

// Generic resolver type
type Resolver<TResult, TParent = {}, TArgs = {}> = (
  parent: TParent,
  args: TArgs,
  context: AppContext,
  info: GraphQLResolveInfo
) => Promise<TResult> | TResult;

// Nullable resolver
type NullableResolver<TResult, TParent = {}, TArgs = {}> =
  Resolver<TResult | null | undefined, TParent, TArgs>;

// List resolver
type ListResolver<TResult, TParent = {}, TArgs = {}> =
  Resolver<TResult[], TParent, TArgs>;

// Subscription resolver type
type SubscriptionResolver<TResult, TArgs = {}> = {
  subscribe: (
    parent: {},
    args: TArgs,
    context: AppContext,
    info: GraphQLResolveInfo
  ) => AsyncIterable<{ [key: string]: TResult }>;
  resolve?: (
    payload: { [key: string]: TResult },
    args: TArgs,
    context: AppContext,
    info: GraphQLResolveInfo
  ) => TResult;
};

// Mapped resolver object type
type ResolverMap<T extends Record<string, unknown>> = {
  [K in keyof T]: Resolver<T[K]>;
};

// Usage
interface UserFields {
  id: UserId;
  name: string;
  email: string;
  posts: PostResult[];
}

const userFieldResolvers: Partial<ResolverMap<UserFields>> = {
  posts: (user, _, { loaders }) =>
    loaders.postsByUserId.load(user.id),
};
```

---

## 📌 Step 805: Generic Pagination Types

```typescript
// src/types/pagination.ts

// Connection pattern types
export interface Edge<T> {
  node: T;
  cursor: string;
}

export interface PageInfo {
  hasNextPage: boolean;
  hasPreviousPage: boolean;
  startCursor: string | null;
  endCursor: string | null;
}

export interface Connection<T> {
  edges: Edge<T>[];
  pageInfo: PageInfo;
  totalCount: number;
}

// Pagination args
export interface ConnectionArgs {
  first?: number | null;
  after?: string | null;
  last?: number | null;
  before?: string | null;
}

// Generic paginate function type
type PaginateFunction<T, TFilter = never> = (
  args: ConnectionArgs & ([TFilter] extends [never] ? {} : { filter?: TFilter | null }),
  context: AppContext
) => Promise<Connection<T>>;

// Helper to create connection from array
export function createConnection<T extends { id: string }>(
  items: T[],
  totalCount: number,
  args: ConnectionArgs
): Connection<T> {
  const edges: Edge<T>[] = items.map(item => ({
    node: item,
    cursor: Buffer.from(item.id).toString('base64'),
  }));

  return {
    edges,
    pageInfo: {
      hasNextPage: args.first ? items.length === args.first : false,
      hasPreviousPage: !!args.after,
      startCursor: edges[0]?.cursor ?? null,
      endCursor: edges[edges.length - 1]?.cursor ?? null,
    },
    totalCount,
  };
}

// Type-safe usage
const usersConnection: Connection<UserResult> = createConnection(
  users,
  100,
  { first: 10 }
);
```

---

## 📌 Step 806: Conditional Types สำหรับ Schema Mapping

```typescript
// src/types/schema-mapping.ts
// Map Prisma types → GraphQL types automatically

import type { User as PrismaUser, Post as PrismaPost } from '@prisma/client';

// Unwrap Promise
type Awaited<T> = T extends Promise<infer U> ? U : T;

// Make nullable fields optional
type GraphQLOptional<T> = {
  [K in keyof T]: null extends T[K] ? T[K] | undefined : T[K];
};

// Strip Prisma-specific fields
type StripPrivate<T, Keys extends keyof T> = Omit<T, Keys>;

// GraphQL-safe User (remove password)
type GraphQLUser = StripPrivate<PrismaUser, 'password' | 'refreshToken'>;

// Recursive deep partial (สำหรับ update operations)
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

// Input type from Prisma create args
type CreateInput<T> = {
  [K in keyof T as T[K] extends null ? never : K]: T[K] extends null | infer U ? U : T[K];
};

// Template literal types สำหรับ event names
type GraphQLEvent<T extends string> = `${Uppercase<T>}_CREATED` | `${Uppercase<T>}_UPDATED` | `${Uppercase<T>}_DELETED`;

type UserEvent = GraphQLEvent<'user'>;
// = "USER_CREATED" | "USER_UPDATED" | "USER_DELETED"

type ProductEvent = GraphQLEvent<'product'>;
// = "PRODUCT_CREATED" | "PRODUCT_UPDATED" | "PRODUCT_DELETED"

type AnyEvent = UserEvent | ProductEvent;

// PubSub with typed events
interface TypedPubSub {
  publish<T extends AnyEvent>(
    trigger: T,
    payload: EventPayload<T>
  ): Promise<void>;
  
  asyncIterator<T extends AnyEvent>(
    triggers: T | T[]
  ): AsyncIterableIterator<EventPayload<T>>;
}

type EventPayload<T extends AnyEvent> =
  T extends `${infer Entity}_${infer Action}`
    ? { entity: Lowercase<Entity>; action: Lowercase<Action>; data: unknown }
    : never;
```

---

## 📌 Step 807: Type-Safe Context

```typescript
// src/types/context.ts
import type { PrismaClient } from '@prisma/client';
import type { Redis } from 'ioredis';
import type DataLoader from 'dataloader';
import type { UserId, ProductId } from './branded.js';
import type { GraphQLUser } from './schema-mapping.js';

// DataLoader registry — type-safe loader access
interface DataLoaders {
  userById: DataLoader<UserId, GraphQLUser | null>;
  productById: DataLoader<ProductId, { id: ProductId; name: string; price: number } | null>;
  postsByUserId: DataLoader<UserId, Array<{ id: string; title: string }>>;
  orderCountByUserId: DataLoader<UserId, number>;
}

// Authenticated context (user is non-null)
export interface AuthenticatedContext {
  prisma: PrismaClient;
  redis: Redis;
  user: {
    sub: UserId;
    email: string;
    role: 'ADMIN' | 'USER' | 'SELLER';
  };
  loaders: DataLoaders;
  requestId: string;
}

// Public context (user may be null)
export interface AppContext extends Omit<AuthenticatedContext, 'user'> {
  user: AuthenticatedContext['user'] | null;
}

// Helper to assert authenticated
export function requireAuth(
  context: AppContext
): asserts context is AppContext & { user: NonNullable<AppContext['user']> } {
  if (!context.user) {
    throw new GraphQLError('Authentication required', {
      extensions: { code: 'UNAUTHENTICATED' },
    });
  }
}

// Usage in resolver
const meResolver = async (_: unknown, __: unknown, ctx: AppContext) => {
  requireAuth(ctx);
  // ctx.user is now non-null!
  return ctx.prisma.user.findUnique({ where: { id: ctx.user.sub } });
};
```

---

## 📌 Step 808: Utility Types สำหรับ Input Validation

```typescript
// src/types/validation.ts
import { z } from 'zod';

// Extract TypeScript type จาก Zod schema
type InferInput<T extends z.ZodType> = z.infer<T>;

// Common input schemas
const CreateUserInputSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
  password: z.string().min(8).max(72),
  role: z.enum(['USER', 'SELLER']).default('USER'),
});

const UpdateProfileInputSchema = z.object({
  name: z.string().min(1).max(100).optional(),
  bio: z.string().max(500).optional(),
  avatar: z.string().url().optional(),
}).refine(data => Object.keys(data).length > 0, {
  message: 'At least one field must be provided',
});

// Auto-derive TypeScript types
type CreateUserInput = InferInput<typeof CreateUserInputSchema>;
type UpdateProfileInput = InferInput<typeof UpdateProfileInputSchema>;

// Validated resolver wrapper
function withValidation<TArgs, TResult>(
  schema: z.ZodType<TArgs>,
  resolver: (args: TArgs, ctx: AppContext) => Promise<TResult>
) {
  return async (
    _parent: unknown,
    args: unknown,
    ctx: AppContext
  ): Promise<TResult> => {
    const validated = schema.parse(args);
    return resolver(validated, ctx);
  };
}

// Type-safe mutation
const createUser = withValidation(
  z.object({ input: CreateUserInputSchema }),
  async ({ input }, ctx) => {
    // input is fully typed as CreateUserInput
    return ctx.prisma.user.create({ data: input });
  }
);
```

---

## 📌 Step 809: TypeScript Project References

```json
// tsconfig.base.json (root)
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "declarationMap": true,
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}

// packages/server/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler"
  },
  "references": [
    { "path": "../shared" },
    { "path": "../types" }
  ]
}

// packages/client/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx"
  },
  "references": [
    { "path": "../shared" },
    { "path": "../types" }
  ]
}
```

```bash
# Build all packages in correct order (respects references)
tsc --build --verbose

# Type-check without emitting
tsc --build --noEmit

# Clean build
tsc --build --clean
```

---

## 📌 Step 810: สรุป Part 047

### เนื้อหาที่เรียนรู้

✅ TypeScript strict mode configuration  
✅ Branded types สำหรับ GraphQL IDs  
✅ Discriminated unions  
✅ Generic resolver types  
✅ Pagination connection types  
✅ Conditional/mapped types  
✅ Type-safe context  
✅ Zod-based input validation types  
✅ Project references  

### TypeScript Best Practices สำหรับ GraphQL

```
1. ใช้ branded types สำหรับทุก entity ID
2. อย่า cast as any — ใช้ type guards แทน
3. ใช้ discriminated unions สำหรับ GraphQL unions/interfaces
4. Generate types จาก schema ด้วย graphql-codegen
5. noUncheckedIndexedAccess: true — ปลอดภัยกว่า
6. requireAuth() function ที่ใช้ type narrowing
7. z.infer<> แทน manual interface definitions
```

### ในส่วนถัดไป

➡️ **[Part 048](./part-048.md)** — Custom Scalars Deep Dive

---

*Part 047 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
