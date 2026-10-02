# Part 056 — Event-Driven GraphQL 📡

> **ระดับ:** Advanced | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1151–1190

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Event-driven architecture patterns
- Apache Kafka + GraphQL Subscriptions
- NATS messaging
- Change Data Capture (CDC) กับ Debezium
- Event sourcing patterns
- Outbox pattern
- GraphQL subscriptions ด้วย Redis Streams
- Webhook → GraphQL pipeline

---

## 📌 Step 1151: Event-Driven Architecture Overview

```
Traditional (Request-Response):
Client → GraphQL Server → Database → Response

Event-Driven (Reactive):
Database → CDC → Event Bus → GraphQL Subscriptions → Client

ประโยชน์:
✅ Decoupling: services ไม่ต้องรู้จักกัน
✅ Scalability: event consumers scale independently
✅ Audit trail: ทุก event เป็น immutable record
✅ Real-time: client ได้ updates ทันที

Use cases:
- Order tracking: PLACED → PAID → SHIPPED → DELIVERED
- Chat applications
- Live dashboards (metrics, alerts)
- Collaborative editing
- IoT data streams
```

---

## 📌 Step 1152: Kafka + GraphQL Subscriptions

```bash
npm install kafkajs graphql-ws ws
```

```typescript
// src/events/kafka.ts
import { Kafka, Producer, Consumer } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'graphql-service',
  brokers: process.env.KAFKA_BROKERS?.split(',') ?? ['localhost:9092'],
  ssl: process.env.NODE_ENV === 'production',
  sasl: process.env.KAFKA_USERNAME ? {
    mechanism: 'plain',
    username: process.env.KAFKA_USERNAME,
    password: process.env.KAFKA_PASSWORD!,
  } : undefined,
});

let producer: Producer;

export async function getProducer() {
  if (!producer) {
    producer = kafka.producer({
      allowAutoTopicCreation: false,
      transactionalId: 'graphql-producer',
    });
    await producer.connect();
  }
  return producer;
}

// Publish event
export async function publishEvent<T>(
  topic: string,
  event: {
    type: string;
    entityId: string;
    data: T;
    metadata?: Record<string, string>;
  }
) {
  const prod = await getProducer();
  
  await prod.send({
    topic,
    messages: [
      {
        key: event.entityId,     // partition key
        value: JSON.stringify(event),
        headers: {
          'event-type': event.type,
          'content-type': 'application/json',
          'service': 'graphql-service',
          'timestamp': Date.now().toString(),
          ...event.metadata,
        },
      },
    ],
  });
}

// Topics
export const TOPICS = {
  ORDERS: 'orders.events',
  PRODUCTS: 'products.events',
  USERS: 'users.events',
  NOTIFICATIONS: 'notifications',
} as const;
```

---

## 📌 Step 1153: Consumer → PubSub Bridge

```typescript
// src/events/consumer.ts
// Kafka → Redis PubSub bridge

import { Consumer } from 'kafkajs';
import { RedisPubSub } from 'graphql-redis-subscriptions';
import { kafka } from './kafka.js';

export class EventConsumer {
  private consumer: Consumer;
  
  constructor(private pubsub: RedisPubSub) {
    this.consumer = kafka.consumer({
      groupId: 'graphql-subscriptions',
      heartbeatInterval: 3000,
      sessionTimeout: 30000,
    });
  }
  
  async start() {
    await this.consumer.connect();
    
    await this.consumer.subscribe({
      topics: [
        'orders.events',
        'products.events',
        'notifications',
      ],
      fromBeginning: false,
    });
    
    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        const eventType = message.headers?.['event-type']?.toString();
        const payload = JSON.parse(message.value?.toString() ?? '{}');
        
        // Publish ไปยัง PubSub
        const trigger = `${topic}:${eventType}`;
        await this.pubsub.publish(trigger, payload);
        
        // Log สำหรับ debugging
        console.log(`Event: ${trigger}`, { partition, offset: message.offset });
      },
      
      // Error handling
      eachBatchAutoResolve: true,
    });
  }
  
  async stop() {
    await this.consumer.disconnect();
  }
}

// Initialize
const pubsub = new RedisPubSub({
  connection: { host: process.env.REDIS_HOST, port: 6379 },
});

const consumer = new EventConsumer(pubsub);
await consumer.start();
```

---

## 📌 Step 1154: GraphQL Subscription Resolvers

```typescript
// src/resolvers/subscription.resolver.ts
import { withFilter } from 'graphql-subscriptions';

const subscriptionResolvers = {
  Subscription: {
    // Order tracking subscription
    orderStatusUpdated: {
      subscribe: withFilter(
        () => pubsub.asyncIterator(['orders.events:ORDER_UPDATED']),
        (payload, args, context) => {
          // กรอง: ส่งเฉพาะ order ที่ user เป็นเจ้าของ
          if (!context.user) return false;
          
          const order = payload.data;
          return order.customerId === context.user.sub &&
            (!args.orderId || args.orderId === order.id);
        }
      ),
      resolve: (payload) => payload.data,
    },
    
    // Live dashboard metrics
    metricsUpdated: {
      subscribe: withFilter(
        () => pubsub.asyncIterator(['metrics:UPDATED']),
        (payload, args, context) => {
          // ต้องเป็น admin
          return context.user?.role === 'ADMIN';
        }
      ),
      resolve: (payload) => payload.data,
    },
    
    // Product inventory alerts
    inventoryAlert: {
      subscribe: withFilter(
        () => pubsub.asyncIterator(['products.events:INVENTORY_LOW']),
        (payload, args, context) => {
          if (!context.user) return false;
          // Sellers เห็นเฉพาะ products ของตัวเอง
          if (context.user.role === 'SELLER') {
            return payload.data.sellerId === context.user.sub;
          }
          return context.user.role === 'ADMIN';
        }
      ),
      resolve: (payload) => payload.data,
    },
    
    // Chat room messages
    newMessage: {
      subscribe: withFilter(
        () => pubsub.asyncIterator(['chat:MESSAGE']),
        (payload, { roomId }, context) => {
          if (!context.user) return false;
          return payload.roomId === roomId;
        }
      ),
      resolve: (payload) => payload.message,
    },
  },
};
```

---

## 📌 Step 1155: Outbox Pattern

```typescript
// Outbox Pattern: ป้องกัน dual-write problem
// เขียน DB + publish event ใน same transaction

// schema.prisma
// model OutboxEvent {
//   id        String   @id @default(uuid())
//   type      String
//   payload   Json
//   topic     String
//   published Boolean  @default(false)
//   createdAt DateTime @default(now())
//   
//   @@index([published, createdAt])
// }

async function createOrderWithOutbox(
  input: CreateOrderInput,
  ctx: AppContext
) {
  // Single transaction: create order + write outbox event
  return ctx.prisma.$transaction(async (tx) => {
    const order = await tx.order.create({
      data: {
        customerId: ctx.user!.sub,
        items: { create: input.items },
        status: 'PENDING',
        total: calculateTotal(input.items),
      },
    });
    
    // Write to outbox (same transaction)
    await tx.outboxEvent.create({
      data: {
        type: 'ORDER_CREATED',
        topic: 'orders.events',
        payload: {
          orderId: order.id,
          customerId: order.customerId,
          total: order.total,
          items: input.items,
        },
      },
    });
    
    return order;
  });
}

// Background worker: publish outbox events
async function processOutbox() {
  while (true) {
    const events = await prisma.outboxEvent.findMany({
      where: { published: false },
      orderBy: { createdAt: 'asc' },
      take: 100,
    });
    
    for (const event of events) {
      try {
        await publishEvent(event.topic, {
          type: event.type,
          entityId: (event.payload as { orderId: string }).orderId,
          data: event.payload,
        });
        
        await prisma.outboxEvent.update({
          where: { id: event.id },
          data: { published: true },
        });
      } catch (error) {
        console.error('Failed to publish outbox event:', error);
        // Retry later
      }
    }
    
    await sleep(1000); // poll ทุก 1 วินาที
  }
}
```

---

## 📌 Step 1156: Change Data Capture กับ Debezium

```yaml
# docker-compose.yml — Debezium CDC setup
services:
  debezium:
    image: debezium/connect:2.4
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: debezium-group
      CONFIG_STORAGE_TOPIC: debezium-config
      OFFSET_STORAGE_TOPIC: debezium-offsets
      STATUS_STORAGE_TOPIC: debezium-status
    depends_on: [kafka, postgres]
```

```bash
# สร้าง PostgreSQL CDC connector
curl -X POST http://localhost:8083/connectors -H 'Content-Type: application/json' -d '{
  "name": "postgres-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "secret",
    "database.dbname": "myapp",
    "database.server.name": "myapp",
    "table.include.list": "public.orders,public.products",
    "plugin.name": "pgoutput"
  }
}'
# → ทุก INSERT/UPDATE/DELETE ใน orders และ products จะไปที่ Kafka
# Topic: myapp.public.orders, myapp.public.products
```

```typescript
// src/events/cdc-consumer.ts
// Process CDC events จาก Debezium

interface DebeziumPayload {
  op: 'c' | 'u' | 'd' | 'r'; // create, update, delete, read
  before: Record<string, unknown> | null;
  after: Record<string, unknown> | null;
  source: { table: string; ts_ms: number };
}

async function processCDCEvent(payload: DebeziumPayload) {
  const { op, before, after, source } = payload;
  
  if (source.table === 'orders') {
    if (op === 'u' && after) {
      const order = after;
      const prevStatus = before?.status;
      const newStatus = after.status;
      
      if (prevStatus !== newStatus) {
        // Publish status change event
        await publishEvent('orders.events', {
          type: 'ORDER_STATUS_CHANGED',
          entityId: order.id as string,
          data: { orderId: order.id, prevStatus, newStatus },
        });
        
        // GraphQL PubSub
        await pubsub.publish('orders.events:ORDER_UPDATED', {
          data: order,
        });
      }
    }
  }
}
```

---

## 📌 Step 1157: Redis Streams สำหรับ Subscriptions

```typescript
// Redis Streams: ordered, persistent event log
// ดีกว่า Redis Pub/Sub สำหรับ event streaming

import { Redis } from 'ioredis';

const redis = new Redis();

// Publish ไปยัง Redis Stream
async function publishToStream(streamKey: string, event: Record<string, string>) {
  // XADD: append to stream, auto-generate ID
  const eventId = await redis.xadd(streamKey, '*', ...Object.entries(event).flat());
  return eventId;
}

// Consumer group: scale consumers horizontally
async function createConsumerGroup(stream: string, group: string) {
  try {
    await redis.xgroup('CREATE', stream, group, '0', 'MKSTREAM');
  } catch (e) {
    // Group already exists
  }
}

// Read from stream
async function readFromStream(
  stream: string,
  group: string,
  consumer: string,
  count = 10
) {
  const results = await redis.xreadgroup(
    'GROUP', group, consumer,
    'COUNT', count,
    'BLOCK', 2000,  // block 2 seconds if no messages
    'STREAMS', stream, '>'  // > = unread messages
  );
  
  if (!results) return [];
  
  return results.flatMap(([_stream, messages]) =>
    messages.map(([id, fields]) => ({
      id,
      data: Object.fromEntries(
        fields.reduce((acc: [string, string][], val, i) => {
          if (i % 2 === 0) acc.push([val, fields[i + 1] ?? '']);
          return acc;
        }, [])
      ),
    }))
  );
}
```

---

## 📌 Step 1158: สรุป Part 056

### เนื้อหาที่เรียนรู้

✅ Event-driven architecture patterns  
✅ Kafka + GraphQL integration  
✅ Consumer → PubSub bridge  
✅ Outbox pattern (dual-write prevention)  
✅ Change Data Capture (CDC) กับ Debezium  
✅ Redis Streams  

### Event Delivery Guarantees

```
At-most-once (Fire and forget):
- Redis Pub/Sub
- เหมาะ: real-time UI updates ที่ missed ok
  
At-least-once (กับ acknowledgment):
- Kafka consumer groups
- Redis Streams + ACK
- เหมาะ: business events ที่ต้อง process ครบ

Exactly-once:
- Kafka transactions
- Outbox + idempotent consumers
- เหมาะ: financial transactions, inventory
```

### ในส่วนถัดไป

➡️ **[Part 057](./part-057.md)** — CQRS with GraphQL

---

*Part 056 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
