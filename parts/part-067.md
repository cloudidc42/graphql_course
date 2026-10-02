# Part 067 — Apollo Federation: Subgraph Development 🕸️

> **ระดับ:** Expert | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1581–1620

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Apollo Federation v2 architecture
- Subgraph schema design
- @key, @external, @provides, @requires directives
- Entity resolution
- Rover CLI
- Shared types ระหว่าง subgraphs
- Federation best practices
- Error handling ใน federation

---

## 📌 Step 1581: Federation Architecture

```
ระบบ E-commerce — Federation Subgraphs:

Gateway (Router)
    ├── Users Subgraph     (auth, profiles)
    ├── Products Subgraph  (catalog, inventory)
    ├── Orders Subgraph    (orders, checkout)
    ├── Reviews Subgraph   (ratings, comments)
    └── Payments Subgraph  (transactions, refunds)

Client → Gateway → resolves สำหรับ all subgraphs → unified response

ข้อดี:
- แต่ละ team own schema ตัวเอง
- Deploy แยกกัน
- Scale แยกกัน
- Technology stack แยกกัน
```

---

## 📌 Step 1582: Users Subgraph

```bash
npm install @apollo/subgraph graphql
```

```typescript
// services/users/src/schema.ts
import { buildSubgraphSchema } from '@apollo/subgraph';
import { gql } from 'graphql-tag';

const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.0",
          import: ["@key", "@shareable", "@external", "@provides", "@requires"])
  
  # User เป็น entity ที่ subgraphs อื่น reference ได้
  type User @key(fields: "id") {
    id: ID!
    email: String!
    name: String!
    role: UserRole!
    createdAt: DateTime!
    profile: UserProfile
  }
  
  type UserProfile {
    bio: String
    avatar: String
    phone: String
    
    # address อาจดึงจาก Orders subgraph
    defaultShippingAddress: Address @external
  }
  
  enum UserRole {
    CUSTOMER
    SELLER
    ADMIN
  }
  
  # Address เป็น value type ที่ share ระหว่าง subgraphs
  type Address @shareable {
    street: String!
    city: String!
    province: String!
    postalCode: String!
    country: String!
  }
  
  type Query {
    me: User
    user(id: ID!): User
    users(filter: UserFilterInput, first: Int, after: String): UserConnection!
  }
  
  type Mutation {
    createUser(input: CreateUserInput!): CreateUserResult!
    updateUser(id: ID!, input: UpdateUserInput!): User!
    deleteUser(id: ID!): Boolean!
  }
  
  input UserFilterInput {
    role: UserRole
    search: String
  }
  
  input CreateUserInput {
    email: String!
    name: String!
    password: String!
    role: UserRole = CUSTOMER
  }
  
  input UpdateUserInput {
    name: String
    bio: String
    avatar: String
  }
  
  union CreateUserResult = User | EmailAlreadyExistsError
  
  type EmailAlreadyExistsError {
    message: String!
    email: String!
  }
  
  type UserConnection {
    edges: [UserEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }
  
  type UserEdge {
    node: User!
    cursor: String!
  }
  
  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }
  
  scalar DateTime
`;

const resolvers = {
  User: {
    // Entity resolver: ดึง User จาก ID
    __resolveReference: async ({ id }: { id: string }, ctx: AppContext) => {
      return ctx.prisma.user.findUnique({ where: { id } });
    },
  },
  
  Query: {
    me: async (_, __, ctx: AppContext) => {
      if (!ctx.user) return null;
      return ctx.prisma.user.findUnique({ where: { id: ctx.user.id } });
    },
    user: async (_, { id }, ctx) => {
      return ctx.prisma.user.findUnique({ where: { id } });
    },
  },
  
  Mutation: {
    createUser: async (_, { input }, ctx) => {
      const existing = await ctx.prisma.user.findUnique({
        where: { email: input.email },
      });
      
      if (existing) {
        return { __typename: 'EmailAlreadyExistsError', message: 'Email already exists', email: input.email };
      }
      
      const user = await ctx.prisma.user.create({
        data: {
          ...input,
          password: await hashPassword(input.password),
        },
      });
      
      return { ...user, __typename: 'User' };
    },
  },
  
  CreateUserResult: {
    __resolveType: (obj) => obj.__typename,
  },
};

export const schema = buildSubgraphSchema({ typeDefs, resolvers });
```

---

## 📌 Step 1583: Products Subgraph

```typescript
// services/products/src/schema.ts
const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.0",
          import: ["@key", "@shareable", "@external", "@provides", "@requires"])
  
  type Product @key(fields: "id") {
    id: ID!
    name: String!
    description: String
    price: Float!
    stock: Int!
    category: Category!
    seller: User!    # Reference User entity
    images: [ProductImage!]!
    createdAt: DateTime!
  }
  
  # stub ของ User จาก Users subgraph
  # Products subgraph ไม่ต้องรู้ User fields ทั้งหมด
  type User @key(fields: "id", resolvable: false) {
    id: ID!
  }
  
  type Category @key(fields: "id") @shareable {
    id: ID!
    name: String!
    slug: String!
    parentId: ID
  }
  
  type ProductImage {
    url: String!
    alt: String
    width: Int
    height: Int
  }
  
  type Query {
    product(id: ID!): Product
    products(filter: ProductFilterInput, first: Int, after: String): ProductConnection!
    categories: [Category!]!
    
    # Federation: Products subgraph provides featuredProducts
    # ซึ่ง Orders subgraph จะ require
    featuredProducts: [Product!]!
  }
  
  type Mutation {
    createProduct(input: CreateProductInput!): Product!
    updateProduct(id: ID!, input: UpdateProductInput!): Product!
    updateStock(id: ID!, delta: Int!): Product!
  }
  
  input ProductFilterInput {
    categoryId: ID
    minPrice: Float
    maxPrice: Float
    search: String
    inStock: Boolean
  }
  
  type ProductConnection {
    edges: [ProductEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }
  
  type ProductEdge {
    node: Product!
    cursor: String!
  }
  
  type PageInfo @shareable {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }
  
  scalar DateTime
`;

const resolvers = {
  Product: {
    __resolveReference: async ({ id }, ctx) => {
      return ctx.prisma.product.findUnique({ where: { id } });
    },
  },
  
  Category: {
    __resolveReference: async ({ id }, ctx) => {
      return ctx.prisma.category.findUnique({ where: { id } });
    },
  },
  
  Query: {
    product: (_, { id }, ctx) => ctx.prisma.product.findUnique({ where: { id } }),
    products: async (_, { filter, first, after }, ctx) => {
      // Relay pagination implementation
      return paginate(ctx.prisma.product, { filter, first, after });
    },
  },
};
```

---

## 📌 Step 1584: Orders Subgraph with @requires

```typescript
// services/orders/src/schema.ts
const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.0",
          import: ["@key", "@shareable", "@external", "@provides", "@requires"])
  
  type Order @key(fields: "id") {
    id: ID!
    customer: User!
    items: [OrderItem!]!
    status: OrderStatus!
    total: Float!
    shippingAddress: Address!
    createdAt: DateTime!
    
    # Computed field ที่ต้องการข้อมูลจาก User subgraph
    estimatedDelivery: DateTime @requires(fields: "customer { role }")
  }
  
  type OrderItem {
    product: Product!
    quantity: Int!
    price: Float!    # Price ณ เวลาที่ order
  }
  
  # Extends User entity จาก Users subgraph
  type User @key(fields: "id") {
    id: ID!
    role: UserRole! @external  # Comes from Users subgraph
    orders: [Order!]!          # เพิ่ม field ใน Users subgraph
    orderStats: OrderStats!
  }
  
  # Extends Product entity
  type Product @key(fields: "id") {
    id: ID!
    price: Float! @external    # ต้องรู้ price จาก Products subgraph
    # @requires price เพื่อคำนวณ orderRevenue
    orderRevenue: Float @requires(fields: "price")
  }
  
  type Address @shareable {
    street: String!
    city: String!
    province: String!
    postalCode: String!
    country: String!
  }
  
  enum OrderStatus {
    PENDING
    CONFIRMED
    PROCESSING
    SHIPPED
    DELIVERED
    CANCELLED
  }
  
  enum UserRole @external {
    CUSTOMER
    SELLER
    ADMIN
  }
  
  type OrderStats {
    totalOrders: Int!
    totalSpent: Float!
    avgOrderValue: Float!
  }
  
  type Query {
    order(id: ID!): Order
    orders(filter: OrderFilterInput, first: Int, after: String): OrderConnection!
    myOrders(first: Int, after: String): OrderConnection!
  }
  
  type Mutation {
    createOrder(input: CreateOrderInput!): CreateOrderResult!
    cancelOrder(id: ID!, reason: String!): Order!
    updateOrderStatus(id: ID!, status: OrderStatus!): Order!
  }
  
  input CreateOrderInput {
    items: [OrderItemInput!]!
    shippingAddressId: ID
    shippingAddress: AddressInput
    paymentMethodId: ID!
  }
  
  union CreateOrderResult = Order | OutOfStockError | PaymentError
  
  type OutOfStockError {
    message: String!
    productId: ID!
    availableStock: Int!
  }
  
  type PaymentError {
    message: String!
    code: String!
  }
  
  scalar DateTime
`;

const resolvers = {
  Order: {
    __resolveReference: async ({ id }, ctx) => {
      return ctx.prisma.order.findUnique({ where: { id } });
    },
    
    // @requires: customer.role ถูก inject มาแล้ว
    estimatedDelivery: (order: Order & { customer: { role: string } }) => {
      const days = order.customer.role === 'VIP' ? 1 : 3;
      const delivery = new Date();
      delivery.setDate(delivery.getDate() + days);
      return delivery;
    },
  },
  
  User: {
    // Extend User: add orders field
    orders: async (user: { id: string }, _, ctx) => {
      return ctx.prisma.order.findMany({
        where: { customerId: user.id },
        orderBy: { createdAt: 'desc' },
      });
    },
    orderStats: async (user: { id: string }, _, ctx) => {
      const orders = await ctx.prisma.order.findMany({
        where: { customerId: user.id, status: 'DELIVERED' },
        select: { total: true },
      });
      
      const totalSpent = orders.reduce((sum, o) => sum + o.total, 0);
      return {
        totalOrders: orders.length,
        totalSpent,
        avgOrderValue: orders.length ? totalSpent / orders.length : 0,
      };
    },
  },
  
  Product: {
    // @requires: product.price inject มาแล้ว
    orderRevenue: async (product: { id: string; price: number }, _, ctx) => {
      const result = await ctx.prisma.orderItem.aggregate({
        where: { productId: product.id },
        _sum: { price: true },
      });
      return result._sum.price ?? 0;
    },
  },
};
```

---

## 📌 Step 1585: Rover CLI

```bash
# Install Rover
curl -sSL https://rover.apollo.dev/nix/latest | sh

# Authenticate
rover config auth

# Validate subgraph schema
rover subgraph check my-graph@prod \
  --schema ./src/schema.graphql \
  --name users

# Publish subgraph to Apollo Registry
rover subgraph publish my-graph@prod \
  --schema ./src/schema.graphql \
  --name users \
  --routing-url https://users-api.myapp.com/graphql

# Compose supergraph locally (for testing)
rover supergraph compose --config ./supergraph.yaml

# supergraph.yaml:
# federation_version: =2.4.0
# subgraphs:
#   users:
#     routing_url: http://localhost:4001/graphql
#     schema:
#       file: ./services/users/schema.graphql
#   products:
#     routing_url: http://localhost:4002/graphql
#     schema:
#       file: ./services/products/schema.graphql
#   orders:
#     routing_url: http://localhost:4003/graphql
#     schema:
#       file: ./services/orders/schema.graphql
```

---

## 📌 Step 1586: สรุป Part 067

### เนื้อหาที่เรียนรู้

✅ Federation v2 architecture  
✅ @key entity definition  
✅ @external, @requires, @provides  
✅ Entity resolution (__resolveReference)  
✅ Cross-subgraph type extension  
✅ Rover CLI  
✅ Error types ใน federation  

### Federation Directive Reference

```
@key(fields: "id")         → Entity identifier
@key(fields: "id", resolvable: false) → Stub entity
@external                  → Field lives in another subgraph
@requires(fields: "...")   → Need field from another subgraph
@provides(fields: "...")   → Can resolve field without going to another subgraph
@shareable                 → Multiple subgraphs can define same type
@link(...)                 → Import federation directives
```

### ในส่วนถัดไป

➡️ **[Part 068](./part-068.md)** — Apollo Federation: Router & Supergraph

---

*Part 067 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
