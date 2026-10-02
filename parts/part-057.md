# Part 057 — CQRS with GraphQL 🔄

> **ระดับ:** Expert | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1191–1230

---

## 🎯 สิ่งที่จะได้เรียนรู้

- CQRS (Command Query Responsibility Segregation) patterns
- Read model vs Write model
- GraphQL Queries → Read side
- GraphQL Mutations → Command side
- Event sourcing basics
- Projection building
- Read replica routing
- Eventual consistency handling

---

## 📌 Step 1191: CQRS Pattern Overview

```
CQRS แยก Read (Query) กับ Write (Command) ออกจากกัน

Write side (Command):
Mutation → Command Handler → Write DB → Event

Read side (Query):
Query → Read Model → Optimized Read DB

ประโยชน์:
✅ Read model optimized สำหรับ query patterns
✅ Write model focused บน business rules
✅ Scale read/write independently
✅ Event-sourced write side (optional)

ใน GraphQL:
Query { ... }    → Read side (read replicas, caches)
Mutation { ... } → Write side (primary DB, commands)
```

---

## 📌 Step 1192: GraphQL Schema สำหรับ CQRS

```graphql
# Write side: Commands (semantic, business-focused)
type Mutation {
  # Commands — ชื่อสื่อถึง intent ชัดเจน
  placeOrder(input: PlaceOrderCommand!): PlaceOrderResult!
  cancelOrder(input: CancelOrderCommand!): CancelOrderResult!
  shipOrder(input: ShipOrderCommand!): ShipOrderResult!
  
  registerUser(input: RegisterUserCommand!): RegisterUserResult!
  changeEmail(input: ChangeEmailCommand!): ChangeEmailResult!
  
  addProductToInventory(input: AddInventoryCommand!): AddInventoryResult!
  reserveInventory(input: ReserveInventoryCommand!): ReserveInventoryResult!
}

input PlaceOrderCommand {
  customerId: ID!
  items: [OrderItemInput!]!
  shippingAddressId: ID!
  paymentMethodId: ID!
  promoCode: String
}

# Read side: Queries (view-optimized)
type Query {
  # Order views — denormalized สำหรับ UI
  orderSummary(id: ID!): OrderSummary
  orderHistory(customerId: ID!, first: Int, after: String): OrderHistoryConnection!
  ordersByStatus(status: OrderStatus!, first: Int): OrderStatusView!
  
  # Dashboard views — aggregated
  dashboardMetrics(period: DateRange!): DashboardMetrics!
  salesReport(filter: SalesFilter!): SalesReport!
  
  # Inventory view
  inventoryStatus(productId: ID!): InventoryStatus!
  lowStockProducts(threshold: Int!): [LowStockProduct!]!
}

# Read models — denormalized, view-specific
type OrderSummary {
  id: ID!
  orderNumber: String!
  status: OrderStatus!
  placedAt: DateTime!
  items: [OrderItemSummary!]!  # เก็บ snapshot ของ product data
  subtotal: Float!
  shipping: Float!
  total: Float!
  customer: CustomerSnapshot!  # snapshot ณ เวลาที่สั่ง
  timeline: [OrderEvent!]!
}

type CustomerSnapshot {
  id: ID!
  name: String!
  email: String!
  # ไม่ต้อง join ไปหา Customer table
}

type DashboardMetrics {
  totalRevenue: Float!
  totalOrders: Int!
  averageOrderValue: Float!
  newCustomers: Int!
  topProducts: [ProductMetric!]!
  revenueByDay: [DailyRevenue!]!
}
```

---

## 📌 Step 1193: Command Handlers

```typescript
// src/commands/order.commands.ts

interface Command {
  type: string;
  timestamp: Date;
  correlationId: string;
  causationId?: string;
  userId?: string;
}

interface PlaceOrderCommand extends Command {
  type: 'PLACE_ORDER';
  payload: {
    customerId: string;
    items: Array<{ productId: string; quantity: number; price: number }>;
    shippingAddressId: string;
    paymentMethodId: string;
    promoCode?: string;
  };
}

// Command Handler: business logic + validation
export class OrderCommandHandler {
  constructor(
    private orderRepo: OrderRepository,
    private productRepo: ProductRepository,
    private paymentService: PaymentService,
    private eventBus: EventBus
  ) {}
  
  async handle(command: PlaceOrderCommand): Promise<PlaceOrderResult> {
    // 1. Validate command
    await this.validatePlaceOrder(command);
    
    // 2. Reserve inventory
    for (const item of command.payload.items) {
      const reserved = await this.productRepo.reserveStock(
        item.productId,
        item.quantity
      );
      if (!reserved) {
        throw new GraphQLError(`Product ${item.productId} is out of stock`, {
          extensions: { code: 'OUT_OF_STOCK' }
        });
      }
    }
    
    // 3. Calculate pricing
    const pricing = await this.calculatePricing(command.payload);
    
    // 4. Process payment authorization
    const paymentAuth = await this.paymentService.authorize({
      methodId: command.payload.paymentMethodId,
      amount: pricing.total,
      currency: 'THB',
    });
    
    // 5. Create order (write model)
    const order = await this.orderRepo.create({
      customerId: command.payload.customerId,
      items: command.payload.items,
      pricing,
      paymentAuthorizationId: paymentAuth.id,
      shippingAddressId: command.payload.shippingAddressId,
    });
    
    // 6. Publish domain event
    await this.eventBus.publish({
      type: 'ORDER_PLACED',
      aggregateId: order.id,
      payload: {
        orderId: order.id,
        customerId: command.payload.customerId,
        items: command.payload.items,
        total: pricing.total,
      },
      correlationId: command.correlationId,
    });
    
    return { orderId: order.id, orderNumber: order.orderNumber };
  }
  
  private async validatePlaceOrder(command: PlaceOrderCommand) {
    if (!command.payload.items.length) {
      throw new GraphQLError('Order must have at least one item');
    }
    
    const customer = await this.orderRepo.findCustomer(command.payload.customerId);
    if (!customer) throw new GraphQLError('Customer not found');
    if (customer.isBlocked) throw new GraphQLError('Customer account is blocked');
  }
}

// Resolver using command handler
const placeOrderResolver = async (_, { input }, ctx) => {
  requireAuth(ctx);
  
  const command: PlaceOrderCommand = {
    type: 'PLACE_ORDER',
    timestamp: new Date(),
    correlationId: ctx.requestId,
    userId: ctx.user!.sub,
    payload: {
      customerId: ctx.user!.sub,
      ...input,
    },
  };
  
  const result = await ctx.commandBus.dispatch(command);
  return { success: true, ...result };
};
```

---

## 📌 Step 1194: Read Models & Projections

```typescript
// src/projections/order.projection.ts
// Projection: build read model จาก events

interface OrderProjection {
  id: string;
  orderNumber: string;
  status: string;
  customerId: string;
  customerName: string;
  customerEmail: string;
  items: ProjectedItem[];
  total: number;
  placedAt: Date;
  updatedAt: Date;
  timeline: TimelineEvent[];
}

class OrderProjectionBuilder {
  // Rebuild projection จาก event stream
  async rebuildFromEvents(orderId: string): Promise<OrderProjection | null> {
    const events = await this.eventStore.getEvents(orderId);
    
    let projection: Partial<OrderProjection> = {};
    
    for (const event of events) {
      projection = this.applyEvent(projection, event);
    }
    
    return projection as OrderProjection;
  }
  
  applyEvent(state: Partial<OrderProjection>, event: DomainEvent): Partial<OrderProjection> {
    switch (event.type) {
      case 'ORDER_PLACED':
        return {
          ...state,
          id: event.aggregateId,
          status: 'PENDING',
          customerId: event.payload.customerId,
          items: event.payload.items,
          total: event.payload.total,
          placedAt: event.timestamp,
          timeline: [{ event: 'Placed', timestamp: event.timestamp }],
        };
        
      case 'ORDER_PAYMENT_CONFIRMED':
        return {
          ...state,
          status: 'PAID',
          timeline: [...(state.timeline ?? []), {
            event: 'Payment confirmed',
            timestamp: event.timestamp,
          }],
        };
        
      case 'ORDER_SHIPPED':
        return {
          ...state,
          status: 'SHIPPED',
          timeline: [...(state.timeline ?? []), {
            event: `Shipped via ${event.payload.courier}`,
            timestamp: event.timestamp,
          }],
        };
        
      case 'ORDER_CANCELLED':
        return {
          ...state,
          status: 'CANCELLED',
          timeline: [...(state.timeline ?? []), {
            event: `Cancelled: ${event.payload.reason}`,
            timestamp: event.timestamp,
          }],
        };
        
      default:
        return state;
    }
  }
}

// Read resolver ใช้ read model (denormalized)
const orderSummaryResolver = async (_, { id }, ctx) => {
  requireAuth(ctx);
  
  // ดึงจาก read model (ไม่ join table!)
  const summary = await ctx.readDb.orderSummary.findUnique({
    where: { id }
  });
  
  if (!summary) return null;
  
  // Authorization: ลูกค้าเห็นเฉพาะ order ของตัวเอง
  if (ctx.user!.role !== 'ADMIN' && summary.customerId !== ctx.user!.sub) {
    throw new GraphQLError('Unauthorized', { extensions: { code: 'FORBIDDEN' } });
  }
  
  return summary;
};
```

---

## 📌 Step 1195: Read Replica Routing

```typescript
// src/db/router.ts
// Route reads ไปยัง replica, writes ไปยัง primary

import { PrismaClient } from '@prisma/client';

const primaryDb = new PrismaClient({
  datasources: { db: { url: process.env.DATABASE_URL } },
});

const replicaDb = new PrismaClient({
  datasources: { db: { url: process.env.DATABASE_REPLICA_URL } },
});

// Context: ส่ง readonly db ไปยัง resolver
export function createContext({ req }) {
  const isReadOperation = req.body?.query?.trim().startsWith('query');
  
  return {
    // Primary สำหรับ writes
    prisma: primaryDb,
    // Replica สำหรับ reads (eventual consistency)
    readDb: isReadOperation ? replicaDb : primaryDb,
    // หรือแยกชัดเจน
    writeDb: primaryDb,
    queryDb: replicaDb,
  };
}

// ใน resolver:
// Query (Read) → ctx.readDb
// Mutation (Write) → ctx.writeDb

// หมายเหตุ: eventual consistency
// หลัง mutation อาจต้อง read จาก primary ชั่วคราว
// เพื่อหลีกเลี่ยง read-your-writes inconsistency
```

---

## 📌 Step 1196: สรุป Part 057

### เนื้อหาที่เรียนรู้

✅ CQRS pattern overview  
✅ Commands vs Queries ใน GraphQL  
✅ Command handlers กับ business logic  
✅ Read models (projections)  
✅ Event sourcing basics  
✅ Read replica routing  

### เมื่อไรควรใช้ CQRS

```
ใช้ CQRS เมื่อ:
✅ Read/write workloads แตกต่างกันมาก
✅ Complex business domain ที่ต้องการ audit trail
✅ ต้องการ scale read กับ write แยกกัน
✅ Multiple read representations ของ data เดียวกัน

ไม่ต้องใช้ CQRS เมื่อ:
❌ Simple CRUD applications
❌ ทีมเล็ก/startup ระยะแรก
❌ Domain ไม่ซับซ้อน
❌ ยังไม่มี scaling problems
```

### ในส่วนถัดไป

➡️ **[Part 058](./part-058.md)** — Multi-Tenant GraphQL Architecture

---

*Part 057 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
