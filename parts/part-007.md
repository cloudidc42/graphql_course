# Part 007 — Subscriptions: Real-time Data

> **ระดับ:** Beginner→Intermediate | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 151–180

---

## 🎯 สิ่งที่จะได้เรียนรู้

- GraphQL Subscriptions คืออะไร
- WebSocket protocol
- ติดตั้ง subscription server
- PubSub system
- Real-time chat application
- Subscription filters
- Authentication ใน subscriptions

---

## 📌 Step 151: Subscriptions คืออะไร

**Subscriptions** ใช้สำหรับรับข้อมูล **real-time** จาก server เมื่อมีการเปลี่ยนแปลง:

```
ปกติ (Query/Mutation):
Client → Request → Server → Response → Client
(One-time, client-initiated)

Subscription:
Client → Subscribe → Server ─────────────────→ Client
                              (event เกิดขึ้น) → Client
                              (event เกิดขึ้น) → Client
                              (event เกิดขึ้น) → Client
(Persistent connection, server-pushed)
```

### Use Cases จริง

```
✅ Real-time chat
✅ Live notifications
✅ Stock price updates
✅ Sports scores
✅ Collaborative editing (Google Docs style)
✅ Live order tracking
✅ Dashboard metrics
✅ Online presence (who's online)
✅ Gaming (game state updates)
```

---

## 📌 Step 152: WebSocket vs HTTP

```
HTTP (Query/Mutation):
- Request-Response pattern
- Short-lived connection
- Client initiates
- HTTP/1.1, HTTP/2

WebSocket (Subscription):
- Bidirectional communication
- Persistent connection
- Both sides can send
- ws:// หรือ wss:// (secure)
```

### Protocols สำหรับ GraphQL Subscriptions

```
1. graphql-ws (ใหม่กว่า, แนะนำ)
   npm install graphql-ws

2. subscriptions-transport-ws (เก่า, deprecated)
   npm install subscriptions-transport-ws
```

---

## 📌 Step 153: ติดตั้ง Dependencies

```bash
npm install graphql-ws ws @graphql-tools/schema
```

---

## 📌 Step 154: Schema สำหรับ Subscriptions

```graphql
type Subscription {
  # User events
  userCreated: User!
  userUpdated(id: ID): User!
  userDeleted: ID!
  
  # Post events
  postCreated: Post!
  postUpdated(id: ID): Post!
  postPublished: Post!
  
  # Comment events
  commentAdded(postId: ID!): Comment!
  commentDeleted(postId: ID!): ID!
  
  # Real-time chat
  messageSent(roomId: ID!): Message!
  
  # Notifications
  notificationReceived(userId: ID!): Notification!
  
  # Online presence
  userOnlineStatusChanged: OnlineStatus!
  
  # Live counters
  postLikeCount(postId: ID!): LikeCountUpdate!
}

type Message {
  id: ID!
  text: String!
  author: User!
  room: Room!
  createdAt: String!
}

type Room {
  id: ID!
  name: String!
  members: [User!]!
}

type Notification {
  id: ID!
  type: NotificationType!
  title: String!
  body: String!
  isRead: Boolean!
  createdAt: String!
}

enum NotificationType {
  COMMENT_ON_POST
  LIKE_POST
  NEW_FOLLOWER
  MENTION
  SYSTEM
}

type OnlineStatus {
  userId: ID!
  isOnline: Boolean!
  lastSeen: String
}

type LikeCountUpdate {
  postId: ID!
  count: Int!
  delta: Int!
}
```

---

## 📌 Step 155: PubSub คืออะไร

**PubSub (Publish-Subscribe)** คือ messaging pattern:

```
Publisher ──publish(event)──→ PubSub ──→ Subscriber 1
                                    └──→ Subscriber 2
                                    └──→ Subscriber 3

ตัวอย่าง:
- User สร้าง comment (publish "NEW_COMMENT")
- Subscribers ทุกคนที่ subscribe postId นั้น ได้รับ comment
```

### In-memory PubSub (Development)

```javascript
import { PubSub } from 'graphql-subscriptions';

export const pubsub = new PubSub();

// Events
export const EVENTS = {
  USER_CREATED: 'USER_CREATED',
  USER_UPDATED: 'USER_UPDATED',
  USER_DELETED: 'USER_DELETED',
  POST_CREATED: 'POST_CREATED',
  POST_UPDATED: 'POST_UPDATED',
  POST_PUBLISHED: 'POST_PUBLISHED',
  COMMENT_ADDED: 'COMMENT_ADDED',
  MESSAGE_SENT: 'MESSAGE_SENT',
  NOTIFICATION_RECEIVED: 'NOTIFICATION_RECEIVED',
  USER_ONLINE_STATUS: 'USER_ONLINE_STATUS',
  POST_LIKE_COUNT: 'POST_LIKE_COUNT',
};
```

> **⚠️ Warning:** In-memory PubSub ไม่เหมาะ production — ต้องใช้ Redis PubSub (เรียนใน Part 032)

---

## 📌 Step 156: Subscription Resolvers

```javascript
// src/resolvers/subscription.js
import { pubsub, EVENTS } from '../pubsub.js';
import { withFilter } from 'graphql-subscriptions';

export const subscriptionResolvers = {
  Subscription: {
    // User Events
    userCreated: {
      subscribe: () => pubsub.asyncIterator([EVENTS.USER_CREATED]),
      resolve: (payload) => payload.user,
    },

    userUpdated: {
      // withFilter ช่วย filter events ตาม id
      subscribe: withFilter(
        () => pubsub.asyncIterator([EVENTS.USER_UPDATED]),
        (payload, variables) => {
          // ถ้าไม่ระบุ id ได้ทุก user update
          if (!variables.id) return true;
          return payload.user.id === variables.id;
        }
      ),
      resolve: (payload) => payload.user,
    },

    // Post Events
    postCreated: {
      subscribe: () => pubsub.asyncIterator([EVENTS.POST_CREATED]),
      resolve: (payload) => payload.post,
    },

    postPublished: {
      subscribe: () => pubsub.asyncIterator([EVENTS.POST_PUBLISHED]),
      resolve: (payload) => payload.post,
    },

    // Comment Events
    commentAdded: {
      subscribe: withFilter(
        () => pubsub.asyncIterator([EVENTS.COMMENT_ADDED]),
        (payload, variables) => {
          return payload.comment.postId === variables.postId;
        }
      ),
      resolve: (payload) => payload.comment,
    },

    // Chat
    messageSent: {
      subscribe: withFilter(
        () => pubsub.asyncIterator([EVENTS.MESSAGE_SENT]),
        (payload, variables) => {
          return payload.message.roomId === variables.roomId;
        }
      ),
      resolve: (payload) => payload.message,
    },

    // Notifications (เฉพาะ user นั้น)
    notificationReceived: {
      subscribe: withFilter(
        () => pubsub.asyncIterator([EVENTS.NOTIFICATION_RECEIVED]),
        (payload, variables, context) => {
          // ตรวจสอบว่า notification นี้เป็นของ user คนนี้
          return payload.notification.userId === variables.userId;
        }
      ),
      resolve: (payload) => payload.notification,
    },

    // Like count
    postLikeCount: {
      subscribe: withFilter(
        () => pubsub.asyncIterator([EVENTS.POST_LIKE_COUNT]),
        (payload, variables) => {
          return payload.update.postId === variables.postId;
        }
      ),
      resolve: (payload) => payload.update,
    },
  },
};
```

---

## 📌 Step 157: Publish Events จาก Mutations

```javascript
// src/resolvers/mutation.js — เพิ่ม publish

createUser: async (_, { input }) => {
  const user = { /* create user */ };
  db.users.push(user);
  
  // Publish event
  await pubsub.publish(EVENTS.USER_CREATED, { user });
  
  return user;
},

updateUser: async (_, { id, input }) => {
  const user = { /* update user */ };
  
  // Publish event
  await pubsub.publish(EVENTS.USER_UPDATED, { user });
  
  return user;
},

createComment: async (_, { postId, text }, context) => {
  const comment = { /* create comment */ };
  db.comments.push(comment);
  
  // Publish comment event
  await pubsub.publish(EVENTS.COMMENT_ADDED, { comment });
  
  // Publish notification ให้ post author
  const post = db.posts.find(p => p.id === postId);
  if (post && post.authorId !== context.currentUser?.id) {
    await pubsub.publish(EVENTS.NOTIFICATION_RECEIVED, {
      notification: {
        id: String(Date.now()),
        userId: post.authorId,
        type: 'COMMENT_ON_POST',
        title: 'มีความคิดเห็นใหม่',
        body: `${context.currentUser?.name} แสดงความคิดเห็นในโพสต์ของคุณ`,
        isRead: false,
        createdAt: new Date().toISOString(),
      },
    });
  }
  
  return comment;
},

likePost: async (_, { postId }, context) => {
  const post = db.posts.find(p => p.id === postId);
  post.likeCount = (post.likeCount || 0) + 1;
  
  // Publish like count update
  await pubsub.publish(EVENTS.POST_LIKE_COUNT, {
    update: {
      postId,
      count: post.likeCount,
      delta: 1,
    },
  });
  
  return post;
},
```

---

## 📌 Step 158: Setup WebSocket Server

```javascript
// src/index.js — เพิ่ม WebSocket support

import { createServer } from 'http';
import { WebSocketServer } from 'ws';
import { useServer } from 'graphql-ws/lib/use/ws';
import { makeExecutableSchema } from '@graphql-tools/schema';
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import { ApolloServerPluginDrainHttpServer } from '@apollo/server/plugin/drainHttpServer';
import express from 'express';
import cors from 'cors';
import { json } from 'body-parser';
import { typeDefs } from './schema/index.js';
import { resolvers } from './resolvers/index.js';
import { createContext } from './context.js';

const app = express();
const httpServer = createServer(app);

// สร้าง schema ที่ใช้ร่วมกัน
const schema = makeExecutableSchema({ typeDefs, resolvers });

// สร้าง WebSocket server สำหรับ subscriptions
const wsServer = new WebSocketServer({
  server: httpServer,
  path: '/graphql',
});

// Cleanup function
const serverCleanup = useServer(
  {
    schema,
    // Context สำหรับ subscriptions
    context: async (ctx) => {
      // Extract token จาก connection params
      const token = ctx.connectionParams?.authorization;
      const currentUser = token ? verifyToken(token) : null;
      
      return {
        currentUser,
        pubsub,
      };
    },
    // เมื่อ client connect
    onConnect: async (ctx) => {
      console.log('Client connected:', ctx.connectionParams);
      return true; // อนุญาต connection
    },
    // เมื่อ client disconnect
    onDisconnect: async (ctx) => {
      console.log('Client disconnected');
    },
  },
  wsServer
);

// สร้าง Apollo Server
const server = new ApolloServer({
  schema,
  plugins: [
    ApolloServerPluginDrainHttpServer({ httpServer }),
    // ปิด WebSocket server เมื่อ Apollo Server หยุด
    {
      async serverWillStart() {
        return {
          async drainServer() {
            await serverCleanup.dispose();
          },
        };
      },
    },
  ],
});

await server.start();

app.use(
  '/graphql',
  cors(),
  json(),
  expressMiddleware(server, { context: createContext })
);

const PORT = 4000;
httpServer.listen(PORT, () => {
  console.log(`🚀 HTTP Server: http://localhost:${PORT}/graphql`);
  console.log(`🔌 WebSocket: ws://localhost:${PORT}/graphql`);
});
```

---

## 📌 Step 159: Chat Application

### Schema

```graphql
type Room {
  id: ID!
  name: String!
  description: String
  members: [User!]!
  messages(last: Int = 50): [Message!]!
  createdAt: String!
}

type Message {
  id: ID!
  text: String!
  author: User!
  room: Room!
  type: MessageType!
  createdAt: String!
}

enum MessageType {
  TEXT
  IMAGE
  FILE
  SYSTEM
}

type Query {
  rooms: [Room!]!
  room(id: ID!): Room
  messages(roomId: ID!, limit: Int = 50, before: ID): [Message!]!
}

type Mutation {
  createRoom(name: String!, description: String): Room!
  joinRoom(roomId: ID!): Room!
  leaveRoom(roomId: ID!): Boolean!
  sendMessage(roomId: ID!, text: String!): Message!
  deleteMessage(id: ID!): Boolean!
}

type Subscription {
  messageSent(roomId: ID!): Message!
  userJoinedRoom(roomId: ID!): User!
  userLeftRoom(roomId: ID!): User!
  messageDeleted(roomId: ID!): ID!
}
```

### Data

```javascript
const db = {
  rooms: [
    {
      id: '1',
      name: 'ทั่วไป',
      description: 'ห้องสนทนาทั่วไป',
      memberIds: ['1', '2', '3'],
      createdAt: '2024-01-01T00:00:00Z',
    },
    {
      id: '2',
      name: 'GraphQL',
      description: 'พูดคุยเรื่อง GraphQL',
      memberIds: ['1', '2'],
      createdAt: '2024-01-02T00:00:00Z',
    },
  ],
  messages: [
    {
      id: '1',
      text: 'สวัสดีทุกคน!',
      authorId: '1',
      roomId: '1',
      type: 'TEXT',
      createdAt: '2024-01-01T10:00:00Z',
    },
    {
      id: '2',
      text: 'สวัสดีครับ',
      authorId: '2',
      roomId: '1',
      type: 'TEXT',
      createdAt: '2024-01-01T10:01:00Z',
    },
  ],
};
```

### Resolvers

```javascript
// Chat resolvers
const chatResolvers = {
  Query: {
    rooms: () => db.rooms,
    room: (_, { id }) => db.rooms.find(r => r.id === id),
    messages: (_, { roomId, limit, before }) => {
      let msgs = db.messages.filter(m => m.roomId === roomId);
      if (before) {
        const idx = msgs.findIndex(m => m.id === before);
        if (idx > 0) msgs = msgs.slice(0, idx);
      }
      return msgs.slice(-limit);
    },
  },

  Mutation: {
    createRoom: (_, { name, description }, context) => {
      const room = {
        id: String(Date.now()),
        name,
        description,
        memberIds: [context.currentUser.id],
        createdAt: new Date().toISOString(),
      };
      db.rooms.push(room);
      return room;
    },

    joinRoom: (_, { roomId }, context) => {
      const room = db.rooms.find(r => r.id === roomId);
      if (!room) throw new Error('ไม่พบห้อง');
      
      if (!room.memberIds.includes(context.currentUser.id)) {
        room.memberIds.push(context.currentUser.id);
        
        // Notify others
        pubsub.publish(EVENTS.USER_JOINED_ROOM, {
          roomId,
          user: db.users.find(u => u.id === context.currentUser.id),
        });
      }
      
      return room;
    },

    sendMessage: async (_, { roomId, text }, context) => {
      if (!context.currentUser) throw new Error('กรุณา Login');
      
      const room = db.rooms.find(r => r.id === roomId);
      if (!room) throw new Error('ไม่พบห้อง');
      if (!room.memberIds.includes(context.currentUser.id)) {
        throw new Error('คุณไม่ได้เป็นสมาชิกห้องนี้');
      }
      
      const message = {
        id: String(Date.now()),
        text,
        authorId: context.currentUser.id,
        roomId,
        type: 'TEXT',
        createdAt: new Date().toISOString(),
      };
      
      db.messages.push(message);
      
      // Publish
      await pubsub.publish(EVENTS.MESSAGE_SENT, { message });
      
      return message;
    },
  },

  Subscription: {
    messageSent: {
      subscribe: withFilter(
        () => pubsub.asyncIterator([EVENTS.MESSAGE_SENT]),
        (payload, variables) => payload.message.roomId === variables.roomId
      ),
      resolve: (payload) => payload.message,
    },
    
    userJoinedRoom: {
      subscribe: withFilter(
        () => pubsub.asyncIterator([EVENTS.USER_JOINED_ROOM]),
        (payload, variables) => payload.roomId === variables.roomId
      ),
      resolve: (payload) => payload.user,
    },
  },

  Room: {
    members: (room) => db.users.filter(u => room.memberIds.includes(u.id)),
    messages: (room, { last = 50 }) => 
      db.messages.filter(m => m.roomId === room.id).slice(-last),
  },

  Message: {
    author: (msg) => db.users.find(u => u.id === msg.authorId),
    room: (msg) => db.rooms.find(r => r.id === msg.roomId),
  },
};
```

---

## 📌 Step 160: Client-side Subscription

### ด้วย Apollo Client

```javascript
import { ApolloClient, InMemoryCache, split, HttpLink, gql, useSubscription } from '@apollo/client';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { createClient } from 'graphql-ws';
import { getMainDefinition } from '@apollo/client/utilities';

// HTTP Link สำหรับ queries/mutations
const httpLink = new HttpLink({
  uri: 'http://localhost:4000/graphql',
});

// WebSocket Link สำหรับ subscriptions
const wsLink = new GraphQLWsLink(createClient({
  url: 'ws://localhost:4000/graphql',
  connectionParams: {
    authorization: localStorage.getItem('token'),
  },
}));

// แยกใช้ transport ตาม operation type
const splitLink = split(
  ({ query }) => {
    const definition = getMainDefinition(query);
    return (
      definition.kind === 'OperationDefinition' &&
      definition.operation === 'subscription'
    );
  },
  wsLink,   // ← subscriptions ไปทาง WebSocket
  httpLink, // ← queries/mutations ไปทาง HTTP
);

const client = new ApolloClient({
  link: splitLink,
  cache: new InMemoryCache(),
});
```

### React Component สำหรับ Chat

```javascript
const MESSAGE_SENT = gql`
  subscription MessageSent($roomId: ID!) {
    messageSent(roomId: $roomId) {
      id
      text
      author {
        id
        name
      }
      createdAt
    }
  }
`;

const GET_MESSAGES = gql`
  query GetMessages($roomId: ID!) {
    messages(roomId: $roomId) {
      id
      text
      author {
        id
        name
      }
      createdAt
    }
  }
`;

const SEND_MESSAGE = gql`
  mutation SendMessage($roomId: ID!, $text: String!) {
    sendMessage(roomId: $roomId, text: $text) {
      id
      text
      createdAt
    }
  }
`;

function ChatRoom({ roomId }) {
  const [text, setText] = React.useState('');
  
  // โหลดข้อความเดิม
  const { data: messagesData } = useQuery(GET_MESSAGES, {
    variables: { roomId },
  });
  
  // Subscribe รับข้อความใหม่
  useSubscription(MESSAGE_SENT, {
    variables: { roomId },
    onData: ({ client, data }) => {
      const newMessage = data.data.messageSent;
      
      // อัพเดท cache
      client.cache.modify({
        fields: {
          messages: (existing = []) => {
            const newRef = client.cache.writeFragment({
              data: newMessage,
              fragment: gql`
                fragment NewMessage on Message {
                  id text author { id name } createdAt
                }
              `,
            });
            return [...existing, newRef];
          },
        },
      });
    },
  });
  
  const [sendMessage] = useMutation(SEND_MESSAGE);
  
  const handleSend = async (e) => {
    e.preventDefault();
    if (!text.trim()) return;
    
    await sendMessage({ variables: { roomId, text } });
    setText('');
  };
  
  return (
    <div>
      <div className="messages">
        {messagesData?.messages.map(msg => (
          <div key={msg.id}>
            <strong>{msg.author.name}:</strong> {msg.text}
          </div>
        ))}
      </div>
      <form onSubmit={handleSend}>
        <input value={text} onChange={e => setText(e.target.value)} />
        <button type="submit">ส่ง</button>
      </form>
    </div>
  );
}
```

---

## 📌 Step 161: Subscription Authentication

```javascript
// WebSocket server context
const serverCleanup = useServer(
  {
    schema,
    context: async (ctx) => {
      const token = ctx.connectionParams?.authorization?.replace('Bearer ', '');
      
      if (!token) {
        // ไม่มี token — อาจเป็น guest
        return { currentUser: null };
      }
      
      try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        const user = db.users.find(u => u.id === decoded.userId);
        return { currentUser: user };
      } catch (err) {
        // Token invalid
        throw new Error('Token ไม่ถูกต้อง');
      }
    },
    
    onConnect: async (ctx) => {
      const token = ctx.connectionParams?.authorization?.replace('Bearer ', '');
      if (!token) return true; // อนุญาต guest
      
      try {
        jwt.verify(token, process.env.JWT_SECRET);
        console.log('Authenticated subscription connection');
        return true;
      } catch {
        console.warn('Rejected unauthenticated subscription');
        return false; // ปฏิเสธ
      }
    },
  },
  wsServer
);
```

### ใน Subscription Resolver

```javascript
notificationReceived: {
  subscribe: withFilter(
    () => pubsub.asyncIterator([EVENTS.NOTIFICATION_RECEIVED]),
    (payload, variables, context) => {
      // ต้อง login
      if (!context.currentUser) return false;
      
      // เฉพาะ notification ของตัวเอง
      return payload.notification.userId === context.currentUser.id;
    }
  ),
  resolve: (payload) => payload.notification,
},
```

---

## 📌 Step 162: Redis PubSub (Production)

```bash
npm install ioredis graphql-redis-subscriptions
```

```javascript
// src/pubsub.js
import { RedisPubSub } from 'graphql-redis-subscriptions';
import Redis from 'ioredis';

const options = {
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
  retryStrategy: (times) => Math.min(times * 50, 2000),
};

export const pubsub = new RedisPubSub({
  publisher: new Redis(options),
  subscriber: new Redis(options),
});
```

### ข้อดีของ Redis PubSub

```
In-memory PubSub:
❌ ไม่ scale (เฉพาะ 1 server)
❌ ข้อมูลหายเมื่อ server restart
❌ ไม่รองรับ horizontal scaling

Redis PubSub:
✅ Scale ได้ (หลาย servers)
✅ Persistent
✅ รองรับ horizontal scaling
✅ เร็วมาก (in-memory database)
```

---

## 📌 Step 163: Subscription Best Practices

```javascript
// 1. จำกัด subscriptions ต่อ client
const MAX_SUBSCRIPTIONS_PER_CLIENT = 10;
const clientSubscriptions = new Map();

onConnect: async (ctx) => {
  const clientId = ctx.connectionParams?.clientId || 'anonymous';
  const count = clientSubscriptions.get(clientId) || 0;
  
  if (count >= MAX_SUBSCRIPTIONS_PER_CLIENT) {
    throw new Error('Too many subscriptions');
  }
  
  clientSubscriptions.set(clientId, count + 1);
  return { clientId };
},

onDisconnect: (ctx) => {
  const { clientId } = ctx.extra;
  if (clientId) {
    const count = clientSubscriptions.get(clientId) || 1;
    clientSubscriptions.set(clientId, count - 1);
  }
},

// 2. Debounce high-frequency events
import { Subject, debounceTime } from 'rxjs';

const likeSubject = new Subject();
likeSubject.pipe(debounceTime(100)).subscribe(update => {
  pubsub.publish(EVENTS.POST_LIKE_COUNT, { update });
});

// 3. เพิ่ม keepalive
const serverCleanup = useServer({
  schema,
  context,
  // ping client ทุก 30 วินาที
  keepAlive: 30000,
}, wsServer);

// 4. Error handling ใน subscription
messageSent: {
  subscribe: async function* (_, { roomId }, context) {
    try {
      const iterator = pubsub.asyncIterator(EVENTS.MESSAGE_SENT);
      for await (const event of iterator) {
        if (event.message.roomId === roomId) {
          yield event;
        }
      }
    } catch (error) {
      console.error('Subscription error:', error);
      throw error;
    }
  },
  resolve: (payload) => payload.message,
},
```

---

## 📌 Step 164: Testing Subscriptions

```javascript
// test/subscription.test.js
import { createTestClient } from 'apollo-testing-client';
import { ApolloServer } from '@apollo/server';
import { makeExecutableSchema } from '@graphql-tools/schema';

describe('Chat Subscriptions', () => {
  let server;
  
  beforeAll(async () => {
    const schema = makeExecutableSchema({ typeDefs, resolvers });
    server = new ApolloServer({ schema });
    await server.start();
  });

  it('should receive message when sent', async () => {
    const messages = [];
    
    // Subscribe
    const subscription = server.subscribe({
      query: `
        subscription {
          messageSent(roomId: "1") {
            id
            text
          }
        }
      `,
    });
    
    // รอ events
    const collectEvents = async () => {
      for await (const event of subscription) {
        messages.push(event.data?.messageSent);
        if (messages.length >= 1) break;
      }
    };
    
    const collectPromise = collectEvents();
    
    // Trigger mutation
    await server.executeOperation({
      query: `
        mutation {
          sendMessage(roomId: "1", text: "Hello") {
            id
          }
        }
      `,
    });
    
    await collectPromise;
    
    expect(messages).toHaveLength(1);
    expect(messages[0].text).toBe('Hello');
  });
});
```

---

## 📌 Step 165: Live Notifications System

```graphql
# Complete notification schema
type Notification {
  id: ID!
  recipient: User!
  sender: User
  type: NotificationType!
  title: String!
  body: String!
  link: String
  isRead: Boolean!
  readAt: String
  createdAt: String!
}

enum NotificationType {
  COMMENT_ON_POST      # มีคนคอมเมนต์โพสต์ของคุณ
  LIKE_POST            # มีคนกด like โพสต์ของคุณ
  REPLY_TO_COMMENT     # มีคนตอบกลับคอมเมนต์ของคุณ
  NEW_FOLLOWER         # มีคน follow คุณ
  POST_MENTION         # มีคนแท็กคุณในโพสต์
  SYSTEM               # ข้อความจากระบบ
}

type Query {
  notifications(unreadOnly: Boolean): [Notification!]!
  unreadNotificationCount: Int!
}

type Mutation {
  markNotificationRead(id: ID!): Notification!
  markAllNotificationsRead: Int!
  deleteNotification(id: ID!): Boolean!
}

type Subscription {
  notificationReceived: Notification!
  notificationCountChanged: Int!
}
```

```javascript
// Notification service
export class NotificationService {
  async send({ recipientId, senderId, type, title, body, link }) {
    const notification = {
      id: String(Date.now()),
      recipientId,
      senderId,
      type,
      title,
      body,
      link,
      isRead: false,
      readAt: null,
      createdAt: new Date().toISOString(),
    };
    
    db.notifications.push(notification);
    
    // Publish real-time
    await pubsub.publish(EVENTS.NOTIFICATION_RECEIVED, {
      recipientId,
      notification,
    });
    
    return notification;
  }
}
```

---

## 📌 Step 166: สรุป Part 007

### Subscription Cheatsheet

```graphql
# Schema
type Subscription {
  eventName: ReturnType!
  filteredEvent(id: ID!): ReturnType!
}

# Client
subscription {
  eventName {
    field1
    field2
  }
}
```

```javascript
// Resolver
eventName: {
  subscribe: () => pubsub.asyncIterator(['EVENT_NAME']),
  resolve: (payload) => payload.data,
},

// Filtered
filteredEvent: {
  subscribe: withFilter(
    () => pubsub.asyncIterator(['EVENT_NAME']),
    (payload, variables) => payload.data.id === variables.id
  ),
  resolve: (payload) => payload.data,
},

// Publish
await pubsub.publish('EVENT_NAME', { data: result });
```

### เนื้อหาที่เรียนรู้

✅ Subscription คืออะไรและ Use Cases  
✅ WebSocket protocol  
✅ PubSub pattern  
✅ withFilter สำหรับ filtering  
✅ Chat application จริง  
✅ Authentication ใน subscriptions  
✅ Redis PubSub สำหรับ production  

### Homework

1. สร้าง "typing indicator" subscription (`isTyping(roomId: ID!): TypingStatus!`)
2. เพิ่ม online/offline status สำหรับ users
3. สร้าง real-time counter ที่อัพเดทเมื่อมี user ใหม่

### ในส่วนถัดไป

➡️ **[Part 008](./part-008.md)** — Resolvers: ทำความเข้าใจแบบลึก

---

*Part 007 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
