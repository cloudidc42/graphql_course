# Part 041 — Federation Advanced Patterns 🏗️

> **ระดับ:** Expert | **เวลาเรียน:** 120 นาที | **ขั้นตอนที่:** 681–720

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Federation v2 features
- @interfaceObject directive
- Progressive @override
- Context propagation
- Shared authentication
- Error handling across subgraphs
- Performance optimization
- Federation with subscriptions

---

## 📌 Step 681: Federation v2 Overview

```
Apollo Federation v2 improvements:

1. Shared types across subgraphs
2. @interfaceObject: extend interfaces
3. @override: gradual field migration
4. Better composition error messages
5. Built-in support for interfaces + unions
6. Shareable types (no single ownership)
```

---

## 📌 Step 682: @shareable Types

```graphql
# ใน Federation v2 — types ต้อง @shareable ถ้า define ใน > 1 subgraph

# subgraph-users/schema.graphql
extend schema @link(url: "https://specs.apollo.dev/federation/v2.3",
  import: ["@key", "@shareable"])

type User @key(fields: "id") {
  id: ID!
  name: String! @shareable  # ← field นี้ใช้ร่วมกันได้
  email: String!
  role: UserRole!
}

# subgraph-notifications/schema.graphql
# สามารถ define User.name ได้ด้วย เพราะ @shareable
type User @key(fields: "id") {
  id: ID!
  name: String! @shareable  # ← ต้องมี @shareable ทั้ง 2 subgraph
  notificationPreferences: NotificationPreferences!
}
```

---

## 📌 Step 683: @interfaceObject

```graphql
# ปัญหา: interface ที่ define ใน subgraph A
# subgraph B ต้องการเพิ่ม fields ให้ทุก type ที่ implement interface นั้น

# subgraph-products/schema.graphql
interface Sellable @key(fields: "id") {
  id: ID!
  price: Float!
}

type Product implements Sellable {
  id: ID!
  price: Float!
  name: String!
}

type Service implements Sellable {
  id: ID!
  price: Float!
  duration: Int!
}

# subgraph-analytics/schema.graphql
# ไม่รู้ types ที่ implement Sellable แต่ต้องการเพิ่ม analytics

extend schema @link(url: "https://specs.apollo.dev/federation/v2.3",
  import: ["@interfaceObject", "@key"])

# @interfaceObject ทำให้ treat Sellable เหมือน concrete type
type Sellable @key(fields: "id") @interfaceObject {
  id: ID!
  viewCount: Int!     # ← เพิ่ม field นี้ให้ทุก Sellable type
  revenue: Float!
}
```

---

## 📌 Step 684: @override สำหรับ Migration

```graphql
# Migration จาก subgraph A → B แบบ progressive

# subgraph-legacy/schema.graphql
type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Float!   # ← ต้องการ migrate field นี้ไป subgraph-catalog
}

# subgraph-catalog/schema.graphql
type Product @key(fields: "id") {
  id: ID!
  # ใช้ @override เพื่อ take over field จาก legacy
  price: Float! @override(from: "legacy")
  
  # เมื่อ migration เสร็จ: ลบ @override ออก และลบ field ใน legacy
  # Progressive rollout:
  # price: Float! @override(from: "legacy", label: "percent(50)")
  # → 50% traffic ไปที่ catalog, 50% ไปที่ legacy
}
```

---

## 📌 Step 685: Context Propagation

```javascript
// Header propagation ระหว่าง Gateway → Subgraphs

// gateway/index.js
import { ApolloGateway, IntrospectAndCompose, RemoteGraphQLDataSource } from '@apollo/gateway';

class AuthenticatedDataSource extends RemoteGraphQLDataSource {
  willSendRequest({ request, context }) {
    // ส่ง auth header ไปทุก subgraph
    request.http.headers.set(
      'Authorization',
      context.token ? `Bearer ${context.token}` : ''
    );
    
    // ส่ง user info
    if (context.user) {
      request.http.headers.set('x-user-id', context.user.sub);
      request.http.headers.set('x-user-role', context.user.role);
    }
    
    // Request ID สำหรับ tracing
    request.http.headers.set(
      'x-request-id',
      context.requestId || crypto.randomUUID()
    );
  }
}

const gateway = new ApolloGateway({
  supergraphSdl: new IntrospectAndCompose({
    subgraphs: [
      { name: 'users', url: process.env.USERS_SERVICE_URL },
      { name: 'products', url: process.env.PRODUCTS_SERVICE_URL },
      { name: 'orders', url: process.env.ORDERS_SERVICE_URL },
    ],
  }),
  buildService: ({ url }) => new AuthenticatedDataSource({ url }),
});

// Gateway server
const server = new ApolloServer({
  gateway,
  context: ({ req }) => ({
    token: req.headers.authorization?.split(' ')[1],
    user: verifyTokenSafe(req.headers.authorization),
    requestId: req.headers['x-request-id'] || crypto.randomUUID(),
  }),
});
```

---

## 📌 Step 686: Subgraph Authentication

```javascript
// subgraph: ตรวจสอบ user จาก gateway headers

export async function createContext({ req }) {
  const userId = req.headers['x-user-id'];
  const userRole = req.headers['x-user-role'];
  
  if (userId) {
    return {
      prisma,
      user: {
        sub: userId,
        role: userRole,
      },
    };
  }
  
  // Direct access ไปที่ subgraph (bypass gateway) → reject
  const authHeader = req.headers.authorization;
  if (authHeader) {
    try {
      const user = verifyToken(authHeader.split(' ')[1]);
      return { prisma, user };
    } catch {
      // ไม่ throw error — ให้ resolvers ตัดสินใจ
    }
  }
  
  return { prisma, user: null };
}
```

---

## 📌 Step 687: Federation Subscriptions

```javascript
// Federation subscriptions ต้องการ special setup

// เนื่องจาก Gateway ไม่ support subscriptions โดยตรง
// ให้ client connect ตรงไปยัง subgraph ที่มี subscription

// Option 1: Dedicated subscription service
// client → subscription-service (direct WebSocket)

// Option 2: Apollo Router (พร้อม subscription support)
// client → Router → subgraph

// subscription-service/schema.graphql
type Subscription {
  orderUpdated(customerId: ID!): Order!
  chatMessage(roomId: ID!): Message!
}

// subscription-service/resolvers.js
export const resolvers = {
  Subscription: {
    orderUpdated: {
      subscribe: withFilter(
        () => pubsub.asyncIterator('ORDER_UPDATED'),
        async (payload, { customerId }, { user }) => {
          // Security check
          if (!user || user.sub !== customerId) return false;
          return payload.orderUpdated.customerId === customerId;
        }
      ),
    },
  },
};
```

---

## 📌 Step 688: สรุป Part 041

### เนื้อหาที่เรียนรู้

✅ Federation v2 features  
✅ @shareable types  
✅ @interfaceObject  
✅ @override migration  
✅ Context propagation  
✅ Subgraph authentication  
✅ Federation subscriptions  

### Federation Best Practices

```
1. แต่ละ subgraph มี domain ชัดเจน
2. ใช้ @key เฉพาะ fields ที่เป็น stable identifier
3. Propagate user context ผ่าน headers
4. ตรวจสอบ subgraph security (ป้องกัน bypass gateway)
5. ใช้ @override สำหรับ gradual migration
6. Monitor composition errors ใน CI/CD
```

### ในส่วนถัดไป

➡️ **[Part 042](./part-042.md)** — GraphQL Code Generation & TypeScript

---

*Part 041 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
