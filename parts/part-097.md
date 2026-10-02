# Part 097 — Real-Time Systems: Chat & Collaborative Apps 💬

> **ระดับ:** World-Class | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 2841–2885

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Real-time chat architecture
- Presence system (typing indicators, online status)
- Collaborative document editing
- Optimistic UI สำหรับ messages
- Message ordering guarantees
- Read receipts
- Push notifications fallback
- Scale: 1M concurrent connections

---

## 📌 Step 2841: Chat System Schema

```graphql
# schema/chat.graphql

type Conversation {
  id: ID!
  participants: [User!]!
  messages(first: Int = 50, before: String): MessageConnection!
  lastMessage: Message
  unreadCount: Int!
  createdAt: String!
}

type Message {
  id: ID!
  conversationId: ID!
  sender: User!
  content: String!
  contentType: MessageContentType!
  readBy: [MessageReadReceipt!]!
  reactions: [MessageReaction!]!
  replyTo: Message
  createdAt: String!
  updatedAt: String!
  deletedAt: String     # null = ยังไม่ลบ
  status: MessageStatus!
}

enum MessageContentType {
  TEXT
  IMAGE
  FILE
  SYSTEM
}

enum MessageStatus {
  SENDING    # optimistic
  SENT
  DELIVERED
  READ
  FAILED
}

type MessageReadReceipt {
  user: User!
  readAt: String!
}

type Subscription {
  # Subscribe to conversation updates
  conversationUpdated(conversationId: ID!): ConversationEvent!
  
  # Presence: typing indicators
  userPresence(conversationId: ID!): PresenceEvent!
}

union ConversationEvent = 
  NewMessageEvent |
  MessageUpdatedEvent |
  MessageDeletedEvent |
  MessageReactionEvent

type NewMessageEvent {
  message: Message!
}

type PresenceEvent {
  userId: ID!
  type: PresenceEventType!  # TYPING_START, TYPING_STOP, ONLINE, OFFLINE
  conversationId: ID!
}
```

---

## 📌 Step 2842: Message Sending กับ Optimistic UI

```typescript
// src/resolvers/chat.resolver.ts
import { withFilter } from 'graphql-subscriptions';

const chatResolvers = {
  Mutation: {
    sendMessage: async (
      _: unknown,
      { input }: SendMessageArgs,
      { user, prisma, pubsub }: AppContext
    ) => {
      requireAuth(user);
      
      // Verify participant
      const conversation = await prisma.conversation.findFirst({
        where: {
          id: input.conversationId,
          participants: { some: { id: user!.id } },
        },
      });
      
      if (!conversation) throw new AppError('Conversation not found', ErrorCode.NOT_FOUND);
      
      // Generate client-side ID (if provided) สำหรับ deduplication
      const existingMsg = input.clientId
        ? await prisma.message.findFirst({ where: { clientId: input.clientId } })
        : null;
      
      if (existingMsg) return existingMsg;  // Idempotent
      
      const message = await prisma.message.create({
        data: {
          id: crypto.randomUUID(),
          clientId: input.clientId,
          conversationId: input.conversationId,
          senderId: user!.id,
          content: input.content,
          contentType: input.contentType ?? 'TEXT',
          replyToId: input.replyToId,
        },
        include: { sender: true, replyTo: { include: { sender: true } } },
      });
      
      // Publish to all participants
      await pubsub.publish(`CONVERSATION:${input.conversationId}`, {
        conversationUpdated: {
          __typename: 'NewMessageEvent',
          message,
        },
      });
      
      // Update conversation lastMessage
      await prisma.conversation.update({
        where: { id: input.conversationId },
        data: { lastMessageAt: new Date() },
      });
      
      return message;
    },
    
    // Typing indicator
    setTypingStatus: async (
      _: unknown,
      { conversationId, isTyping }: { conversationId: string; isTyping: boolean },
      { user, pubsub, redis }: AppContext
    ) => {
      requireAuth(user);
      
      // Store typing status ใน Redis (TTL 5 วินาที)
      const key = `typing:${conversationId}:${user!.id}`;
      if (isTyping) {
        await redis.setex(key, 5, '1');
      } else {
        await redis.del(key);
      }
      
      // Notify other participants
      await pubsub.publish(`PRESENCE:${conversationId}`, {
        userPresence: {
          userId: user!.id,
          type: isTyping ? 'TYPING_START' : 'TYPING_STOP',
          conversationId,
        },
      });
      
      return { success: true };
    },
  },
  
  Subscription: {
    conversationUpdated: {
      subscribe: withFilter(
        (_, { conversationId }, ctx) => {
          requireAuth(ctx.user);
          return ctx.pubsub.asyncIterator(`CONVERSATION:${conversationId}`);
        },
        async (payload, { conversationId }, ctx) => {
          // Verify subscriber is participant
          const isParticipant = await prisma.conversation.findFirst({
            where: {
              id: conversationId,
              participants: { some: { id: ctx.user.id } },
            },
          });
          return !!isParticipant;
        }
      ),
    },
    
    userPresence: {
      subscribe: withFilter(
        (_, { conversationId }) =>
          pubsub.asyncIterator(`PRESENCE:${conversationId}`),
        (payload, _, ctx) =>
          payload.userPresence.userId !== ctx.user?.id  // ไม่ส่งหา sender เอง
      ),
    },
  },
};
```

---

## 📌 Step 2843: Read Receipts

```typescript
// src/resolvers/read-receipt.resolver.ts

const readReceiptResolvers = {
  Mutation: {
    markMessagesRead: async (
      _: unknown,
      { conversationId, lastMessageId }: MarkReadArgs,
      { user, prisma, pubsub }: AppContext
    ) => {
      requireAuth(user);
      
      // Mark all messages up to lastMessageId as read
      const messagesRead = await prisma.$executeRaw`
        INSERT INTO message_read_receipts (message_id, user_id, read_at)
        SELECT m.id, ${user!.id}, NOW()
        FROM messages m
        WHERE m.conversation_id = ${conversationId}
          AND m.sender_id != ${user!.id}
          AND m.created_at <= (
            SELECT created_at FROM messages WHERE id = ${lastMessageId}
          )
        ON CONFLICT (message_id, user_id) DO NOTHING
      `;
      
      // Notify sender that messages were read
      await pubsub.publish(`CONVERSATION:${conversationId}`, {
        conversationUpdated: {
          __typename: 'ReadReceiptsUpdatedEvent',
          userId: user!.id,
          conversationId,
          lastReadMessageId: lastMessageId,
        },
      });
      
      return { count: messagesRead };
    },
  },
};
```

---

## 📌 Step 2844: Presence System (Online Status)

```typescript
// src/presence/presence.service.ts
// Track online users ด้วย Redis

class PresenceService {
  private readonly TTL = 30;  // 30 วินาที
  
  constructor(private redis: Redis, private pubsub: PubSubEngine) {}
  
  async setOnline(userId: string) {
    const wasOffline = !(await this.redis.exists(`presence:${userId}`));
    await this.redis.setex(`presence:${userId}`, this.TTL, '1');
    
    if (wasOffline) {
      await this.pubsub.publish('USER_PRESENCE', { userId, type: 'ONLINE' });
    }
  }
  
  async setOffline(userId: string) {
    const wasOnline = await this.redis.exists(`presence:${userId}`);
    await this.redis.del(`presence:${userId}`);
    
    if (wasOnline) {
      await this.pubsub.publish('USER_PRESENCE', { userId, type: 'OFFLINE' });
    }
  }
  
  async isOnline(userId: string): Promise<boolean> {
    return !!(await this.redis.exists(`presence:${userId}`));
  }
  
  async getOnlineUsers(userIds: string[]): Promise<Set<string>> {
    const pipeline = this.redis.pipeline();
    for (const id of userIds) pipeline.exists(`presence:${id}`);
    const results = await pipeline.exec();
    
    return new Set(
      userIds.filter((id, i) => results?.[i]?.[1] === 1)
    );
  }
}

// Heartbeat: client ping ทุก 20 วินาที เพื่อ maintain presence
const presenceResolvers = {
  Mutation: {
    heartbeat: async (_: unknown, _args: unknown, { user, presence }: AppContext) => {
      if (!user) return { online: false };
      await presence.setOnline(user.id);
      return { online: true };
    },
  },
};
```

---

## 📌 Step 2845: Scale: 1M Concurrent Connections

```
Architecture สำหรับ 1M WebSocket connections:

1. Horizontal scaling WebSocket servers:
   - sticky sessions (IP hash หรือ cookie)
   - หรือ stateless routing ด้วย Redis adapter

2. Redis PubSub adapter:
   - ทุก server subscribe ต่อ Redis
   - Publish หนึ่งครั้ง → fan-out ทุก server → ส่งไปแต่ละ client
   - Redis Cluster สำหรับ high throughput

3. บน Kubernetes:
   - WebSocket servers: 50 replicas × 20,000 connections = 1M
   - HPA บน active connections metric
   - Ingress: NGINX proxy-read-timeout: 3600
   
4. Connection limits:
   - Node.js: ulimit -n 65536 file descriptors
   - OS: sysctl net.core.somaxconn=65535
   - Container resources: memory ≈ 2MB per connection (1M = 2GB+)

5. Message fan-out optimization:
   - ไม่ส่ง full payload ผ่าน Redis
   - ส่งแค่ event notification → server fetch payload จาก DB/cache

6. Monitoring:
   - active_connections metric
   - messages_per_second
   - subscription_duration_histogram
```

---

## 📌 Step 2846: สรุป Part 097

### เนื้อหาที่เรียนรู้

✅ Chat system schema  
✅ Message sending + idempotency  
✅ Typing indicators  
✅ Read receipts  
✅ Presence system  
✅ Scale considerations  

### Chat Architecture Checklist

```
Core Features:
□ Message deduplication (clientId)
□ Optimistic UI pattern
□ Read receipts
□ Typing indicators (Redis TTL 5s)
□ Online presence (heartbeat + Redis TTL 30s)
□ Message pagination (cursor-based, newest first)

Performance:
□ Messages table: index (conversation_id, created_at)
□ Read receipts: bulk insert
□ Presence: Redis (ไม่ใช้ DB)
□ Subscription fan-out: Redis PubSub

Scale:
□ Sticky sessions หรือ Redis adapter
□ Memory per connection (~2MB)
□ Horizontal scaling WebSocket servers
□ Monitor active_connections metric
```

### ในส่วนถัดไป

➡️ **[Part 098](./part-098.md)** — GraphQL AI Integration: LLM + GraphQL APIs

---

*Part 097 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
