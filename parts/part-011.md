# Part 011 — Directives: @deprecated, @auth, Custom Directives 🎛️

> **ระดับ:** Intermediate | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 291–330

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Built-in directives: @deprecated, @skip, @include, @specifiedBy
- Custom schema directives
- @auth directive
- @rateLimit directive
- @cacheControl directive
- @constraint directive
- Implementing directives with mapSchema
- Directive composition

---

## 📌 Step 291: Built-in Directives

```graphql
# @deprecated — mark fields/enum values เป็น deprecated
type User {
  id: ID!
  name: String!
  email: String!
  
  # @deprecated เตือน client ว่า field นี้จะหายไปในอนาคต
  username: String @deprecated(reason: "Use 'name' field instead")
  
  profilePicture: String @deprecated(reason: "Use 'avatar' field instead")
  avatar: String
}

enum UserStatus {
  ACTIVE
  INACTIVE
  SUSPENDED
  BANNED @deprecated(reason: "Use SUSPENDED instead")
}
```

### @skip และ @include (ใช้ใน operations)

```graphql
# @skip(if: Boolean!) — skip field ถ้า true
# @include(if: Boolean!) — include field ถ้า true
query GetUser($userId: ID!, $withDetails: Boolean!) {
  user(id: $userId) {
    id
    name
    
    # แสดงเฉพาะเมื่อ withDetails = true
    email @include(if: $withDetails)
    phone @include(if: $withDetails)
    
    # ซ่อนเมื่อ withDetails = false
    bio @skip(if: { not: $withDetails })
  }
}
```

---

## 📌 Step 292: Custom Schema Directives — @auth

```javascript
// src/directives/auth.js
import { mapSchema, getDirective, MapperKind } from '@graphql-tools/utils';
import { defaultFieldResolver } from 'graphql';

export function authDirectiveTransformer(schema) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const authDirective = getDirective(schema, fieldConfig, 'auth')?.[0];
      
      if (!authDirective) return fieldConfig;
      
      const { requires } = authDirective;
      const { resolve = defaultFieldResolver } = fieldConfig;
      
      return {
        ...fieldConfig,
        resolve: async function (source, args, context, info) {
          // ตรวจสอบ authentication
          if (!context.user) {
            throw new GraphQLError('You must be logged in', {
              extensions: { code: 'UNAUTHENTICATED' }
            });
          }
          
          // ตรวจสอบ role ถ้า requires ระบุ
          if (requires) {
            const userRole = context.user.role;
            const hasRole = Array.isArray(requires)
              ? requires.includes(userRole)
              : userRole === requires;
            
            if (!hasRole) {
              throw new GraphQLError(
                `You need ${requires} role to access this field`,
                { extensions: { code: 'FORBIDDEN' } }
              );
            }
          }
          
          return resolve(source, args, context, info);
        },
      };
    },
  });
}
```

### ใช้ @auth ใน Schema

```javascript
const typeDefs = `
  directive @auth(requires: String) on FIELD_DEFINITION
  directive @adminOnly on FIELD_DEFINITION
  
  type Query {
    # ทุกคนเข้าถึงได้
    publicPosts: [Post!]!
    
    # ต้อง login
    myProfile: User! @auth
    
    # ต้องเป็น ADMIN
    allUsers: [User!]! @auth(requires: "ADMIN")
    systemStats: Stats! @auth(requires: "ADMIN")
  }
  
  type Mutation {
    # ต้อง login
    createPost(input: CreatePostInput!): Post! @auth
    
    # ต้องเป็น ADMIN
    deleteUser(id: ID!): Boolean! @auth(requires: "ADMIN")
    updateSystemConfig(input: ConfigInput!): Config! @auth(requires: "ADMIN")
  }
  
  type User {
    id: ID!
    name: String!
    email: String! @auth              # ดูได้เฉพาะตัวเอง (หรือ admin)
    phone: String @auth(requires: "ADMIN")  # ดูได้เฉพาะ admin
    
    # Fields ที่ sensitive
    paymentMethods: [PaymentMethod!]! @auth
    orders: [Order!]! @auth
  }
`;
```

### Apply Directive Transformer

```javascript
import { makeExecutableSchema } from '@graphql-tools/schema';
import { authDirectiveTransformer } from './directives/auth.js';

let schema = makeExecutableSchema({ typeDefs, resolvers });
schema = authDirectiveTransformer(schema);

const server = new ApolloServer({ schema });
```

---

## 📌 Step 293: @rateLimit Directive

```javascript
// src/directives/rate-limit.js
import { mapSchema, getDirective, MapperKind } from '@graphql-tools/utils';
import { defaultFieldResolver } from 'graphql';

const rateLimitStore = new Map(); // ใช้ Redis ใน production

export function rateLimitDirectiveTransformer(schema) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const rateLimitDirective = getDirective(schema, fieldConfig, 'rateLimit')?.[0];
      
      if (!rateLimitDirective) return fieldConfig;
      
      const { max, window: windowSec } = rateLimitDirective;
      const { resolve = defaultFieldResolver } = fieldConfig;
      
      return {
        ...fieldConfig,
        resolve: async function (source, args, context, info) {
          // Key = user IP + field name
          const key = `rateLimit:${context.ip}:${info.fieldName}`;
          const now = Date.now();
          const windowMs = windowSec * 1000;
          
          // ดึง history
          const history = rateLimitStore.get(key) || [];
          
          // กรองเฉพาะ requests ในช่วง window
          const recent = history.filter(t => t > now - windowMs);
          
          if (recent.length >= max) {
            const resetIn = Math.ceil((recent[0] + windowMs - now) / 1000);
            throw new GraphQLError(
              `Rate limit exceeded. Try again in ${resetIn} seconds.`,
              { extensions: { code: 'RATE_LIMITED', resetIn } }
            );
          }
          
          // บันทึก request
          recent.push(now);
          rateLimitStore.set(key, recent);
          
          return resolve(source, args, context, info);
        },
      };
    },
  });
}
```

### ใช้ @rateLimit

```graphql
directive @rateLimit(
  max: Int!
  window: Int!   # seconds
) on FIELD_DEFINITION

type Mutation {
  login(email: String!, password: String!): AuthPayload!
    @rateLimit(max: 5, window: 60)      # 5 ครั้ง / นาที
  
  sendOTP(phone: String!): Boolean!
    @rateLimit(max: 3, window: 300)     # 3 ครั้ง / 5 นาที
  
  resetPassword(email: String!): Boolean!
    @rateLimit(max: 3, window: 3600)    # 3 ครั้ง / ชั่วโมง
}

type Query {
  search(term: String!): [SearchResult!]!
    @rateLimit(max: 100, window: 60)    # 100 ครั้ง / นาที
}
```

---

## 📌 Step 294: @cacheControl Directive

```javascript
// src/directives/cache-control.js
import { mapSchema, getDirective, MapperKind } from '@graphql-tools/utils';
import { defaultFieldResolver } from 'graphql';

export function cacheControlDirectiveTransformer(schema) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const cacheDirective = getDirective(schema, fieldConfig, 'cacheControl')?.[0];
      
      if (!cacheDirective) return fieldConfig;
      
      const { maxAge, scope } = cacheDirective;
      const { resolve = defaultFieldResolver } = fieldConfig;
      
      return {
        ...fieldConfig,
        resolve: async function (source, args, context, info) {
          const result = await resolve(source, args, context, info);
          
          // Set cache hints
          if (info.cacheControl) {
            info.cacheControl.setCacheHint({ maxAge, scope });
          }
          
          // Set response headers
          if (context.res && maxAge) {
            const cacheHeader = scope === 'PRIVATE'
              ? `private, max-age=${maxAge}`
              : `public, max-age=${maxAge}`;
            context.res.setHeader('Cache-Control', cacheHeader);
          }
          
          return result;
        },
      };
    },
  });
}
```

### ใช้ @cacheControl

```graphql
enum CacheControlScope {
  PUBLIC
  PRIVATE
}

directive @cacheControl(
  maxAge: Int
  scope: CacheControlScope
) on FIELD_DEFINITION | OBJECT

type Query {
  # Cache public data นาน
  categories: [Category!]! @cacheControl(maxAge: 3600, scope: PUBLIC)
  featuredProducts: [Product!]! @cacheControl(maxAge: 300, scope: PUBLIC)
  
  # ข้อมูลส่วนตัว — cache สั้นหน่อย, PRIVATE
  myProfile: User! @cacheControl(maxAge: 60, scope: PRIVATE) @auth
  
  # ไม่ cache ข้อมูล real-time
  liveStockPrice(symbol: String!): StockPrice!
}
```

---

## 📌 Step 295: @constraint Directive

```javascript
// src/directives/constraint.js
import { mapSchema, getDirective, MapperKind } from '@graphql-tools/utils';
import { defaultFieldResolver, GraphQLScalarType, Kind } from 'graphql';

function constraintTypeTransformer(type, constraints) {
  return new GraphQLScalarType({
    ...type.toConfig(),
    parseValue(value) {
      validate(value, constraints, type.name);
      return type.parseValue(value);
    },
    parseLiteral(ast) {
      const value = type.parseLiteral(ast);
      validate(value, constraints, type.name);
      return value;
    },
  });
}

function validate(value, constraints, typeName) {
  const { min, max, minLength, maxLength, pattern, format } = constraints;
  
  if (typeName === 'Int' || typeName === 'Float') {
    if (min !== undefined && value < min) {
      throw new GraphQLError(`Value ${value} is less than minimum ${min}`);
    }
    if (max !== undefined && value > max) {
      throw new GraphQLError(`Value ${value} exceeds maximum ${max}`);
    }
  }
  
  if (typeName === 'String') {
    if (minLength !== undefined && value.length < minLength) {
      throw new GraphQLError(`String is too short (min ${minLength} chars)`);
    }
    if (maxLength !== undefined && value.length > maxLength) {
      throw new GraphQLError(`String is too long (max ${maxLength} chars)`);
    }
    if (pattern && !new RegExp(pattern).test(value)) {
      throw new GraphQLError(`String does not match pattern ${pattern}`);
    }
  }
}

export function constraintDirectiveTransformer(schema) {
  return mapSchema(schema, {
    [MapperKind.ARGUMENT]: (argConfig) => {
      const constraintDirective = getDirective(schema, argConfig, 'constraint')?.[0];
      
      if (!constraintDirective) return argConfig;
      
      return {
        ...argConfig,
        type: constraintTypeTransformer(argConfig.type, constraintDirective),
      };
    },
  });
}
```

### ใช้ @constraint

```graphql
directive @constraint(
  min: Int
  max: Int
  minLength: Int
  maxLength: Int
  pattern: String
  format: String
) on INPUT_FIELD_DEFINITION | ARGUMENT_DEFINITION

type Mutation {
  createUser(input: CreateUserInput!): User!
}

input CreateUserInput {
  name: String! @constraint(minLength: 2, maxLength: 100)
  email: String! @constraint(format: "email")
  password: String! @constraint(minLength: 8, maxLength: 100)
  age: Int @constraint(min: 18, max: 120)
  phone: String @constraint(pattern: "^[0-9+\\-\\s]{10,15}$")
}
```

---

## 📌 Step 296: @deprecated ใน Practice

```graphql
type Product {
  id: ID!
  name: String!
  
  # Version 1 fields — deprecated
  imageUrl: String @deprecated(reason: "Use 'images' field instead")
  categoryId: ID @deprecated(reason: "Use 'category { id }' instead")
  
  # Version 2 fields
  images: [ProductImage!]!
  category: Category!
  
  # Price fields — migration
  cost: Float @deprecated(reason: "Use 'price' field which includes all fees")
  price: Float!
}

type Query {
  # Old API
  getUser(id: ID!): User @deprecated(reason: "Use 'user(id: ID!)' instead")
  
  # New API
  user(id: ID!): User
}
```

### ดู Deprecated Fields ใน GraphQL Playground

```graphql
# ดูว่า fields ไหน deprecated
query IntrospectDeprecated {
  __type(name: "Product") {
    fields(includeDeprecated: true) {
      name
      isDeprecated
      deprecationReason
    }
  }
}
```

---

## 📌 Step 297: Directive Composition

```javascript
// รวมหลาย directives ด้วย compose pattern
import { makeExecutableSchema } from '@graphql-tools/schema';
import { authDirectiveTransformer } from './directives/auth.js';
import { rateLimitDirectiveTransformer } from './directives/rate-limit.js';
import { cacheControlDirectiveTransformer } from './directives/cache-control.js';
import { constraintDirectiveTransformer } from './directives/constraint.js';

function buildSchema() {
  let schema = makeExecutableSchema({ typeDefs, resolvers });
  
  // Apply directives in order
  schema = authDirectiveTransformer(schema);
  schema = rateLimitDirectiveTransformer(schema);
  schema = cacheControlDirectiveTransformer(schema);
  schema = constraintDirectiveTransformer(schema);
  
  return schema;
}

const server = new ApolloServer({
  schema: buildSchema(),
});
```

### ใช้หลาย Directives ใน Field เดียว

```graphql
type Mutation {
  login(email: String!, password: String!): AuthPayload!
    @rateLimit(max: 5, window: 60)
    @constraint(minLength: 1)  # ← ถ้า @constraint support mutation args
}

type Query {
  sensitiveData: SensitiveInfo!
    @auth(requires: "ADMIN")
    @rateLimit(max: 10, window: 60)
    @cacheControl(maxAge: 30, scope: PRIVATE)
}
```

---

## 📌 Step 298: สรุป Part 011

### เนื้อหาที่เรียนรู้

✅ @deprecated สำหรับ schema evolution  
✅ @skip และ @include สำหรับ conditional fields  
✅ @auth directive + implementation  
✅ @rateLimit directive  
✅ @cacheControl directive  
✅ @constraint directive  
✅ Directive composition  

### Homework

1. implement @auth directive ที่รองรับ multiple roles (ADMIN | MODERATOR)
2. implement @rateLimit ด้วย Redis แทน in-memory Map
3. ทดสอบ @deprecated โดย deprecate field และดูคำเตือนใน GraphQL Playground

### ในส่วนถัดไป

➡️ **[Part 012](./part-012.md)** — Error Handling: Production-grade

---

*Part 011 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
