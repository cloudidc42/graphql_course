# Part 092 — Code-First Schema: Pothos & TypeGraphQL 🔧

> **ระดับ:** Expert | **เวลาเรียน:** 85 นาที | **ขั้นตอนที่:** 2616–2660

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Schema-first vs Code-first เปรียบเทียบ
- Pothos setup และ patterns
- TypeGraphQL approach
- Plugins: Prisma, Auth, Complexity
- Relay connections ด้วย Pothos
- Type inference กับ code-first
- Testing code-first schemas
- Migration จาก schema-first

---

## 📌 Step 2616: Schema-First vs Code-First

```
Schema-First (SDL-first):
  .graphql → codegen → TypeScript resolvers
  
  ✅ Schema เป็น source of truth
  ✅ ทีมที่ไม่ใช่ dev อ่านได้
  ✅ Language-agnostic documentation
  ❌ Type drift (schema ≠ resolver types)
  ❌ Codegen step ที่ต้องรัน

Code-First:
  TypeScript → auto-generate schema SDL
  
  ✅ Single source of truth (TypeScript)
  ✅ Compile-time safety
  ✅ IDE autocomplete บน schema
  ✅ Refactor รู้ผลทันที
  ❌ Schema SDL ยากอ่าน (generated)
  ❌ ต้องรู้ TypeScript ก่อน

Pothos (code-first) vs TypeGraphQL (code-first):
  Pothos: functional, plugin system, type-safe plugins
  TypeGraphQL: decorator-based, class-centric
```

---

## 📌 Step 2617: Pothos Setup

```bash
npm install @pothos/core @pothos/plugin-prisma @pothos/plugin-auth @pothos/plugin-relay @pothos/plugin-complexity
npm install -D prisma
```

```typescript
// src/schema/builder.ts
import SchemaBuilder from '@pothos/core';
import PrismaPlugin from '@pothos/plugin-prisma';
import AuthPlugin from '@pothos/plugin-auth';
import RelayPlugin from '@pothos/plugin-relay';
import ComplexityPlugin from '@pothos/plugin-complexity';
import { PrismaClient } from '@prisma/client';
import PrismaTypes from '@pothos/plugin-prisma/generated';

export type AuthScopes = {
  authenticated: boolean;
  admin: boolean;
  ownsResource: (resourceId: string) => boolean;
};

export const builder = new SchemaBuilder<{
  PrismaTypes: PrismaTypes;
  Context: AppContext;
  AuthScopes: AuthScopes;
  Scalars: {
    DateTime: { Input: Date; Output: Date };
    JSON: { Input: unknown; Output: unknown };
  };
}>({
  plugins: [PrismaPlugin, AuthPlugin, RelayPlugin, ComplexityPlugin],
  
  prisma: {
    client: (ctx) => ctx.prisma,
    filterConnectionTotalCount: true,
  },
  
  authScopes: async (ctx) => ({
    authenticated: !!ctx.user,
    admin: ctx.user?.role === 'ADMIN',
    ownsResource: (id: string) => ctx.user?.id === id,
  }),
  
  relayOptions: {
    clientMutationId: 'omit',
    cursorType: 'String',
  },
  
  complexity: {
    limit: { complexity: 5000, depth: 10 },
    defaultComplexity: 1,
    defaultListMultiplier: 10,
  },
});

// Add scalars
builder.addScalarType('DateTime', {
  serialize: (d) => (d as Date).toISOString(),
  parseValue: (v) => new Date(v as string),
});
```

---

## 📌 Step 2618: Pothos Types

```typescript
// src/schema/types/user.ts
import { builder } from '../builder';

// Prisma object type (auto maps ทุก fields)
export const UserType = builder.prismaObject('User', {
  authScopes: (user, ctx) => ({
    ownsResource: user.id,
    admin: true,  // admin เห็นทุก users
  }),
  
  fields: (t) => ({
    id: t.exposeID('id'),
    email: t.exposeString('email'),
    name: t.exposeString('name', { nullable: true }),
    role: t.exposeString('role'),
    createdAt: t.expose('createdAt', { type: 'DateTime' }),
    
    // Relation fields
    orders: t.relatedConnection('orders', {
      cursor: 'id',
      args: {
        status: t.arg.string({ required: false }),
      },
      query: (args) => ({
        where: args.status ? { status: args.status } : {},
        orderBy: { createdAt: 'desc' },
      }),
      complexity: { field: 5, multiplier: 'first' },
    }),
    
    // Computed field
    totalOrderAmount: t.float({
      nullable: true,
      resolve: async (user, _args, ctx) => {
        const agg = await ctx.prisma.order.aggregate({
          where: { userId: user.id },
          _sum: { totalAmount: true },
        });
        return agg._sum.totalAmount;
      },
    }),
  }),
});
```

---

## 📌 Step 2619: Pothos Queries & Mutations

```typescript
// src/schema/resolvers/user.resolver.ts
import { builder } from '../builder';
import { UserType } from '../types/user';

builder.queryField('me', (t) =>
  t.prismaField({
    type: UserType,
    nullable: true,
    authScopes: { authenticated: true },
    resolve: async (query, _root, _args, ctx) => {
      return ctx.prisma.user.findUnique({
        ...query,  // auto-include fields needed by selection
        where: { id: ctx.user!.id },
      });
    },
  })
);

builder.queryField('user', (t) =>
  t.prismaField({
    type: UserType,
    nullable: true,
    authScopes: { admin: true },
    args: {
      id: t.arg.id({ required: true }),
    },
    resolve: async (query, _root, args, ctx) => {
      return ctx.prisma.user.findUnique({
        ...query,
        where: { id: String(args.id) },
      });
    },
  })
);

// Input type
const CreateUserInput = builder.inputType('CreateUserInput', {
  fields: (t) => ({
    email: t.string({ required: true, validate: { email: true } }),
    name: t.string({ required: true, validate: { minLength: 1, maxLength: 100 } }),
    password: t.string({ required: true, validate: { minLength: 8 } }),
  }),
});

builder.mutationField('createUser', (t) =>
  t.prismaField({
    type: UserType,
    args: {
      input: t.arg({ type: CreateUserInput, required: true }),
    },
    resolve: async (query, _root, args, ctx) => {
      const { email, name, password } = args.input;
      const hashed = await bcrypt.hash(password, 12);
      
      return ctx.prisma.user.create({
        ...query,
        data: { email, name, password: hashed },
      });
    },
  })
);
```

---

## 📌 Step 2620: TypeGraphQL Approach

```typescript
// src/typegraphql/types/user.type.ts
import { ObjectType, Field, ID, Resolver, Query, Mutation, Arg, Ctx, Authorized } from 'type-graphql';
import { User as PrismaUser } from '@prisma/client';

@ObjectType()
export class User {
  @Field(() => ID)
  id!: string;
  
  @Field()
  email!: string;
  
  @Field({ nullable: true })
  name?: string;
  
  @Field()
  createdAt!: Date;
}

@InputType()
export class CreateUserInput {
  @Field()
  @IsEmail()
  email!: string;
  
  @Field()
  @MinLength(1)
  @MaxLength(100)
  name!: string;
  
  @Field()
  @MinLength(8)
  password!: string;
}

@Resolver(() => User)
export class UserResolver {
  @Query(() => User, { nullable: true })
  @Authorized()
  async me(@Ctx() ctx: AppContext): Promise<User | null> {
    return ctx.prisma.user.findUnique({ where: { id: ctx.user!.id } });
  }
  
  @Mutation(() => User)
  async createUser(
    @Arg('input') input: CreateUserInput,
    @Ctx() ctx: AppContext
  ): Promise<User> {
    const hashed = await bcrypt.hash(input.password, 12);
    return ctx.prisma.user.create({
      data: { email: input.email, name: input.name, password: hashed },
    });
  }
}

// Server setup
const schema = await buildSchema({
  resolvers: [UserResolver, ProductResolver, OrderResolver],
  authChecker: ({ context }, roles) => {
    const { user } = context as AppContext;
    if (!user) return false;
    if (roles.length === 0) return true;
    return roles.includes(user.role);
  },
});
```

---

## 📌 Step 2621: เมื่อไหร่เลือกอะไร

```
เลือก Schema-First (SDL) เมื่อ:
✅ Team มีทั้ง frontend/backend/designers
✅ Contract-first approach
✅ Multi-language (Python + Node.js subgraphs)
✅ Schema เป็น public API (documentation important)

เลือก Pothos เมื่อ:
✅ Pure TypeScript team
✅ Plugin ecosystem (Prisma, Auth, Relay)
✅ Type safety เป็น priority
✅ Complex permission patterns

เลือก TypeGraphQL เมื่อ:
✅ Class-based OOP style
✅ NestJS (มี @nestjs/graphql integration)
✅ Team ถนัด decorators
✅ Simpler setups

Recommendation:
- New project + TypeScript team → Pothos
- NestJS project → TypeGraphQL
- Cross-team/multi-lang → Schema-First
```

---

## 📌 Step 2622: สรุป Part 092

### เนื้อหาที่เรียนรู้

✅ Schema-first vs Code-first trade-offs  
✅ Pothos setup + plugins  
✅ Prisma integration ด้วย Pothos  
✅ Auth scopes  
✅ Relay connections  
✅ TypeGraphQL decorators  

### ในส่วนถัดไป

➡️ **[Part 093](./part-093.md)** — Open Source Contribution to GraphQL Ecosystem

---

*Part 092 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
