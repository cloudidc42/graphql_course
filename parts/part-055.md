# Part 055 — GraphQL Mesh & Schema Unification 🕸️

> **ระดับ:** Advanced | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1111–1150

---

## 🎯 สิ่งที่จะได้เรียนรู้

- GraphQL Mesh คืออะไร
- รวม REST APIs เป็น GraphQL
- รวม multiple GraphQL endpoints
- OpenAPI/Swagger handler
- gRPC handler
- Database handler
- Schema transforms
- Stitching กับ federation comparison

---

## 📌 Step 1111: GraphQL Mesh Overview

```bash
npm install @graphql-mesh/cli @graphql-mesh/core
npm install @graphql-mesh/openapi     # REST/OpenAPI
npm install @graphql-mesh/graphql     # GraphQL remote
npm install @graphql-mesh/grpc        # gRPC
npm install @graphql-mesh/postgres    # PostgreSQL direct
npm install @graphql-mesh/transform-rename
npm install @graphql-mesh/transform-filter-schema
```

```yaml
# .meshrc.yml — configuration file

sources:
  # REST API via OpenAPI spec
  - name: Stripe
    handler:
      openapi:
        source: https://raw.githubusercontent.com/stripe/openapi/master/openapi/spec3.json
        baseUrl: https://api.stripe.com
        operationHeaders:
          Authorization: "Bearer {env.STRIPE_API_KEY}"
  
  # Another GraphQL API
  - name: ProductsAPI
    handler:
      graphql:
        endpoint: http://products-service:4001/graphql
        operationHeaders:
          Authorization: "{context.headers.authorization}"
  
  # Direct PostgreSQL
  - name: Analytics
    handler:
      postgraphile:
        connectionString: "{env.ANALYTICS_DATABASE_URL}"
        schemaName: public
        live: false

transforms:
  # Rename types เพื่อหลีกเลี่ยง conflicts
  - rename:
      renames:
        - from:
            type: Stripe_PaymentIntent
          to:
            type: PaymentIntent
  
  # Filter ออก fields ที่ไม่ต้องการ expose
  - filterSchema:
      filters:
        - "!Stripe.Mutation.{postAccountsAccountPeople*}"
        - "!Stripe.Mutation.{postEphemeralKeys*}"

additionalTypeDefs: |
  extend type Product {
    paymentIntents: [PaymentIntent!]!
  }

additionalResolvers:
  - ./src/additional-resolvers.ts

plugins:
  - maskedErrors: true
  - responseCache:
      ttl: 300  # 5 minutes
```

---

## 📌 Step 1112: REST to GraphQL Transform

```typescript
// additional-resolvers.ts
// เพิ่ม resolvers ที่ต้องการ cross-source data

import { Resolvers } from './.mesh';

export const resolvers: Resolvers = {
  Product: {
    // ดึง Stripe payment intents สำหรับ product นี้
    paymentIntents: async (product, _, { Stripe }) => {
      const response = await Stripe.query.getPaymentIntents({
        input: {
          // Stripe query args
          limit: 10,
        },
      });
      
      return response.data.filter(pi =>
        pi.metadata?.productId === product.id
      );
    },
  },
};
```

---

## 📌 Step 1113: gRPC to GraphQL

```protobuf
// user.proto
syntax = "proto3";

package users;

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (ListUsersResponse);
  rpc CreateUser (CreateUserRequest) returns (User);
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
}

message GetUserRequest {
  string id = 1;
}

message ListUsersRequest {
  int32 limit = 1;
  string cursor = 2;
}

message ListUsersResponse {
  repeated User users = 1;
  string next_cursor = 2;
  int32 total = 3;
}
```

```yaml
# .meshrc.yml
sources:
  - name: UsersGRPC
    handler:
      grpc:
        endpoint: grpc://user-service:50051
        protoFilePath: ./proto/user.proto
        
transforms:
  - rename:
      renames:
        - from: { type: users_User }
          to: { type: User }
        - from: { type: users_ListUsersResponse }
          to: { type: UserList }
```

---

## 📌 Step 1114: Schema Stitching vs Mesh vs Federation

```
GraphQL Schema Stitching:
- Manual: เขียน stitch code เอง
- Type merging ด้วย @merge directive
- เหมาะสำหรับ legacy systems

GraphQL Mesh:
- Config-driven: YAML config
- รองรับ REST, gRPC, database โดยตรง
- Transforms built-in
- เหมาะสำหรับ API aggregation layer

Apollo Federation:
- Schema-first: แต่ละ subgraph define ส่วนตัว
- @key directive สำหรับ entity resolution
- Best for microservices ใหม่ที่ design ให้ federated
- เหมาะสำหรับ greenfield microservices

เลือกใช้อะไร?
- Existing REST/gRPC services → GraphQL Mesh
- New microservices → Apollo Federation
- Simple merging → Schema Stitching
```

---

## 📌 Step 1115: Mesh Transforms

```yaml
# .meshrc.yml transforms examples

transforms:
  # 1. Rename — แก้ชื่อ conflicts
  - rename:
      renames:
        - from: { type: OldTypeName }
          to: { type: NewTypeName }
        - from: { type: User, field: user_id }
          to: { type: User, field: id }
  
  # 2. Filter — ซ่อน internal operations
  - filterSchema:
      filters:
        - "!Query.internalAdminQuery"
        - "!Mutation.dangerousDelete*"
        - "Type.{id, name, email}"  # whitelist fields
  
  # 3. Prefix — ป้องกัน type name conflicts
  - prefix:
      prefix: "Shopify_"
      includeRootOperations: true
      ignore:
        - ID
        - String
        - Boolean
  
  # 4. Naming convention
  - namingConvention:
      typeNames: pascalCase
      fieldNames: camelCase
      enumValues: upperCase
  
  # 5. Cache transform
  - cache:
      - field: Query.products
        ttl: 300        # 5 min
        cacheKey: "products-{args.filter}"
      - field: Query.user
        ttl: 60
        invalidate:
          mutation: updateUser  # invalidate เมื่อ update

  # 6. Mock — สำหรับ development
  - mock:
      mocks:
        - apply: User
          faker: true   # ใช้ faker.js
```

---

## 📌 Step 1116: Mesh กับ Authentication

```yaml
# .meshrc.yml — auth propagation

sources:
  - name: ProductsAPI
    handler:
      graphql:
        endpoint: http://products:4001/graphql
        # ส่ง auth header จาก request context
        operationHeaders:
          Authorization: "{context.headers.authorization}"
        # หรือ static API key
        # Authorization: "Bearer {env.PRODUCTS_API_KEY}"
  
  - name: OrdersREST
    handler:
      openapi:
        source: ./openapi/orders.yaml
        baseUrl: http://orders:3000
        operationHeaders:
          x-user-id: "{context.userId}"
          x-user-role: "{context.userRole}"

plugins:
  # JWT validation ก่อน forward
  - jwtAuth:
      secret: "{env.JWT_SECRET}"
      tokenType: Bearer
      extractJwt:
        header: Authorization
      reject:
        - unauthenticated
```

---

## 📌 Step 1117: สรุป Part 055

### เนื้อหาที่เรียนรู้

✅ GraphQL Mesh architecture  
✅ OpenAPI/REST handler  
✅ gRPC handler  
✅ Schema transforms (rename, filter, prefix)  
✅ Authentication propagation  
✅ Mesh vs Federation vs Stitching  

### Use Cases สำหรับ GraphQL Mesh

```
1. Legacy REST API modernization
   → Wrap REST endpoints เป็น GraphQL
   
2. Third-party API aggregation
   → Stripe + Shopify + SendGrid → unified GraphQL
   
3. BFF (Backend for Frontend)
   → Mobile BFF ที่รวม multiple services
   
4. API Gateway replacement
   → Kong/NGINX → GraphQL Mesh
   
5. Gradual migration
   → REST → GraphQL แบบ incremental
```

### ในส่วนถัดไป

➡️ **[Part 056](./part-056.md)** — Event-Driven GraphQL

---

*Part 055 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
