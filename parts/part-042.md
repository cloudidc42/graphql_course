# Part 042 — GraphQL Code Generation & TypeScript 🛠️

> **ระดับ:** Intermediate-Advanced | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 721–760

---

## 🎯 สิ่งที่จะได้เรียนรู้

- GraphQL Code Generator setup
- Generate types จาก schema
- Generate resolvers types
- Generate client hooks (React + Apollo)
- Generate MSW mocks
- Type-safe resolvers
- Fragment types
- Watch mode สำหรับ development

---

## 📌 Step 721: ทำไมต้องใช้ Code Generator

```
ปัญหาโดยไม่ใช้ Code Gen:

1. Types ใน schema ≠ TypeScript types
2. Resolver args ไม่มี type safety
3. Client queries อาจ break เมื่อ schema เปลี่ยน
4. Manual typing ซ้ำซ้อนและ error-prone

ประโยชน์ของ Code Gen:
✅ TypeScript types จาก schema อัตโนมัติ
✅ Type-safe resolver implementations
✅ Type-safe Apollo Client hooks
✅ Catch schema breaks ตอน compile time
✅ IntelliSense สำหรับ queries และ mutations
```

---

## 📌 Step 722: Setup Code Generator

```bash
npm install -D @graphql-codegen/cli
npm install -D @graphql-codegen/typescript
npm install -D @graphql-codegen/typescript-resolvers
npm install -D @graphql-codegen/typescript-operations
npm install -D @graphql-codegen/typescript-react-apollo
npm install -D @graphql-codegen/introspection
```

```yaml
# codegen.yml
schema: "./src/schema/**/*.graphql"

generates:
  # 1. TypeScript types จาก schema
  ./src/generated/types.ts:
    plugins:
      - typescript
    config:
      scalars:
        DateTime: string
        Email: string
        URL: string
        Decimal: number
        Upload: File
      enumsAsTypes: true
      futureProofEnums: true
  
  # 2. Resolver types (server-side)
  ./src/generated/resolvers.ts:
    plugins:
      - typescript
      - typescript-resolvers
    config:
      contextType: "../types#AppContext"
      defaultMapper: "Partial<{T}>"
      scalars:
        DateTime: string
        Email: string
      mappers:
        User: "../models/user#UserModel"
        Product: "../models/product#ProductModel"
  
  # 3. Operations types + hooks (client-side)
  ./src/client/generated/operations.ts:
    documents: "./src/client/**/*.graphql"
    plugins:
      - typescript
      - typescript-operations
      - typescript-react-apollo
    config:
      withHooks: true
      withComponent: false
      withHOC: false
      apolloReactHooksImportFrom: "@apollo/client"
```

---

## 📌 Step 723: Type-safe Resolvers

```typescript
// src/generated/resolvers.ts (auto-generated)
// ตัวอย่าง generated types

export type QueryResolvers<
  ContextType = AppContext,
  ParentType extends ResolversParentTypes['Query'] = ResolversParentTypes['Query']
> = {
  user?: Resolver<Maybe<ResolversTypes['User']>, ParentType, ContextType, RequireFields<QueryUserArgs, 'id'>>;
  users?: Resolver<Array<ResolversTypes['User']>, ParentType, ContextType, Partial<QueryUsersArgs>>;
  me?: Resolver<Maybe<ResolversTypes['User']>, ParentType, ContextType>;
};
```

```typescript
// src/resolvers/user.resolver.ts — ใช้ generated types

import { QueryResolvers, MutationResolvers, UserResolvers } from '../generated/resolvers.js';
import type { AppContext } from '../types.js';

// ✅ Type-safe resolvers
const userQueryResolvers: QueryResolvers<AppContext> = {
  user: async (_, { id }, { prisma }) => {
    // id: string (inferred from schema!)
    return prisma.user.findUnique({ where: { id } });
  },
  
  users: async (_, { limit = 10, offset = 0, filter }, { prisma }) => {
    // limit: number | null | undefined
    // filter: UsersFilterInput | null | undefined
    return prisma.user.findMany({
      take: limit ?? 10,
      skip: offset ?? 0,
    });
  },
  
  me: async (_, __, { user, prisma }) => {
    if (!user) return null;
    return prisma.user.findUnique({ where: { id: user.sub } });
  },
};

const userMutationResolvers: MutationResolvers<AppContext> = {
  createUser: async (_, { input }, { prisma }) => {
    // input: CreateUserInput (fully typed!)
    return prisma.user.create({ data: input });
  },
};

const userFieldResolvers: UserResolvers<AppContext> = {
  posts: (user, _, { loaders }) => {
    // user: UserModel (from mappers in codegen.yml)
    return loaders.postsByUserId.load(user.id);
  },
  
  fullName: (user) => `${user.firstName} ${user.lastName}`,
};

export const userResolvers = {
  Query: userQueryResolvers,
  Mutation: userMutationResolvers,
  User: userFieldResolvers,
};
```

---

## 📌 Step 724: Client-side Generated Hooks

```typescript
// src/client/queries/user.graphql
query GetUser($id: ID!) {
  user(id: $id) {
    id
    name
    email
    role
    posts {
      id
      title
      createdAt
    }
  }
}

mutation UpdateProfile($input: UpdateProfileInput!) {
  updateProfile(input: $input) {
    id
    name
    email
    updatedAt
  }
}
```

```tsx
// src/client/components/UserProfile.tsx
// ใช้ generated hooks

import {
  useGetUserQuery,
  useUpdateProfileMutation,
  GetUserQuery,
} from '../generated/operations.js';

interface UserProfileProps {
  userId: string;
}

export function UserProfile({ userId }: UserProfileProps) {
  // ✅ Fully typed!
  const { data, loading, error } = useGetUserQuery({
    variables: { id: userId },
  });
  
  const [updateProfile, { loading: updating }] = useUpdateProfileMutation();
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  
  const user = data?.user; // type: GetUserQuery['user']
  if (!user) return <div>User not found</div>;
  
  const handleUpdate = async () => {
    await updateProfile({
      variables: {
        input: {
          name: 'New Name', // type-safe!
          bio: 'Updated bio',
        },
      },
    });
  };
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <p>Role: {user.role}</p>
      
      <ul>
        {user.posts.map(post => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
      
      <button onClick={handleUpdate} disabled={updating}>
        Update Profile
      </button>
    </div>
  );
}
```

---

## 📌 Step 725: Fragment Types

```typescript
// src/client/fragments/user.graphql
fragment UserBasic on User {
  id
  name
  avatar
}

fragment UserFull on User {
  ...UserBasic
  email
  role
  createdAt
  posts { id title }
}

// Auto-generated types:
// UserBasicFragment = { id: string; name: string; avatar: string | null }
// UserFullFragment = UserBasicFragment & { email: string; role: UserRole; ... }

// ใช้ใน component
import { UserBasicFragment } from '../generated/operations.js';

interface UserCardProps {
  user: UserBasicFragment;
}

export function UserCard({ user }: UserCardProps) {
  return (
    <div>
      <img src={user.avatar ?? '/default-avatar.png'} alt={user.name} />
      <span>{user.name}</span>
    </div>
  );
}
```

---

## 📌 Step 726: Watch Mode & CI Integration

```json
// package.json
{
  "scripts": {
    "codegen": "graphql-codegen",
    "codegen:watch": "graphql-codegen --watch",
    "typecheck": "tsc --noEmit",
    "validate:schema": "graphql-codegen --check"
  }
}
```

```yaml
# .github/workflows/ci.yml
- name: Validate GraphQL Schema
  run: npm run validate:schema

- name: Check generated types are up-to-date
  run: |
    npm run codegen
    git diff --exit-code src/generated/
    # Fail ถ้า generated files ไม่ตรงกับ schema
```

---

## 📌 Step 727: สรุป Part 042

### เนื้อหาที่เรียนรู้

✅ Code Generator setup  
✅ Schema → TypeScript types  
✅ Type-safe resolvers  
✅ Generated Apollo Client hooks  
✅ Fragment types  
✅ CI validation  

### Workflow

```
1. แก้ schema .graphql
2. รัน npm run codegen
3. TypeScript compile ตรวจสอบทันที
4. แก้ resolvers/components ตาม types
5. ทุก CI run → validate codegen up-to-date
```

### ในส่วนถัดไป

➡️ **[Part 043](./part-043.md)** — GraphQL Testing Strategies

---

*Part 042 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
