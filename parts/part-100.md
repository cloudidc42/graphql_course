# Part 100 — Capstone: Complete Production System 🏆

> **ระดับ:** World-Class | **เวลาเรียน:** 120 นาที | **ขั้นตอนที่:** 2976–3000+

---

## 🎯 โปรเจค Capstone: Thai E-Commerce Platform

สร้าง e-commerce platform ครบถ้วน ครอบคลุมทุกสิ่งที่เรียนมา:
- Federation architecture (4 subgraphs)
- Real-time order tracking
- AI-powered product search
- Mobile-ready API
- Production-grade security
- Full observability stack

---

## 📌 Step 2976: Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                     Clients                              │
│  Next.js Web App  │  React Native Mobile  │  Partners   │
└──────────┬────────┴──────────┬────────────┴──────────────┘
           │                  │
           ▼                  ▼
┌─────────────────────────────────────────────────────────┐
│              Cloudflare (CDN + WAF + DDoS)               │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│           Apollo Router (K8s: 3 replicas)                │
│  JWT auth │ Rate limiting │ APQ │ OTLP telemetry         │
└───┬───────┬────────┬──────────────┬─────────────────────┘
    │       │        │              │
    ▼       ▼        ▼              ▼
┌──────┐ ┌──────┐ ┌──────┐  ┌──────────────┐
│Users │ │Prods │ │Orders│  │Subscriptions │
│Subg. │ │Subg. │ │Subg. │  │  Subgraph    │
└──┬───┘ └──┬───┘ └──┬───┘  └──────┬───────┘
   │        │        │             │
   ▼        ▼        ▼             ▼
┌──────┐ ┌──────┐ ┌──────┐  ┌──────────┐
│ PG   │ │ PG + │ │ PG + │  │  Redis   │
│      │ │Elastic│ │Rabbit│  │  PubSub  │
└──────┘ └──────┘ └──────┘  └──────────┘
```

---

## 📌 Step 2977: Supergraph Schema

```graphql
# products-subgraph/schema.graphql
extend schema
  @link(url: "https://specs.apollo.dev/federation/v2.3",
        import: ["@key", "@shareable", "@external"])

type Product @key(fields: "id") {
  id: ID!
  name: String!
  slug: String!
  description: String
  price: Float!
  compareAtPrice: Float
  stock: Int!
  status: ProductStatus!
  images: [ProductImage!]!
  category: Category!
  tags: [String!]!
  
  # Localized fields
  localizedName(locale: Locale = TH): String!
  formattedPrice(currency: Currency = THB): String!
  
  # Reviews (from reviews subgraph via @requires)
  averageRating: Float
  reviewCount: Int
  
  createdAt: String!
  updatedAt: String!
}

type Category @key(fields: "id") {
  id: ID!
  name: String!
  slug: String!
  parentId: ID
  parent: Category
  children: [Category!]!
}

type Query {
  product(id: ID!): Product
  products(
    filter: ProductFilter
    sort: ProductSort
    first: Int = 20
    after: String
  ): ProductConnection! @cacheControl(maxAge: 60)
  
  # AI-powered search
  semanticSearch(query: String!, limit: Int = 10): [ProductSearchResult!]!
  
  # Featured/promotional
  featuredProducts: [Product!]! @cacheControl(maxAge: 300)
}

type Mutation {
  createProduct(input: CreateProductInput!): Product! @auth(requires: ADMIN)
  updateProduct(id: ID!, input: UpdateProductInput!): Product! @auth(requires: ADMIN)
  
  # File upload flow
  requestProductImageUpload(productId: ID!, filename: String!, contentType: String!): UploadRequest!
  confirmProductImageUpload(uploadId: ID!): Product!
}

# users-subgraph/schema.graphql
type User @key(fields: "id") {
  id: ID!
  email: String!
  name: String!
  phone: String
  role: UserRole!
  addresses: [Address!]!
  
  # From orders subgraph
  orders(status: OrderStatus, first: Int = 10): OrderConnection!
  
  # Stats
  totalOrders: Int!
  totalSpent: Float!
  
  createdAt: String!
}

type AuthPayload {
  accessToken: String!
  refreshToken: String!
  user: User!
}

type Mutation {
  register(input: RegisterInput!): AuthPayload!
  login(email: String!, password: String!): AuthPayload!
  loginWithGoogle(idToken: String!): AuthPayload!
  refreshToken(refreshToken: String!): AuthPayload!
  logout(refreshToken: String!): Boolean!
  
  updateProfile(input: UpdateProfileInput!): User! @auth(requires: USER)
  addAddress(input: AddressInput!): Address! @auth(requires: USER)
}

# orders-subgraph/schema.graphql
type Order @key(fields: "id") {
  id: ID!
  orderNumber: String!
  status: OrderStatus!
  
  # Customer from users subgraph
  customer: User! @external
  
  items: [OrderItem!]!
  shippingAddress: Address!
  totalAmount: Float!
  shippingFee: Float!
  discount: Float!
  
  # Tracking
  tracking: OrderTracking
  estimatedDelivery: String
  
  paymentMethod: PaymentMethod!
  paymentStatus: PaymentStatus!
  
  createdAt: String!
  updatedAt: String!
}

type Mutation {
  createOrder(input: CreateOrderInput!): CreateOrderResult! @auth(requires: USER)
  cancelOrder(orderId: ID!): Order! @auth(requires: USER)
  
  # Admin
  updateOrderStatus(orderId: ID!, status: OrderStatus!): Order! @auth(requires: ADMIN)
}

# subscriptions-subgraph/schema.graphql
type Subscription {
  orderStatusChanged(orderId: ID!): OrderStatusEvent! @auth(requires: USER)
  newMessage(conversationId: ID!): Message! @auth(requires: USER)
  productStockAlert(productId: ID!): StockAlertEvent! @auth(requires: USER)
}
```

---

## 📌 Step 2978: Users Subgraph Implementation

```typescript
// users-subgraph/src/resolvers/auth.ts
export const authResolvers = {
  Mutation: {
    register: async (_: unknown, { input }: RegisterArgs, { prisma }: AppContext) => {
      // Check duplicate email
      const existing = await prisma.user.findUnique({ where: { email: input.email } });
      if (existing) throw new AppError('Email already registered', ErrorCode.CONFLICT);
      
      const hashed = await bcrypt.hash(input.password, 12);
      const user = await prisma.user.create({
        data: {
          email: input.email,
          name: input.name,
          phone: input.phone,
          password: hashed,
          role: 'USER',
        },
      });
      
      // Send welcome email (async via queue)
      await emailQueue.add('welcome', { userId: user.id, email: user.email });
      
      return issueTokens(user);
    },
    
    loginWithGoogle: async (_: unknown, { idToken }: { idToken: string }, { prisma }: AppContext) => {
      const ticket = await googleClient.verifyIdToken({ idToken, audience: process.env.GOOGLE_CLIENT_ID });
      const payload = ticket.getPayload()!;
      
      const user = await prisma.user.upsert({
        where: { email: payload.email! },
        create: {
          email: payload.email!,
          name: payload.name ?? '',
          avatarUrl: payload.picture,
          provider: 'GOOGLE',
          providerId: payload.sub,
          emailVerified: true,
          password: '',  // ไม่ใช้ password
          role: 'USER',
        },
        update: { name: payload.name ?? undefined, avatarUrl: payload.picture },
      });
      
      return issueTokens(user);
    },
  },
  
  // Federation: resolve User entity จาก id
  User: {
    __resolveReference: async ({ id }: { id: string }, { prisma }: AppContext) => {
      return prisma.user.findUnique({ where: { id } });
    },
    
    addresses: (user, _args, { prisma }) =>
      prisma.address.findMany({ where: { userId: user.id } }),
    
    totalOrders: (user, _args, { loaders }) =>
      loaders.orderCountByUserId.load(user.id),
  },
};
```

---

## 📌 Step 2979: Order Processing กับ Saga

```typescript
// orders-subgraph/src/sagas/create-order.saga.ts

class CreateOrderSaga {
  private steps: SagaStep[] = [];
  private compensations: (() => Promise<void>)[] = [];
  
  async execute(input: CreateOrderInput, ctx: AppContext): Promise<Order> {
    try {
      // Step 1: Validate cart
      const cart = await this.validateCart(input.cartId, ctx);
      
      // Step 2: Reserve inventory
      await this.reserveInventory(cart.items, ctx);
      this.compensations.unshift(() => this.releaseInventory(cart.items, ctx));
      
      // Step 3: Create order record
      const order = await this.createOrderRecord(input, cart, ctx);
      this.compensations.unshift(() => this.cancelOrderRecord(order.id, ctx));
      
      // Step 4: Process payment
      await this.processPayment(order, input.paymentDetails, ctx);
      this.compensations.unshift(() => this.refundPayment(order.id, ctx));
      
      // Step 5: Send confirmation
      await ctx.emailQueue.add('order-confirmation', { orderId: order.id });
      
      // Step 6: Notify via subscription
      await ctx.pubsub.publish(`ORDER:${order.id}`, {
        orderStatusChanged: { orderId: order.id, status: 'CONFIRMED' },
      });
      
      return order;
      
    } catch (err) {
      // Compensate ย้อนกลับทุก step
      for (const compensate of this.compensations) {
        try {
          await compensate();
        } catch (cErr) {
          logger.error({ error: cErr }, 'Compensation failed');
        }
      }
      throw err;
    }
  }
  
  private async reserveInventory(items: CartItem[], ctx: AppContext): Promise<void> {
    for (const item of items) {
      const updated = await ctx.prisma.product.updateMany({
        where: { id: item.productId, stock: { gte: item.quantity } },
        data: { stock: { decrement: item.quantity } },
      });
      
      if (updated.count === 0) {
        throw new AppError(`สินค้า ${item.productId} มีไม่เพียงพอ`, ErrorCode.INSUFFICIENT_STOCK);
      }
    }
  }
}
```

---

## 📌 Step 2980: Deployment Configuration

```yaml
# k8s/products-subgraph.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: products-subgraph
  namespace: graphql-platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: products-subgraph
  template:
    spec:
      containers:
        - name: products-subgraph
          image: registry/products-subgraph:latest
          ports:
            - containerPort: 4001
          resources:
            requests: { cpu: "250m", memory: "256Mi" }
            limits:   { cpu: "1000m", memory: "512Mi" }
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: products-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: shared-secrets
                  key: redis-url
          livenessProbe:
            httpGet: { path: /health, port: 4001 }
            initialDelaySeconds: 10
          readinessProbe:
            httpGet: { path: /ready, port: 4001 }
            initialDelaySeconds: 5
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
      terminationGracePeriodSeconds: 30
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
---
# HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: products-subgraph-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: products-subgraph
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: graphql_active_requests
        target:
          type: AverageValue
          averageValue: "100"
```

---

## 📌 Step 2981: CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy GraphQL Platform

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: npm test -- --coverage
      - name: Schema contract tests
        run: npm run test:contracts

  schema-check:
    runs-on: ubuntu-latest
    steps:
      - name: Check breaking changes
        run: |
          npx rover subgraph check graphql-platform@current \
            --schema ./schema.graphql \
            --name products
        env:
          APOLLO_KEY: ${{ secrets.APOLLO_KEY }}

  build-and-push:
    needs: [test, schema-check]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [users, products, orders, subscriptions]
    steps:
      - name: Build Docker image
        run: |
          docker build -t registry/${{ matrix.service }}-subgraph:${{ github.sha }} \
            -f services/${{ matrix.service }}/Dockerfile .
      - name: Push to registry
        run: docker push registry/${{ matrix.service }}-subgraph:${{ github.sha }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/${{ matrix.service }}-subgraph \
            ${{ matrix.service }}-subgraph=registry/${{ matrix.service }}-subgraph:${{ github.sha }}
          kubectl rollout status deployment/${{ matrix.service }}-subgraph

      - name: Publish schema
        run: |
          npx rover subgraph publish graphql-platform@current \
            --schema ./schema.graphql \
            --name products \
            --routing-url https://products.internal/graphql
        env:
          APOLLO_KEY: ${{ secrets.APOLLO_KEY }}
```

---

## 📌 Step 2982: สรุป Part 100 & หลักสูตรทั้งหมด

### 🎉 ยินดีด้วย! คุณผ่านหลักสูตร GraphQL ฉบับสมบูรณ์แล้ว

```
ทักษะที่ได้รับตลอดหลักสูตร 100 Parts:

Fundamentals (Parts 001-040):
□ GraphQL syntax และ concepts
□ Queries, Mutations, Subscriptions
□ Schema design principles
□ Resolvers และ context

Intermediate (Parts 041-070):
□ Authentication & Authorization
□ DataLoader pattern
□ Testing (unit, integration, e2e)
□ Performance optimization
□ Production deployment

Expert (Parts 071-090):
□ Apollo Federation
□ Security (defensive)
□ Error handling patterns
□ Scale architecture
□ Mobile development
□ TypeScript integration

World-Class (Parts 091-100):
□ Message queues
□ AI integration
□ Real-time systems at scale
□ Enterprise governance
□ Open source contribution
□ Case studies (Netflix, GitHub, Shopify)
```

### หลังจากนี้

```
Short-term:
1. สร้าง project ใช้เองจริง
2. Contribute to 1 OSS project
3. เขียน blog post เกี่ยวกับ pattern ที่ชอบ

Medium-term:
4. Present ใน meetup/conference
5. สอน คนในทีม
6. Lead GraphQL adoption ในองค์กร

Long-term:
7. Participate ใน graphql-wg
8. Build tools ที่ community ต้องการ
9. เป็น GraphQL authority ในวงการไทย
```

### สิ่งที่สำคัญที่สุด

```
"GraphQL ไม่ใช่แค่ technology
 มันคือ philosophy ของ API design:

 → Declarative: clients define their needs
 → Type-safe: schema เป็น contract
 → Evolvable: deprecate gracefully
 → Observable: know who uses what
 → Team-scalable: Federation

 Build APIs ที่ทีมของคุณ และ users ของคุณรัก"
 
 — หลักสูตร GraphQL ฉบับสมบูรณ์
```

---

## 📊 หลักสูตรสรุป: 100 Parts ครอบคลุม

| Parts | หัวข้อ | Level |
|-------|--------|-------|
| 001-020 | GraphQL Basics | Beginner |
| 021-040 | Schema Design | Beginner |
| 041-060 | Auth, DataLoader, Testing | Intermediate |
| 061-070 | Performance, Deployment | Expert |
| 071-080 | Scale, Docs, Migration | Expert |
| 081-090 | Monitoring, Mobile, Edge | Expert |
| 091-095 | MQ, Code-First, OSS, Interviews | Expert |
| 096-100 | Case Studies, Capstone | World-Class |

**Total:** 100+ parts | 3,000+ steps | สามารถใช้งานได้จริง 100%

---

## 🙏 ขอบคุณที่เรียนหลักสูตรนี้

หลักสูตรนี้ครอบคลุม GraphQL จากพื้นฐานสู่ระดับโลก ด้วยความตั้งใจให้ผู้เรียนทุกคนสามารถนำไปใช้งานได้จริงในโปรเจคระดับ production

**GraphQL คือ อนาคตของ API development — และตอนนี้คุณพร้อมแล้ว!** 🚀

---

*Part 100 of 100 | หลักสูตร GraphQL ฉบับสมบูรณ์*  
*ครอบคลุมตั้งแต่ Step 1 ถึง Step 3000+*
