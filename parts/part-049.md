# Part 049 — Union Types & Interface Patterns 🔀

> **ระดับ:** Intermediate-Advanced | **เวลาเรียน:** 75 นาที | **ขั้นตอนที่:** 871–910

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Interface vs Union — เมื่อไรใช้อะไร
- Interface implementation ขั้นสูง
- Union discriminated types
- __resolveType และ __resolveAbstractType
- Abstract type caching
- Polymorphic queries
- Error union pattern
- Interface inheritance ผ่าน SDL

---

## 📌 Step 871: Interface vs Union

```graphql
# Interface: types มี fields ร่วมกัน
interface Node {
  id: ID!
}

interface Timestamped {
  createdAt: DateTime!
  updatedAt: DateTime!
}

# ใช้ multiple interfaces
type User implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  name: String!
  email: String!
}

type Product implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  name: String!
  price: Float!
}

# Union: types ไม่จำเป็นต้องมี fields ร่วมกัน
union SearchResult = User | Product | Post | Tag

# Query ด้วย inline fragments
query Search($term: String!) {
  search(term: $term) {
    ... on User { id name email }
    ... on Product { id name price }
    ... on Post { id title content }
    ... on Tag { id name color }
  }
}

# Interface query — ได้ fields ร่วมโดยตรง
query GetNode($id: ID!) {
  node(id: $id) {
    id  # shared field
    ... on User { name }
    ... on Product { name price }
  }
}
```

---

## 📌 Step 872: __resolveType Implementation

```typescript
// src/resolvers/abstract.ts

// Interface resolver
const NodeResolver = {
  __resolveType(obj: { __typename?: string; id: string }) {
    // Option 1: ใช้ __typename ที่ set ไว้
    if (obj.__typename) return obj.__typename;
    
    // Option 2: ดูจาก ID prefix
    if (obj.id.startsWith('user_')) return 'User';
    if (obj.id.startsWith('product_')) return 'Product';
    if (obj.id.startsWith('post_')) return 'Post';
    
    return null; // ไม่รู้จัก type
  }
};

// Union resolver
const SearchResultResolver = {
  __resolveType(obj: Record<string, unknown>) {
    // Option 3: duck typing — ดูจาก fields
    if ('email' in obj && 'role' in obj) return 'User';
    if ('price' in obj && 'inventory' in obj) return 'Product';
    if ('title' in obj && 'content' in obj) return 'Post';
    if ('color' in obj) return 'Tag';
    
    throw new Error(`Cannot resolve type for: ${JSON.stringify(obj)}`);
  }
};

// Register in resolvers
export const resolvers = {
  Node: NodeResolver,
  SearchResult: SearchResultResolver,
  // ... other resolvers
};
```

---

## 📌 Step 873: Error Union Pattern

```graphql
# Pattern ที่ดีกว่าการ throw error:
# ส่ง error เป็นส่วนหนึ่งของ response

# Union สำหรับ mutation result
union CreateUserResult = 
  CreateUserSuccess |
  EmailAlreadyExistsError |
  ValidationError

union LoginResult = 
  LoginSuccess |
  InvalidCredentialsError |
  AccountLockedError |
  MFARequiredError

type CreateUserSuccess {
  user: User!
}

type LoginSuccess {
  accessToken: String!
  refreshToken: String!
  user: User!
}

# Error types
interface Error {
  message: String!
  code: String!
}

type EmailAlreadyExistsError implements Error {
  message: String!
  code: String!
  email: String!
}

type ValidationError implements Error {
  message: String!
  code: String!
  field: String!
  value: String
}

type InvalidCredentialsError implements Error {
  message: String!
  code: String!
  remainingAttempts: Int
}

type AccountLockedError implements Error {
  message: String!
  code: String!
  lockedUntil: DateTime!
}

type MFARequiredError implements Error {
  message: String!
  code: String!
  mfaToken: String!  # token ใช้ต่อ MFA flow
}

type Mutation {
  createUser(input: CreateUserInput!): CreateUserResult!
  login(email: String!, password: String!): LoginResult!
}
```

---

## 📌 Step 874: Error Union Resolvers

```typescript
// src/resolvers/auth.resolver.ts

const authResolvers = {
  Mutation: {
    createUser: async (_, { input }, ctx) => {
      // ตรวจสอบ email ซ้ำ
      const existing = await ctx.prisma.user.findUnique({
        where: { email: input.email }
      });
      
      if (existing) {
        // Return error type แทน throw
        return {
          __typename: 'EmailAlreadyExistsError',
          message: `Email ${input.email} is already registered`,
          code: 'EMAIL_ALREADY_EXISTS',
          email: input.email,
        };
      }
      
      // Validation
      if (input.password.length < 8) {
        return {
          __typename: 'ValidationError',
          message: 'Password must be at least 8 characters',
          code: 'VALIDATION_ERROR',
          field: 'password',
          value: null,
        };
      }
      
      // สร้าง user
      const user = await ctx.prisma.user.create({
        data: {
          name: input.name,
          email: input.email,
          password: await bcrypt.hash(input.password, 12),
        }
      });
      
      return {
        __typename: 'CreateUserSuccess',
        user,
      };
    },
    
    login: async (_, { email, password }, ctx) => {
      const user = await ctx.prisma.user.findUnique({ where: { email } });
      
      if (!user || !(await bcrypt.compare(password, user.password))) {
        // Increment failed attempts
        await incrementFailedAttempts(email, ctx.redis);
        const remaining = await getRemainingAttempts(email, ctx.redis);
        
        return {
          __typename: 'InvalidCredentialsError',
          message: 'Invalid email or password',
          code: 'INVALID_CREDENTIALS',
          remainingAttempts: remaining,
        };
      }
      
      // ตรวจสอบ account locked
      if (user.lockedUntil && user.lockedUntil > new Date()) {
        return {
          __typename: 'AccountLockedError',
          message: 'Account is temporarily locked',
          code: 'ACCOUNT_LOCKED',
          lockedUntil: user.lockedUntil,
        };
      }
      
      // ตรวจสอบ MFA
      if (user.mfaEnabled) {
        const mfaToken = await createMfaSession(user.id, ctx.redis);
        return {
          __typename: 'MFARequiredError',
          message: 'MFA verification required',
          code: 'MFA_REQUIRED',
          mfaToken,
        };
      }
      
      // Success
      const { accessToken, refreshToken } = await createTokens(user);
      return {
        __typename: 'LoginSuccess',
        accessToken,
        refreshToken,
        user,
      };
    },
  },
  
  // __resolveType สำหรับ unions
  CreateUserResult: {
    __resolveType: (obj: { __typename: string }) => obj.__typename,
  },
  LoginResult: {
    __resolveType: (obj: { __typename: string }) => obj.__typename,
  },
  Error: {
    __resolveType: (obj: { __typename: string }) => obj.__typename,
  },
};

// Client usage:
// mutation Login($email: String!, $password: String!) {
//   login(email: $email, password: $password) {
//     ... on LoginSuccess {
//       accessToken
//       user { id name }
//     }
//     ... on InvalidCredentialsError {
//       message
//       remainingAttempts
//     }
//     ... on AccountLockedError {
//       message
//       lockedUntil
//     }
//     ... on MFARequiredError {
//       mfaToken
//     }
//   }
// }
```

---

## 📌 Step 875: Interface Inheritance

```graphql
# SDL ไม่รองรับ interface inheritance โดยตรง
# แต่ type สามารถ implement multiple interfaces

interface Identifiable {
  id: ID!
}

interface Auditable {
  createdAt: DateTime!
  updatedAt: DateTime!
  createdBy: User!
}

interface Publishable {
  publishedAt: DateTime
  isPublished: Boolean!
}

interface SoftDeletable {
  deletedAt: DateTime
  isDeleted: Boolean!
}

# Type implement หลาย interfaces
type Article implements Identifiable & Auditable & Publishable & SoftDeletable {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  createdBy: User!
  publishedAt: DateTime
  isPublished: Boolean!
  deletedAt: DateTime
  isDeleted: Boolean!
  title: String!
  content: String!
}

# Query ด้วย interface fields
query GetPublishableItems {
  # ถ้า node คือ Publishable ก็ได้ fields เหล่านี้
  articles {
    ... on Publishable {
      publishedAt
      isPublished
    }
    ... on Article {
      id
      title
    }
  }
}
```

---

## 📌 Step 876: Abstract Type Caching

```typescript
// การ cache __resolveType เพื่อประสิทธิภาพ

const typeCache = new WeakMap<object, string>();

const SearchResultResolver = {
  __resolveType(obj: Record<string, unknown>): string {
    // Check cache ก่อน
    if (typeCache.has(obj as object)) {
      return typeCache.get(obj as object)!;
    }
    
    let typename: string;
    
    // Resolve type
    if ('email' in obj) typename = 'User';
    else if ('price' in obj) typename = 'Product';
    else if ('title' in obj && 'authorId' in obj) typename = 'Post';
    else typename = 'Tag';
    
    // Cache result
    typeCache.set(obj as object, typename);
    return typename;
  }
};

// ดีกว่า: set __typename ที่ database layer
// เมื่อ query ข้อมูลหลาย types ให้ include __typename ใน query

async function searchAll(term: string) {
  const [users, products, posts] = await Promise.all([
    prisma.user.findMany({
      where: { name: { contains: term } },
      // เพิ่ม __typename
    }).then(items => items.map(u => ({ ...u, __typename: 'User' as const }))),
    
    prisma.product.findMany({
      where: { name: { contains: term } },
    }).then(items => items.map(p => ({ ...p, __typename: 'Product' as const }))),
    
    prisma.post.findMany({
      where: { title: { contains: term } },
    }).then(items => items.map(p => ({ ...p, __typename: 'Post' as const }))),
  ]);
  
  return [...users, ...products, ...posts];
}
```

---

## 📌 Step 877: สรุป Part 049

### เนื้อหาที่เรียนรู้

✅ Interface vs Union — use cases  
✅ __resolveType implementation strategies  
✅ Error Union pattern (vs throw)  
✅ Multiple interface implementation  
✅ Abstract type caching  
✅ Type-safe union handling  

### Error Union vs GraphQL Errors

```
GraphQL Errors (throw):
✅ เหมาะสำหรับ unexpected errors
✅ เหมาะสำหรับ permission denied
❌ Client ต้องจัดการใน errors array แยกต่างหาก
❌ ไม่มี type safety

Error Unions:
✅ Client ได้ type safety เต็มรูปแบบ
✅ Expected error flows ชัดเจน
✅ Schema-documented error cases
✅ Client บังคับ handle ทุก error case
❌ Schema verbose มากขึ้น
```

### ในส่วนถัดไป

➡️ **[Part 050](./part-050.md)** — Relay-Style Cursor Pagination

---

*Part 049 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
