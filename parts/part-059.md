# Part 059 — Real-Time Collaboration 🤝

> **ระดับ:** Expert | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1271–1310

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Collaborative editing patterns
- Operational Transformation (OT) basics
- CRDTs (Conflict-free Replicated Data Types) overview
- Presence indicators (who's online/editing)
- Cursor sharing
- GraphQL subscriptions สำหรับ collaboration
- Yjs + WebSocket integration
- Optimistic locking conflicts

---

## 📌 Step 1271: Collaboration Architecture

```
Real-time collaboration ต้องจัดการ:
1. Concurrent edits: 2 คนแก้พร้อมกัน
2. Conflict resolution: ใครถูก?
3. Presence: ใครออนไลน์/กำลังแก้ที่ไหน
4. Offline: แก้ขณะ offline แล้ว sync เมื่อ reconnect

Solutions:
- Operational Transformation (OT): Google Docs approach
- CRDTs: Distributed approach (Yjs, Automerge)
- Last-Write-Wins: ง่าย แต่ conflict ได้
- Optimistic Locking: lock-based, ง่ายกว่า

GraphQL subscriptions ใช้สำหรับ:
- Broadcast changes ไปยัง collaborators
- Presence updates
- Cursor positions
```

---

## 📌 Step 1272: Presence System

```graphql
# src/schema/collaboration.graphql

type Subscription {
  # Presence: ใครกำลัง view/edit document
  documentPresence(documentId: ID!): PresenceUpdate!
  
  # Document changes
  documentChanged(documentId: ID!): DocumentChange!
  
  # Cursor positions
  cursorMoved(documentId: ID!): CursorPosition!
  
  # Comment/annotation updates
  annotationsUpdated(documentId: ID!): AnnotationUpdate!
}

type Mutation {
  # Join/leave collaboration session
  joinDocument(documentId: ID!): CollabSession!
  leaveDocument(documentId: ID!): Boolean!
  
  # Update cursor position
  updateCursor(documentId: ID!, position: CursorInput!): Boolean!
  
  # Submit document change
  submitChange(documentId: ID!, change: ChangeInput!): ChangeResult!
}

type PresenceUpdate {
  documentId: ID!
  users: [CollabUser!]!
  joined: CollabUser
  left: CollabUser
}

type CollabUser {
  userId: ID!
  name: String!
  avatar: String
  color: String!     # unique color สำหรับ cursor
  cursor: CursorPosition
  lastSeen: DateTime!
}

type CursorPosition {
  userId: ID!
  anchor: Int!    # character offset
  head: Int!      # selection end
  line: Int!
  column: Int!
}

type DocumentChange {
  documentId: ID!
  version: Int!
  changes: [Operation!]!
  userId: ID!
  timestamp: DateTime!
}

type Operation {
  type: OperationType!
  position: Int!
  text: String
  length: Int
}

enum OperationType {
  INSERT
  DELETE
  RETAIN
}
```

---

## 📌 Step 1273: Presence Service

```typescript
// src/collaboration/presence.ts

interface UserPresence {
  userId: string;
  documentId: string;
  name: string;
  avatar?: string;
  color: string;
  cursor?: { anchor: number; head: number; line: number; column: number };
  lastActivity: Date;
}

const PRESENCE_KEY = (documentId: string) => `presence:${documentId}`;
const PRESENCE_TTL = 30; // seconds

// Color palette for collaborators
const COLORS = [
  '#FF6B6B', '#4ECDC4', '#45B7D1', '#96CEB4', '#FFEAA7',
  '#DDA0DD', '#98D8C8', '#F7DC6F', '#BB8FCE', '#85C1E9',
];

export class PresenceService {
  constructor(
    private redis: Redis,
    private pubsub: RedisPubSub
  ) {}
  
  async join(userId: string, documentId: string, userInfo: { name: string; avatar?: string }) {
    // Assign color
    const existingUsers = await this.getPresence(documentId);
    const usedColors = existingUsers.map(u => u.color);
    const color = COLORS.find(c => !usedColors.includes(c)) ?? COLORS[0]!;
    
    const presence: UserPresence = {
      userId,
      documentId,
      name: userInfo.name,
      avatar: userInfo.avatar,
      color,
      lastActivity: new Date(),
    };
    
    // Store in Redis hash with TTL
    const key = PRESENCE_KEY(documentId);
    await this.redis.hset(key, userId, JSON.stringify(presence));
    await this.redis.expire(key, 60 * 60); // 1 hour TTL for the hash
    
    // Update user-specific TTL with sorted set
    await this.redis.zadd(
      `presence:ttl:${documentId}`,
      Date.now() + PRESENCE_TTL * 1000,
      userId
    );
    
    // Publish presence update
    await this.pubsub.publish(`presence:${documentId}`, {
      documentId,
      users: await this.getPresence(documentId),
      joined: presence,
      left: null,
    });
    
    return presence;
  }
  
  async leave(userId: string, documentId: string) {
    const key = PRESENCE_KEY(documentId);
    await this.redis.hdel(key, userId);
    await this.redis.zrem(`presence:ttl:${documentId}`, userId);
    
    const users = await this.getPresence(documentId);
    await this.pubsub.publish(`presence:${documentId}`, {
      documentId,
      users,
      joined: null,
      left: { userId },
    });
  }
  
  async updateCursor(
    userId: string,
    documentId: string,
    cursor: UserPresence['cursor']
  ) {
    const key = PRESENCE_KEY(documentId);
    const raw = await this.redis.hget(key, userId);
    if (!raw) return;
    
    const presence = JSON.parse(raw) as UserPresence;
    presence.cursor = cursor;
    presence.lastActivity = new Date();
    
    await this.redis.hset(key, userId, JSON.stringify(presence));
    
    // Publish cursor update
    await this.pubsub.publish(`cursor:${documentId}`, {
      userId,
      ...cursor,
    });
  }
  
  async getPresence(documentId: string): Promise<UserPresence[]> {
    const key = PRESENCE_KEY(documentId);
    const hash = await this.redis.hgetall(key);
    if (!hash) return [];
    
    // Filter expired users
    const now = Date.now();
    const ttlKey = `presence:ttl:${documentId}`;
    const expired = await this.redis.zrangebyscore(ttlKey, '-inf', now);
    
    // Remove expired
    if (expired.length) {
      await this.redis.hdel(key, ...expired);
      await this.redis.zremrangebyscore(ttlKey, '-inf', now);
    }
    
    return Object.values(hash)
      .map(v => JSON.parse(v) as UserPresence)
      .filter(u => !expired.includes(u.userId));
  }
  
  // Heartbeat — client ส่งทุก 10 วินาที
  async heartbeat(userId: string, documentId: string) {
    await this.redis.zadd(
      `presence:ttl:${documentId}`,
      Date.now() + PRESENCE_TTL * 1000,
      userId
    );
  }
}
```

---

## 📌 Step 1274: Operational Transformation

```typescript
// src/collaboration/ot.ts
// Simple Operational Transformation implementation

type Operation =
  | { type: 'insert'; position: number; text: string }
  | { type: 'delete'; position: number; length: number }
  | { type: 'retain'; length: number };

// Apply operation to text
export function applyOperation(text: string, op: Operation): string {
  switch (op.type) {
    case 'insert':
      return text.slice(0, op.position) + op.text + text.slice(op.position);
    case 'delete':
      return text.slice(0, op.position) + text.slice(op.position + op.length);
    case 'retain':
      return text;
  }
}

// Transform op1 against op2 (concurrent ops)
// Returns transformed op1 that can be applied after op2
export function transform(op1: Operation, op2: Operation): Operation {
  if (op1.type === 'insert' && op2.type === 'insert') {
    if (op2.position <= op1.position) {
      return { ...op1, position: op1.position + op2.text.length };
    }
    return op1;
  }
  
  if (op1.type === 'insert' && op2.type === 'delete') {
    if (op2.position < op1.position) {
      const newPos = Math.max(op2.position, op1.position - op2.length);
      return { ...op1, position: newPos };
    }
    return op1;
  }
  
  if (op1.type === 'delete' && op2.type === 'insert') {
    if (op2.position <= op1.position) {
      return { ...op1, position: op1.position + op2.text.length };
    }
    return op1;
  }
  
  if (op1.type === 'delete' && op2.type === 'delete') {
    if (op2.position + op2.length <= op1.position) {
      return { ...op1, position: op1.position - op2.length };
    }
    if (op2.position >= op1.position + op1.length) {
      return op1;
    }
    // Overlapping deletes — truncate
    const newLength = Math.max(0,
      op1.length - Math.min(op1.position + op1.length, op2.position + op2.length) +
      Math.max(op1.position, op2.position)
    );
    return { ...op1, length: newLength };
  }
  
  return op1;
}

// Document service: versioned changes
export class DocumentService {
  async submitChange(
    documentId: string,
    userId: string,
    clientVersion: number,
    operations: Operation[]
  ) {
    return this.prisma.$transaction(async (tx) => {
      const document = await tx.document.findUniqueOrThrow({
        where: { id: documentId },
        select: { version: true, content: true },
      });
      
      // ตรวจสอบ version conflict
      if (clientVersion > document.version) {
        throw new GraphQLError('Client version is ahead of server');
      }
      
      // Transform operations ถ้า client ส่งมาด้วย old version
      let finalOps = operations;
      if (clientVersion < document.version) {
        // Get missing operations
        const missingOps = await tx.documentOp.findMany({
          where: { documentId, version: { gt: clientVersion } },
          orderBy: { version: 'asc' },
        });
        
        // Transform against missing ops
        for (const missedOp of missingOps) {
          const serverOps = missedOp.operations as Operation[];
          finalOps = finalOps.map(op =>
            serverOps.reduce((transformed, serverOp) => transform(transformed, serverOp), op)
          );
        }
      }
      
      // Apply operations
      let content = document.content;
      for (const op of finalOps) {
        content = applyOperation(content, op);
      }
      
      const newVersion = document.version + 1;
      
      // Save
      await tx.document.update({
        where: { id: documentId },
        data: { content, version: newVersion },
      });
      
      await tx.documentOp.create({
        data: { documentId, userId, version: newVersion, operations: finalOps },
      });
      
      // Publish to collaborators
      await pubsub.publish(`doc:${documentId}`, {
        documentId,
        version: newVersion,
        changes: finalOps,
        userId,
        timestamp: new Date(),
      });
      
      return { version: newVersion, applied: finalOps };
    });
  }
}
```

---

## 📌 Step 1275: Yjs Integration (Production-Grade)

```bash
npm install yjs y-websocket y-protocols
```

```typescript
// src/collaboration/yjs-server.ts
// Yjs เป็น CRDT library ที่ production-ready

import { WebSocketServer } from 'ws';
import { setupWSConnection } from 'y-websocket/bin/utils';
import { LeveldbPersistence } from 'y-leveldb';

const persistence = new LeveldbPersistence('./collab-data');

const wss = new WebSocketServer({ port: 4001 });

wss.on('connection', async (conn, req) => {
  // Extract document ID and auth from URL
  const url = new URL(req.url ?? '', 'http://localhost');
  const documentId = url.searchParams.get('documentId');
  const token = url.searchParams.get('token');
  
  // Verify auth
  try {
    const user = verifyToken(token ?? '');
    
    // Check document access
    const hasAccess = await checkDocumentAccess(user.sub, documentId!);
    if (!hasAccess) {
      conn.close(4001, 'Unauthorized');
      return;
    }
    
    setupWSConnection(conn, req, {
      docName: documentId!,
      gc: true,
      
      // Persistence: save to LevelDB
      async onConnect(conn, yjsDoc) {
        // Load persisted state
        const persistedDoc = await persistence.getYDoc(documentId!);
        if (persistedDoc) {
          Yjs.applyUpdate(yjsDoc, persistedDoc);
        }
        
        // Update presence
        await presenceService.join(user.sub, documentId!, { name: user.name });
      },
      
      async onDisconnect(conn) {
        await presenceService.leave(user.sub, documentId!);
      },
    });
    
    // Save document changes
    yjsDoc.on('update', async (update: Uint8Array) => {
      await persistence.storeUpdate(documentId!, update);
    });
    
  } catch {
    conn.close(4001, 'Unauthorized');
  }
});

// GraphQL resolver: get document content
const documentResolver = async (_, { id }, ctx) => {
  // Yjs ใช้ own WebSocket, แต่ metadata ยังอยู่ใน GraphQL
  const doc = await ctx.prisma.document.findUnique({ where: { id } });
  const users = await presenceService.getPresence(id);
  
  return { ...doc, activeCollaborators: users };
};
```

---

## 📌 Step 1276: สรุป Part 059

### เนื้อหาที่เรียนรู้

✅ Collaboration architecture patterns  
✅ Presence system ด้วย Redis  
✅ Operational Transformation basics  
✅ Yjs CRDT integration  
✅ Cursor sharing  
✅ Conflict resolution strategies  

### Approach เปรียบเทียบ

```
Last-Write-Wins:
✅ ง่ายมาก
❌ สูญหายข้อมูลเมื่อ conflict

Optimistic Locking:
✅ ป้องกัน conflicts
❌ ต้อง retry เมื่อ version mismatch

Operational Transformation (OT):
✅ Google Docs approach
✅ แก้พร้อมกันได้ทุกที่
❌ ซับซ้อน, bugs ง่าย

CRDTs (Yjs, Automerge):
✅ Mathematically correct
✅ Production-ready libraries
✅ Offline support ดีเยี่ยม
❌ ขนาด document บวมได้
```

### ในส่วนถัดไป

➡️ **[Part 060](./part-060.md)** — AI/ML Integration กับ GraphQL

---

*Part 059 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
