# Part 025 — Redis Integration & Real-time Features ⚡

> **ระดับ:** Advanced | **เวลาเรียน:** 120 นาที | **ขั้นตอนที่:** 871–910

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Redis setup และ connection
- PubSub ด้วย Redis
- Real-time subscriptions ด้วย Redis
- Session management ใน Redis
- Rate limiting ด้วย Redis
- Leaderboard ด้วย Sorted Sets
- Queue ด้วย Redis (Bull/BullMQ)
- Pub/Sub patterns

---

## 📌 Step 871: Redis Setup

```bash
npm install redis ioredis
npm install graphql-redis-subscriptions
```

```javascript
// src/redis/client.js
import { createClient } from 'redis';

export const redis = createClient({
  url: process.env.REDIS_URL || 'redis://localhost:6379',
  socket: {
    reconnectStrategy: (retries) => {
      if (retries > 10) return new Error('Max retries exceeded');
      return Math.min(retries * 50, 500);
    },
  },
});

redis.on('error', (err) => console.error('Redis error:', err));
redis.on('connect', () => console.log('Redis connected'));
redis.on('reconnecting', () => console.log('Redis reconnecting...'));

await redis.connect();

export const subscriber = redis.duplicate();
await subscriber.connect();

export const publisher = redis.duplicate();
await publisher.connect();
```

---

## 📌 Step 872: Redis PubSub สำหรับ Subscriptions

```javascript
// src/pubsub/redis-pubsub.js
import { RedisPubSub } from 'graphql-redis-subscriptions';
import Redis from 'ioredis';

const ioRedis = new Redis(process.env.REDIS_URL);

export const pubsub = new RedisPubSub({
  publisher: ioRedis,
  subscriber: ioRedis.duplicate(),
  
  // serializer/deserializer
  serialize: JSON.stringify,
  deserialize: JSON.parse,
});

// Event channels
export const EVENTS = {
  NEW_MESSAGE: 'NEW_MESSAGE',
  USER_JOINED: 'USER_JOINED',
  USER_LEFT: 'USER_LEFT',
  POST_PUBLISHED: 'POST_PUBLISHED',
  ORDER_UPDATED: 'ORDER_UPDATED',
  NOTIFICATION: 'NOTIFICATION',
};
```

### Subscription Resolvers

```javascript
// src/resolvers/subscription.resolver.js
import { withFilter } from 'graphql-subscriptions';
import { pubsub, EVENTS } from '../pubsub/redis-pubsub.js';

export const subscriptionResolvers = {
  Subscription: {
    // Messages ใน channel
    newMessage: {
      subscribe: withFilter(
        () => pubsub.asyncIterator(EVENTS.NEW_MESSAGE),
        (payload, variables) => {
          return payload.newMessage.channelId === variables.channelId;
        }
      ),
    },
    
    // Order updates สำหรับ specific user
    orderUpdated: {
      subscribe: withFilter(
        () => pubsub.asyncIterator(EVENTS.ORDER_UPDATED),
        (payload, variables, context) => {
          if (!context.user) return false;
          return payload.orderUpdated.customerId === context.user.sub;
        }
      ),
    },
    
    // Notifications
    notification: {
      subscribe: withFilter(
        () => pubsub.asyncIterator(EVENTS.NOTIFICATION),
        (payload, _, context) => {
          if (!context.user) return false;
          const { userId } = payload.notification;
          return !userId || userId === context.user.sub;
        }
      ),
    },
  },
};

// Publish events (ใช้ใน resolvers)
export async function publishNewMessage(message) {
  await pubsub.publish(EVENTS.NEW_MESSAGE, { newMessage: message });
}

export async function publishOrderUpdate(order) {
  await pubsub.publish(EVENTS.ORDER_UPDATED, { orderUpdated: order });
}
```

---

## 📌 Step 873: Session Management

```javascript
// src/redis/session.js

const SESSION_TTL = 60 * 60 * 24 * 7; // 7 days

export async function createSession(userId, deviceInfo) {
  const sessionId = `sess_${uuidv4()}`;
  
  await redis.setEx(
    `session:${sessionId}`,
    SESSION_TTL,
    JSON.stringify({
      userId,
      deviceInfo,
      createdAt: new Date().toISOString(),
    })
  );
  
  // Track user's active sessions
  await redis.sAdd(`user_sessions:${userId}`, sessionId);
  await redis.expire(`user_sessions:${userId}`, SESSION_TTL);
  
  return sessionId;
}

export async function getSession(sessionId) {
  const data = await redis.get(`session:${sessionId}`);
  return data ? JSON.parse(data) : null;
}

export async function revokeSession(sessionId) {
  const session = await getSession(sessionId);
  if (session) {
    await redis.sRem(`user_sessions:${session.userId}`, sessionId);
    await redis.del(`session:${sessionId}`);
  }
}

export async function revokeAllSessions(userId) {
  const sessionIds = await redis.sMembers(`user_sessions:${userId}`);
  
  await Promise.all([
    ...sessionIds.map(id => redis.del(`session:${id}`)),
    redis.del(`user_sessions:${userId}`),
  ]);
}
```

---

## 📌 Step 874: Rate Limiting ด้วย Redis

```javascript
// src/redis/rate-limiter.js

export async function checkRateLimit(key, maxRequests, windowSeconds) {
  const now = Date.now();
  const windowMs = windowSeconds * 1000;
  const windowKey = `ratelimit:${key}:${Math.floor(now / windowMs)}`;
  
  const pipeline = redis.multi();
  pipeline.incr(windowKey);
  pipeline.expire(windowKey, windowSeconds);
  
  const [count] = await pipeline.exec();
  
  return {
    allowed: count <= maxRequests,
    remaining: Math.max(0, maxRequests - count),
    resetIn: windowSeconds - Math.floor((now % windowMs) / 1000),
  };
}

// Sliding window rate limit (accurate แต่ใช้ memory มากกว่า)
export async function slidingWindowRateLimit(key, maxRequests, windowSeconds) {
  const now = Date.now();
  const windowMs = windowSeconds * 1000;
  const rateKey = `ratelimit:sliding:${key}`;
  
  // Remove old requests
  await redis.zRemRangeByScore(rateKey, '-inf', now - windowMs);
  
  // Count current requests
  const count = await redis.zCard(rateKey);
  
  if (count >= maxRequests) {
    const oldest = await redis.zRange(rateKey, 0, 0, { withScores: true });
    const resetIn = Math.ceil((oldest[0].score + windowMs - now) / 1000);
    
    return { allowed: false, remaining: 0, resetIn };
  }
  
  // Add current request
  await redis.zAdd(rateKey, { score: now, value: `${now}-${Math.random()}` });
  await redis.expire(rateKey, windowSeconds);
  
  return {
    allowed: true,
    remaining: maxRequests - count - 1,
    resetIn: windowSeconds,
  };
}
```

---

## 📌 Step 875: Leaderboard

```javascript
// src/redis/leaderboard.js

const LEADERBOARD_KEY = 'leaderboard:scores';

export async function updateScore(userId, points) {
  return redis.zIncrBy(LEADERBOARD_KEY, points, userId);
}

export async function getTopUsers(limit = 10) {
  const results = await redis.zRangeWithScores(
    LEADERBOARD_KEY, 0, limit - 1, { rev: true }
  );
  
  return results.map((item, index) => ({
    rank: index + 1,
    userId: item.value,
    score: item.score,
  }));
}

export async function getUserRank(userId) {
  const [rank, score] = await Promise.all([
    redis.zRevRank(LEADERBOARD_KEY, userId),
    redis.zScore(LEADERBOARD_KEY, userId),
  ]);
  
  return {
    rank: rank !== null ? rank + 1 : null,
    score: score || 0,
  };
}

// GraphQL resolver
const resolvers = {
  Query: {
    leaderboard: async (_, { limit = 10 }) => {
      const topUsers = await getTopUsers(limit);
      
      // Batch load user data
      const users = await User.findByIds(topUsers.map(u => u.userId));
      const userMap = new Map(users.map(u => [u.id, u]));
      
      return topUsers.map(entry => ({
        ...entry,
        user: userMap.get(entry.userId),
      }));
    },
    
    myRank: async (_, __, { user }) => {
      if (!user) throw new GraphQLError('Unauthorized');
      return getUserRank(user.sub);
    },
  },
};
```

---

## 📌 Step 876: BullMQ Queue

```bash
npm install bullmq
```

```javascript
// src/queue/index.js
import { Queue, Worker } from 'bullmq';

const connection = { url: process.env.REDIS_URL };

// สร้าง queues
export const emailQueue = new Queue('email', { connection });
export const imageQueue = new Queue('image-processing', { connection });
export const notificationQueue = new Queue('notifications', { connection });

// Workers
const emailWorker = new Worker('email', async (job) => {
  const { to, subject, template, data } = job.data;
  await sendEmail(to, subject, template, data);
}, { connection });

const imageWorker = new Worker('image-processing', async (job) => {
  const { imageId, operations } = job.data;
  await processImage(imageId, operations);
}, { connection, concurrency: 5 });

// Use in resolvers
const resolvers = {
  Mutation: {
    createUser: async (_, { input }) => {
      const user = await User.create(input);
      
      // ส่ง welcome email แบบ async (ไม่รอ)
      await emailQueue.add('welcome-email', {
        to: user.email,
        subject: 'Welcome!',
        template: 'welcome',
        data: { name: user.name },
      });
      
      return user;
    },
    
    uploadProductImage: async (_, { file, productId }) => {
      const upload = await saveFile(file);
      
      // Process image ใน background
      await imageQueue.add('resize', {
        imageId: upload.id,
        operations: [
          { type: 'resize', width: 800, height: 800 },
          { type: 'thumbnail', width: 200, height: 200 },
          { type: 'webp' },
        ],
      });
      
      return upload;
    },
  },
};
```

---

## 📌 Step 877: สรุป Part 025

### เนื้อหาที่เรียนรู้

✅ Redis setup + connection  
✅ Redis PubSub สำหรับ subscriptions  
✅ Session management  
✅ Rate limiting (fixed + sliding window)  
✅ Leaderboard ด้วย Sorted Sets  
✅ BullMQ background queue  

### Homework

1. implement Redis PubSub สำหรับ chat app
2. เพิ่ม sliding window rate limiting
3. สร้าง leaderboard สำหรับ quiz app

### ในส่วนถัดไป

➡️ **[Part 026](./part-026.md)** — Docker & Kubernetes Deployment

---

*Part 025 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
