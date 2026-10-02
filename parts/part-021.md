# Part 021 — Apollo Federation: Microservices Architecture 🏗️

> **ระดับ:** Advanced | **เวลาเรียน:** 150 นาที | **ขั้นตอนที่:** 691–740

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Apollo Federation คืออะไร
- Subgraph คืออะไร
- Gateway (Router)
- @key directive
- Entity reference resolvers
- @external, @requires, @provides
- Federated queries
- Federation v2 features

---

## 📌 Step 691: Federation Architecture

```
Traditional Monolith:
┌──────────────────────────────────┐
│      Single GraphQL Server       │
│  Users | Products | Orders | ... │
└──────────────────────────────────┘

Apollo Federation (Microservices):
                ┌────────────┐
                │   Router   │  ← Gateway ที่รับ request
                │ (Supergraph)│
                └─────┬──────┘
          ┌───────────┼───────────┐
          ↓           ↓           ↓
    ┌───────────┐ ┌──────────┐ ┌──────────┐
    │   Users   │ │ Products │ │  Orders  │
    │ Subgraph  │ │ Subgraph │ │ Subgraph │
    └───────────┘ └──────────┘ └──────────┘
```

---

## 📌 Step 692: ติดตั้ง Federation

```bash
# Subgraph packages
npm install @apollo/subgraph

# Router (Supergraph)
npm install @apollo/gateway
# หรือ Apollo Router (Rust-based, แนะนำสำหรับ production)
```

---

## 📌 Step 693: Users Subgraph

```javascript
// services/users/src/index.js
import { ApolloServer } from '@apollo/server';
import { buildSubgraphSchema } from '@apollo/subgraph';
import { gql } from 'graphql-tag';

const typeDefs = gql`
  # บอก Federation ว่านี่คือ subgraph
  extend schema @link(url: "https://specs.apollo.dev/federation/v2.3", import: ["@key", "@shareable"])
  
  type User @key(fields: "id") {
    id: ID!
    name: String!
    email: String!
    avatar: String
    role: UserRole!
    createdAt: String!
  }
  
  enum UserRole {
    ADMIN
    USER
  }
  
  type Query {
    me: User
    user(id: ID!): User
    users(limit: Int): [User!]!
  }
  
  type Mutation {
    login(email: String!, password: String!): AuthPayload!
    register(input: RegisterInput!): AuthPayload!
  }
  
  type AuthPayload {
    user: User!
    accessToken: String!
    refreshToken: String!
  }
  
  input RegisterInput {
    name: String!
    email: String!
    password: String!
  }
`;

const resolvers = {
  User: {
    // Entity resolver — ถูกเรียกเมื่อ subgraph อื่นต้องการ User
    __resolveReference: async ({ id }) => {
      return User.findById(id);
    },
  },
  
  Query: {
    me: async (_, __, { user }) => {
      if (!user) return null;
      return User.findById(user.sub);
    },
    user: (_, { id }) => User.findById(id),
    users: (_, { limit = 10 }) => User.findAll({ limit }),
  },
  
  Mutation: {
    login: async (_, { email, password }) => {
      // ... login logic
    },
  },
};

const server = new ApolloServer({
  schema: buildSubgraphSchema({ typeDefs, resolvers }),
});

await server.start();
app.use('/graphql', expressMiddleware(server, { context: createContext }));
app.listen(4001);
```

---

## 📌 Step 694: Products Subgraph

```javascript
// services/products/src/index.js
const typeDefs = gql`
  extend schema @link(url: "https://specs.apollo.dev/federation/v2.3", import: ["@key", "@external", "@requires"])
  
  type Product @key(fields: "id") {
    id: ID!
    name: String!
    price: Float!
    description: String
    sellerId: ID!
    
    # @external — field นี้ defined ใน Users subgraph
    seller: User! @external
  }
  
  # Extend User type จาก Users subgraph
  type User @key(fields: "id") {
    id: ID! @external
    
    # เพิ่ม field ใหม่ใน Users type
    products: [Product!]!
  }
  
  type Query {
    product(id: ID!): Product
    products(limit: Int): [Product!]!
  }
`;

const resolvers = {
  Product: {
    __resolveReference: async ({ id }) => Product.findById(id),
    
    seller: async (product) => {
      // Reference — Router จะ resolve ด้วย Users subgraph
      return { __typename: 'User', id: product.sellerId };
    },
  },
  
  User: {
    // Extend User with products
    products: async (user) => {
      return Product.findAll({ where: { sellerId: user.id } });
    },
  },
  
  Query: {
    product: (_, { id }) => Product.findById(id),
    products: (_, { limit = 10 }) => Product.findAll({ limit }),
  },
};

const server = new ApolloServer({
  schema: buildSubgraphSchema({ typeDefs, resolvers }),
});

app.listen(4002);
```

---

## 📌 Step 695: Orders Subgraph

```javascript
// services/orders/src/index.js
const typeDefs = gql`
  extend schema @link(url: "https://specs.apollo.dev/federation/v2.3", import: ["@key", "@external"])
  
  type Order @key(fields: "id") {
    id: ID!
    customerId: ID!
    status: OrderStatus!
    total: Float!
    createdAt: String!
  }
  
  enum OrderStatus {
    PENDING
    CONFIRMED
    SHIPPED
    DELIVERED
    CANCELLED
  }
  
  # Extend User with orders
  type User @key(fields: "id") {
    id: ID! @external
    orders(limit: Int): [Order!]!
    orderCount: Int!
  }
  
  type Query {
    order(id: ID!): Order
    orders: [Order!]!
  }
`;

const resolvers = {
  Order: {
    __resolveReference: async ({ id }) => Order.findById(id),
  },
  
  User: {
    orders: async (user, { limit = 10 }) => {
      return Order.findAll({
        where: { customerId: user.id },
        limit,
        order: [['createdAt', 'DESC']],
      });
    },
    
    orderCount: async (user) => {
      return Order.count({ where: { customerId: user.id } });
    },
  },
};

app.listen(4003);
```

---

## 📌 Step 696: Gateway Setup

```javascript
// gateway/src/index.js
import { ApolloServer } from '@apollo/server';
import { ApolloGateway, IntrospectAndCompose } from '@apollo/gateway';

const gateway = new ApolloGateway({
  supergraphSdl: new IntrospectAndCompose({
    subgraphs: [
      { name: 'users', url: 'http://localhost:4001/graphql' },
      { name: 'products', url: 'http://localhost:4002/graphql' },
      { name: 'orders', url: 'http://localhost:4003/graphql' },
    ],
  }),
});

const server = new ApolloServer({ gateway });

await server.start();
app.use('/graphql', expressMiddleware(server, {
  context: async ({ req }) => ({
    headers: req.headers,  // Pass auth headers to subgraphs
  }),
}));

app.listen(4000);
console.log('Gateway running at http://localhost:4000/graphql');
```

---

## 📌 Step 697: Federated Query ตัวอย่าง

```graphql
# Query ที่ Router จะ fan out ไปยัง subgraphs หลายตัว
query GetUserWithEverything($userId: ID!) {
  user(id: $userId) {
    # From Users subgraph
    id
    name
    email
    
    # From Products subgraph (extends User)
    products(limit: 5) {
      id
      name
      price
    }
    
    # From Orders subgraph (extends User)
    orders(limit: 5) {
      id
      status
      total
      createdAt
    }
    orderCount
  }
}
```

### Router Query Plan

```
Plan:
1. Fetch User (Users subgraph):
   → GET /users-graphql: query { user(id: $userId) { id name email } }
   
2. Parallel:
   a. Fetch products (Products subgraph):
      → query { _entities(representations: [{__typename: "User", id: "..."}]) { products } }
   
   b. Fetch orders (Orders subgraph):
      → query { _entities(representations: [{__typename: "User", id: "..."}]) { orders orderCount } }
   
3. Merge results and return to client
```

---

## 📌 Step 698: @requires Directive

```graphql
# Products subgraph
type Product @key(fields: "id") {
  id: ID!
  basePrice: Float!
  sellerId: ID!
  
  # requires seller.country สำหรับคำนวณ tax
  seller: Seller @external
  
  priceWithTax: Float! @requires(fields: "seller { country }")
}

type Seller @key(fields: "id") {
  id: ID! @external
  country: String! @external
}
```

```javascript
const resolvers = {
  Product: {
    priceWithTax: (product) => {
      // product.seller.country ถูก fetch มาให้แล้ว
      const taxRate = TAX_RATES[product.seller.country] || 0;
      return product.basePrice * (1 + taxRate);
    },
  },
};
```

---

## 📌 Step 699: สรุป Part 021

### เนื้อหาที่เรียนรู้

✅ Federation architecture  
✅ Subgraph ด้วย buildSubgraphSchema  
✅ @key directive สำหรับ entities  
✅ Entity reference resolvers  
✅ Extending types จาก subgraphs อื่น  
✅ Gateway setup ด้วย IntrospectAndCompose  
✅ Query planning  
✅ @requires directive  

### Homework

1. สร้าง 3 subgraphs: Users, Products, Reviews
2. Extend Product ด้วย reviews
3. ทดสอบ federated query ผ่าน Gateway

### ในส่วนถัดไป

➡️ **[Part 022](./part-022.md)** — Schema Stitching

---

*Part 021 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
