# Part 091 — GraphQL กับ Message Queues 📨

> **ระดับ:** Expert | **เวลาเรียน:** 80 นาที | **ขั้นตอนที่:** 2571–2615

---

## 🎯 สิ่งที่จะได้เรียนรู้

- เมื่อไหร่ควรใช้ Message Queue
- RabbitMQ กับ GraphQL mutations
- AWS SQS integration
- Event-driven resolvers
- CQRS pattern
- Outbox pattern
- Dead letter queues
- Background job processing

---

## 📌 Step 2571: เมื่อไหร่ใช้ Message Queue

```
ใช้ Message Queue เมื่อ:
✅ งานที่ใช้เวลานาน (video processing, email sending)
✅ Fan-out: ส่ง event ให้หลาย consumers
✅ Peak traffic: buffer requests
✅ Reliability: retry on failure
✅ Decoupling: producer ไม่รอ consumer

ไม่ต้องใช้:
❌ Simple CRUD ที่ต้องการ immediate response
❌ Small scale (< 1000 req/day)

GraphQL + Queue pattern:
  Client → Mutation → Queue → Worker → DB
  
  1. Mutation รับ request → publish to queue → return jobId
  2. Worker process → update DB → notify via subscription
  3. Client poll jobId หรือ subscribe สำหรับ result
```

---

## 📌 Step 2572: RabbitMQ Integration

```bash
npm install amqplib @types/amqplib bull
```

```typescript
// src/queue/rabbitmq.ts
import amqp, { Channel, Connection } from 'amqplib';

class RabbitMQClient {
  private connection: Connection | null = null;
  private channel: Channel | null = null;
  
  async connect() {
    this.connection = await amqp.connect(process.env.RABBITMQ_URL!);
    this.channel = await this.connection.createChannel();
    
    // Dead letter exchange
    await this.channel.assertExchange('dlx', 'direct', { durable: true });
    await this.channel.assertQueue('dead-letter', {
      durable: true,
      arguments: { 'x-queue-type': 'classic' },
    });
    await this.channel.bindQueue('dead-letter', 'dlx', 'dead');
    
    // Main queues
    await this.channel.assertQueue('email-notifications', {
      durable: true,
      arguments: {
        'x-dead-letter-exchange': 'dlx',
        'x-dead-letter-routing-key': 'dead',
        'x-message-ttl': 86400000,  // 24 ชั่วโมง
      },
    });
    
    await this.channel.assertQueue('image-processing', {
      durable: true,
      arguments: {
        'x-dead-letter-exchange': 'dlx',
        'x-dead-letter-routing-key': 'dead',
      },
    });
    
    // Prefetch: process 5 messages ต่อครั้ง
    await this.channel.prefetch(5);
  }
  
  async publish(queue: string, message: object) {
    if (!this.channel) throw new Error('Not connected');
    
    return this.channel.sendToQueue(
      queue,
      Buffer.from(JSON.stringify(message)),
      {
        persistent: true,
        contentType: 'application/json',
        timestamp: Date.now(),
        messageId: crypto.randomUUID(),
      }
    );
  }
  
  async consume(queue: string, handler: (message: unknown, ack: () => void, nack: (requeue?: boolean) => void) => Promise<void>) {
    if (!this.channel) throw new Error('Not connected');
    
    const ch = this.channel;
    
    await ch.consume(queue, async (msg) => {
      if (!msg) return;
      
      const content = JSON.parse(msg.content.toString());
      
      await handler(
        content,
        () => ch.ack(msg),
        (requeue = false) => ch.nack(msg, false, requeue)
      );
    });
  }
}

export const rabbit = new RabbitMQClient();
```

---

## 📌 Step 2573: GraphQL Mutation → Queue

```typescript
// src/resolvers/order.resolver.ts
const orderResolvers = {
  Mutation: {
    // Long-running: return jobId ทันที
    processOrder: async (_: unknown, { orderId }: { orderId: string }, { user, prisma }: AppContext) => {
      requireAuth(user);
      
      // Validate order exists
      const order = await prisma.order.findUnique({ where: { id: orderId } });
      if (!order) throw new AppError('Order not found', ErrorCode.NOT_FOUND);
      if (order.userId !== user.id) throw new AppError('Forbidden', ErrorCode.FORBIDDEN);
      
      // Create job record
      const job = await prisma.backgroundJob.create({
        data: {
          id: crypto.randomUUID(),
          type: 'PROCESS_ORDER',
          status: 'PENDING',
          payload: { orderId },
          userId: user.id,
        },
      });
      
      // Publish to queue
      await rabbit.publish('order-processing', {
        jobId: job.id,
        orderId,
        userId: user.id,
        timestamp: new Date().toISOString(),
      });
      
      return { jobId: job.id, status: 'PENDING' };
    },
  },
  
  Query: {
    // Client poll จนกว่า job จะ complete
    jobStatus: async (_: unknown, { jobId }: { jobId: string }, { user, prisma }: AppContext) => {
      requireAuth(user);
      
      const job = await prisma.backgroundJob.findUnique({ where: { id: jobId } });
      if (!job || job.userId !== user.id) throw new AppError('Job not found', ErrorCode.NOT_FOUND);
      
      return job;
    },
  },
};
```

---

## 📌 Step 2574: Worker Process

```typescript
// src/workers/order-worker.ts
// Worker: ทำงานแยกจาก API server

import { rabbit } from '../queue/rabbitmq';
import { prisma } from '../db/prisma';
import { pubsub } from '../pubsub/nats';

async function startOrderWorker() {
  await rabbit.connect();
  
  await rabbit.consume('order-processing', async (message, ack, nack) => {
    const { jobId, orderId } = message as { jobId: string; orderId: string };
    
    try {
      // Update job status
      await prisma.backgroundJob.update({
        where: { id: jobId },
        data: { status: 'PROCESSING', startedAt: new Date() },
      });
      
      // Process order (ใช้เวลานาน)
      await processOrderInternal(orderId);
      
      // Update job as complete
      await prisma.backgroundJob.update({
        where: { id: jobId },
        data: { status: 'COMPLETED', completedAt: new Date() },
      });
      
      // Notify via subscription
      await pubsub.publish(`JOB_COMPLETED:${jobId}`, {
        jobStatus: { id: jobId, status: 'COMPLETED' },
      });
      
      ack();
      
    } catch (err) {
      logger.error({ jobId, error: err }, 'Order processing failed');
      
      // Update job as failed
      await prisma.backgroundJob.update({
        where: { id: jobId },
        data: {
          status: 'FAILED',
          error: (err as Error).message,
          failedAt: new Date(),
        },
      });
      
      // nack(false) = don't requeue (send to DLQ)
      // nack(true) = requeue (retry)
      const retryable = isRetryableError(err as Error);
      nack(retryable);
    }
  });
  
  logger.info('Order worker started');
}

startOrderWorker();
```

---

## 📌 Step 2575: Bull Queue สำหรับ Node.js

```typescript
// src/queue/bull-jobs.ts
// Bull: Redis-based job queue สำหรับ Node.js
import Bull, { Queue, Job } from 'bull';

// ประเภท jobs
const emailQueue = new Bull('email', {
  redis: { host: process.env.REDIS_HOST, port: 6379 },
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 5000 },
    removeOnComplete: 100,  // เก็บ 100 completed jobs
    removeOnFail: 200,
  },
});

const imageQueue = new Bull('image-processing', {
  redis: { host: process.env.REDIS_HOST, port: 6379 },
  limiter: {
    max: 10,        // process max 10 jobs
    duration: 1000, // ต่อ 1 วินาที
  },
});

// Processor สำหรับ email jobs
emailQueue.process('order-confirmation', async (job: Job) => {
  const { orderId, userEmail, orderDetails } = job.data;
  
  await job.progress(10);
  
  const emailContent = await renderOrderEmail(orderDetails);
  await job.progress(50);
  
  await sendEmail({
    to: userEmail,
    subject: `ยืนยันคำสั่งซื้อ #${orderId}`,
    html: emailContent,
  });
  
  await job.progress(100);
  
  return { sent: true };
});

// Mutation ที่ใช้ Bull
const notificationResolvers = {
  Mutation: {
    placeOrder: async (_: unknown, { input }: PlaceOrderArgs, ctx: AppContext) => {
      requireAuth(ctx.user);
      
      const order = await createOrderInDB(input, ctx);
      
      // Queue email + image processing jobs
      await emailQueue.add('order-confirmation', {
        orderId: order.id,
        userEmail: ctx.user!.email,
        orderDetails: order,
      });
      
      // ถ้ามี product image upload
      if (input.productImageUpload) {
        await imageQueue.add('resize', {
          uploadId: input.productImageUpload,
          orderId: order.id,
        });
      }
      
      return order;
    },
  },
};
```

---

## 📌 Step 2576: Outbox Pattern

```typescript
// src/patterns/outbox.ts
// Outbox Pattern: guarantees event delivery even if queue is down
// 1. DB transaction: บันทึก record + outbox event ด้วยกัน
// 2. Separate process: อ่าน outbox → publish → mark as sent

// Prisma schema
// model OutboxEvent {
//   id          String    @id
//   type        String
//   payload     Json
//   status      String    @default("PENDING")  // PENDING, SENT, FAILED
//   createdAt   DateTime  @default(now())
//   sentAt      DateTime?
// }

const orderResolvers = {
  Mutation: {
    createOrder: async (_: unknown, { input }: CreateOrderArgs, { prisma, user }: AppContext) => {
      requireAuth(user);
      
      // Transaction: order + outbox event
      const [order] = await prisma.$transaction([
        prisma.order.create({ data: { ...input, userId: user!.id } }),
        prisma.outboxEvent.create({
          data: {
            id: crypto.randomUUID(),
            type: 'ORDER_CREATED',
            payload: { orderId: '...', userId: user!.id },
          },
        }),
      ]);
      
      return order;
    },
  },
};

// Outbox processor (polls DB ทุก 1 วินาที)
async function processOutbox() {
  const events = await prisma.outboxEvent.findMany({
    where: { status: 'PENDING' },
    orderBy: { createdAt: 'asc' },
    take: 100,
  });
  
  for (const event of events) {
    try {
      await rabbit.publish(`events.${event.type}`, event.payload);
      await prisma.outboxEvent.update({
        where: { id: event.id },
        data: { status: 'SENT', sentAt: new Date() },
      });
    } catch (err) {
      await prisma.outboxEvent.update({
        where: { id: event.id },
        data: { status: 'FAILED' },
      });
    }
  }
}

setInterval(processOutbox, 1000);
```

---

## 📌 Step 2577: สรุป Part 091

### เนื้อหาที่เรียนรู้

✅ RabbitMQ integration  
✅ Mutation → Queue → Worker pattern  
✅ Bull queue สำหรับ Node.js  
✅ Outbox pattern  
✅ Dead letter queues  
✅ Job status tracking + subscription  

### Message Queue Architecture

```
Patterns:
1. Fire-and-forget: mutation → queue → worker (ไม่รอ result)
2. Request-reply: mutation → queue → worker → subscription
3. Event sourcing: mutation → event store → projections
4. Outbox: transaction + event → outbox poller → queue

Queue vs Direct:
Direct (sync):  < 100ms expected, user waits
Queue (async):  > 1s expected, email/video/reports

Reliability:
□ Dead letter queue สำหรับ failed messages
□ Retry with exponential backoff
□ Idempotent consumers (prevent double-processing)
□ Outbox pattern สำหรับ guaranteed delivery
□ Monitor queue depth + consumer lag
```

### ในส่วนถัดไป

➡️ **[Part 092](./part-092.md)** — Code-First Schema: Pothos & TypeGraphQL

---

*Part 091 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
