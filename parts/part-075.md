# Part 075 — Advanced Subscriptions & Real-Time Patterns 📡

> **ระดับ:** Expert | **เวลาเรียน:** 85 นาที | **ขั้นตอนที่:** 1901–1940

---

## 🎯 สิ่งที่จะได้เรียนรู้

- WebSocket vs SSE สำหรับ subscriptions
- graphql-ws protocol
- Subscription auth & reconnect
- Scalable PubSub (Redis Pub/Sub, NATS)
- Filtered subscriptions
- Subscription rate limiting
- GraphQL Live Queries
- Real-time dashboard patterns

---

## 📌 Step 1901: WebSocket vs SSE

```
WebSocket:
✅ Bidirectional: client และ server ส่งได้ทั้งคู่
✅ Low latency
✅ Standard protocol
❌ Stateful: sticky sessions จำเป็น
❌ ผ่าน proxies ยากกว่า
❌ Load balancing ซับซ้อน
→ ใช้สำหรับ: subscriptions ที่ต้องการ low-latency สูง

Server-Sent Events (SSE):
✅ HTTP-based: ไม่ต้องการ sticky sessions
✅ Auto-reconnect built-in
✅ Simpler: unidirectional (server → client only)
✅ HTTP/2 multiplexing
❌ Unidirectional เท่านั้น
→ ใช้สำหรับ: live queries, dashboards, notifications

graphql-over-http spec:
- แนะนำ SSE สำหรับ subscriptions ที่ไม่ต้องการ bidirectional
```

---

## 📌 Step 1902: graphql-ws Setup

```bash
npm install graphql-ws ws
```

```typescript
// src/subscriptions/ws-server.ts
import { createServer } from 'http';
import { WebSocketServer } from 'ws';
import { useServer } from 'graphql-ws/lib/use/ws';
import { schema } from '../schema.js';

const httpServer = createServer(expressApp);

const wsServer = new WebSocketServer({
  server: httpServer,
  path: '/graphql',
});

// Setup graphql-ws
const serverCleanup = useServer(
  {
    schema,
    
    // Auth: validate on connection
    onConnect: async (ctx) => {
      const token = ctx.connectionParams?.authorization as string;
      
      if (!token) {
        return { user: null }; // Anonymous
      }
      
      try {
        const user = verifyToken(token);
        return { user };  // Stored in context
      } catch {
        throw new Error('Unauthorized');  // Reject connection
      }
    },
    
    // Create context per subscription
    context: async (ctx, msg, args) => {
      const { user } = ctx.extra as { user: UserPayload | null };
      return {
        user,
        prisma,
        pubsub,
        loaders: createDataLoaders(),
      };
    },
    
    // Handle subscription errors
    onError: (ctx, msg, errors) => {
      console.error('Subscription error:', errors);
    },
    
    // Cleanup on disconnect
    onComplete: async (ctx, msg) => {
      console.log('Client disconnected:', ctx.connectionId);
      // Release any resources
    },
  },
  wsServer
);

// Shutdown cleanup
process.on('SIGTERM', () => {
  serverCleanup.dispose();
});
```

---

## 📌 Step 1903: SSE สำหรับ Subscriptions

```typescript
// src/subscriptions/sse-handler.ts
// Server-Sent Events: ดีกว่า WebSocket สำหรับ horizontal scaling

import { execute, parse, subscribe } from 'graphql';

app.get('/graphql/sse', async (req, res) => {
  // SSE headers
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.setHeader('X-Accel-Buffering', 'no');  // Nginx: ปิด buffering
  res.flushHeaders();
  
  // Parse query from URL params
  const { query, variables, operationName } = req.query;
  
  if (!query || typeof query !== 'string') {
    res.write('event: error\ndata: {"message": "Missing query"}\n\n');
    return res.end();
  }
  
  // Auth
  const token = req.headers.authorization?.split(' ')[1];
  const user = token ? verifyToken(token) : null;
  
  const context: AppContext = {
    user,
    prisma,
    pubsub,
    loaders: createDataLoaders(),
  };
  
  try {
    const document = parse(query as string);
    const subscription = await subscribe({
      schema,
      document,
      variableValues: variables ? JSON.parse(variables as string) : undefined,
      operationName: operationName as string,
      contextValue: context,
    });
    
    if ('errors' in subscription) {
      res.write(`event: error\ndata: ${JSON.stringify({ errors: subscription.errors })}\n\n`);
      return res.end();
    }
    
    // Stream events
    (async () => {
      for await (const result of subscription) {
        if (res.writableEnded) break;
        
        res.write(`data: ${JSON.stringify(result)}\n\n`);
      }
    })();
    
    // Client disconnect
    req.on('close', () => {
      // Subscription will be GC'd
    });
    
  } catch (err) {
    res.write(`event: error\ndata: ${JSON.stringify({ message: String(err) })}\n\n`);
    res.end();
  }
});

// Client-side SSE usage
// const source = new EventSource('/graphql/sse?query=subscription{orderUpdated{id status}}');
// source.onmessage = (e) => console.log(JSON.parse(e.data));
// source.addEventListener('error', (e) => console.error('SSE error', e));
```

---

## 📌 Step 1904: Scalable PubSub กับ NATS

```bash
npm install nats
```

```typescript
// src/pubsub/nats-pubsub.ts
import { connect, NatsConnection, StringCodec } from 'nats';
import { PubSubEngine } from 'graphql-subscriptions';

export class NatsPubSub implements PubSubEngine {
  private nc!: NatsConnection;
  private sc = StringCodec();
  private subscriptions = new Map<number, { subject: string; sub: ReturnType<NatsConnection['subscribe']> }>();
  private subIdCounter = 0;
  
  async connect(servers: string | string[]) {
    this.nc = await connect({ servers });
    return this;
  }
  
  async publish(triggerName: string, payload: unknown): Promise<void> {
    this.nc.publish(triggerName, this.sc.encode(JSON.stringify(payload)));
  }
  
  async subscribe(
    triggerName: string,
    onMessage: Function,
    options?: unknown
  ): Promise<number> {
    const id = ++this.subIdCounter;
    
    const sub = this.nc.subscribe(triggerName, {
      callback: (err, msg) => {
        if (err) return;
        try {
          const payload = JSON.parse(this.sc.decode(msg.data));
          onMessage(payload);
        } catch (e) {
          console.error('NATS parse error:', e);
        }
      },
    });
    
    this.subscriptions.set(id, { subject: triggerName, sub });
    return id;
  }
  
  unsubscribe(subId: number) {
    const sub = this.subscriptions.get(subId);
    if (sub) {
      sub.sub.unsubscribe();
      this.subscriptions.delete(subId);
    }
  }
  
  async drain() {
    await this.nc.drain();
  }
}

// Singleton
export const pubsub = await new NatsPubSub().connect(
  process.env.NATS_URL ?? 'nats://localhost:4222'
);
```

---

## 📌 Step 1905: Filtered Subscriptions

```typescript
// src/resolvers/subscriptions.ts
// Filter subscriptions per-user ไม่ให้รับ events ของคนอื่น

const subscriptionResolvers = {
  Subscription: {
    // ดู order updates ของ user นี้เท่านั้น
    orderUpdated: {
      subscribe: withFilter(
        (_, __, ctx) => {
          requireAuth(ctx);
          // Subscribe ไปยัง NATS subject สำหรับ user นี้
          return pubsub.asyncIterableIterator(`ORDER_UPDATED:${ctx.user!.id}`);
        },
        
        // Filter function: return true ถ้า event นี้ควรส่งให้ subscriber นี้
        (payload, variables, ctx) => {
          // Double-check: ป้องกัน edge cases
          return payload.orderUpdated.customerId === ctx.user!.id;
        }
      ),
      
      resolve: (payload) => payload.orderUpdated,
    },
    
    // Live inventory updates (public, filtered by product)
    inventoryChanged: {
      subscribe: withFilter(
        (_, { productIds }) => {
          return pubsub.asyncIterableIterator('INVENTORY_CHANGED');
        },
        (payload, { productIds }) => {
          // Only send if product is in the requested list
          return !productIds?.length || productIds.includes(payload.inventoryChanged.productId);
        }
      ),
      
      resolve: (payload) => payload.inventoryChanged,
    },
    
    // Admin-only: all order updates
    allOrderUpdates: {
      subscribe: (_, __, ctx) => {
        requireRole(ctx, 'ADMIN');
        return pubsub.asyncIterableIterator('ORDER_UPDATED');
      },
      
      resolve: (payload) => payload.orderUpdated,
    },
  },
};

// Publish ใน mutation
const fulfillOrderMutation = async (_, { orderId }, ctx) => {
  const order = await ctx.prisma.order.update({
    where: { id: orderId },
    data: { status: 'SHIPPED' },
    include: { customer: true },
  });
  
  // Publish to user-specific topic
  await pubsub.publish(`ORDER_UPDATED:${order.customerId}`, {
    orderUpdated: order,
  });
  
  // Also publish to admin topic
  await pubsub.publish('ORDER_UPDATED', {
    orderUpdated: order,
  });
  
  return order;
};
```

---

## 📌 Step 1906: สรุป Part 075

### เนื้อหาที่เรียนรู้

✅ WebSocket vs SSE comparison  
✅ graphql-ws setup  
✅ SSE subscriptions  
✅ NATS PubSub  
✅ Filtered subscriptions  
✅ Auth สำหรับ subscriptions  

### Production Subscription Checklist

```
Transport:
□ graphql-ws protocol (ไม่ใช่ subscriptions-transport-ws เก่า)
□ SSE สำหรับ unidirectional use cases
□ Auth บน connect ไม่ใช่ per-message
□ Reconnect handling ใน client

Scalability:
□ ไม่ใช้ in-memory PubSub (ไม่ scale)
□ Redis/NATS/Kafka สำหรับ multi-instance
□ Sticky sessions (WebSocket) หรือ stateless (SSE)
□ Connection limit per user

Security:
□ Auth ทุก subscription
□ Filter: user รับเฉพาะ events ของตัวเอง
□ Rate limit subscriptions
□ Max concurrent subscriptions per user
```

### ในส่วนถัดไป

➡️ **[Part 076](./part-076.md)** — GraphQL File Upload & Binary Data

---

*Part 075 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
