# Part 037 — Advanced Batching & DataLoader Patterns 📦

> **ระดับ:** Advanced | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 531–570

---

## 🎯 สิ่งที่จะได้เรียนรู้

- DataLoader internals
- Batching strategies
- Cache invalidation
- DataLoader สำหรับ relations
- Multi-key DataLoader
- DataLoader กับ Prisma
- DataLoader testing
- Performance monitoring

---

## 📌 Step 531: DataLoader Internals

```javascript
// DataLoader ทำงานยังไง?
// 1. Collect IDs ทุกตัวในระหว่าง execution tick เดียว
// 2. รอ microtask queue หมด
// 3. เรียก batch function ครั้งเดียว พร้อมทุก IDs
// 4. Map results กลับ

import DataLoader from 'dataloader';

// Batch function: รับ array ของ IDs → return array ของ results
const userLoader = new DataLoader(async (userIds) => {
  console.log(`Loading ${userIds.length} users in one query`);
  
  const users = await prisma.user.findMany({
    where: { id: { in: userIds } },
  });
  
  // สำคัญ! ต้อง return ตามลำดับ IDs เดิม
  return userIds.map(id =>
    users.find(u => u.id === id) || new Error(`User ${id} not found`)
  );
});

// ใช้งาน: load ทีละ ID แต่ batch อัตโนมัติ
const [user1, user2, user3] = await Promise.all([
  userLoader.load('id-1'),
  userLoader.load('id-2'),
  userLoader.load('id-3'),
]);
// → Query เดียว: SELECT * FROM users WHERE id IN ('id-1', 'id-2', 'id-3')
```

---

## 📌 Step 532: DataLoader Factory สำหรับ Prisma

```javascript
// src/loaders/index.js
import DataLoader from 'dataloader';
import { prisma } from '../db.js';

// Generic DataLoader factory
function createLoader(model, field = 'id') {
  return new DataLoader(async (keys) => {
    const records = await prisma[model].findMany({
      where: { [field]: { in: keys } },
    });
    
    return keys.map(key =>
      records.find(r => r[field] === key) || null
    );
  }, {
    maxBatchSize: 100,
    cache: true, // Default: per-request cache
  });
}

// สร้าง loaders factory สำหรับใส่ใน context
export function createLoaders() {
  return {
    userById: createLoader('user'),
    productById: createLoader('product'),
    categoryById: createLoader('category'),
    
    // Many-to-one: load posts by userId
    postsByUserId: new DataLoader(async (userIds) => {
      const posts = await prisma.post.findMany({
        where: { authorId: { in: userIds } },
        orderBy: { createdAt: 'desc' },
      });
      
      // Group by userId
      return userIds.map(userId =>
        posts.filter(p => p.authorId === userId)
      );
    }),
    
    // Aggregate DataLoader
    postCountByUserId: new DataLoader(async (userIds) => {
      const counts = await prisma.post.groupBy({
        by: ['authorId'],
        where: { authorId: { in: userIds } },
        _count: true,
      });
      
      const countMap = new Map(counts.map(c => [c.authorId, c._count]));
      return userIds.map(id => countMap.get(id) || 0);
    }),
    
    // Order items with product details
    orderItemsByOrderId: new DataLoader(async (orderIds) => {
      const items = await prisma.orderItem.findMany({
        where: { orderId: { in: orderIds } },
        include: { product: true },
      });
      
      return orderIds.map(orderId =>
        items.filter(i => i.orderId === orderId)
      );
    }),
  };
}
```

---

## 📌 Step 533: Context Integration

```javascript
// src/context.js
import { createLoaders } from './loaders/index.js';

export async function createContext({ req }) {
  const user = await getUser(req);
  
  return {
    prisma,
    redis,
    user,
    loaders: createLoaders(), // ← ใหม่ทุก request
    req,
  };
}
```

---

## 📌 Step 534: Multi-key DataLoader

```javascript
// DataLoader ที่ batch ด้วย composite key

// ตัวอย่าง: Load product inventory โดย productId + warehouseId
const inventoryLoader = new DataLoader(
  async (keys) => {
    // keys = [{ productId, warehouseId }, ...]
    const inventories = await prisma.inventory.findMany({
      where: {
        OR: keys.map(k => ({
          productId: k.productId,
          warehouseId: k.warehouseId,
        })),
      },
    });
    
    return keys.map(({ productId, warehouseId }) =>
      inventories.find(
        i => i.productId === productId && i.warehouseId === warehouseId
      ) || null
    );
  },
  {
    cacheKeyFn: (key) => `${key.productId}:${key.warehouseId}`,
  }
);

// ใช้งาน
const inventory = await inventoryLoader.load({
  productId: 'prod-1',
  warehouseId: 'wh-1',
});
```

---

## 📌 Step 535: DataLoader Cache Invalidation

```javascript
// DataLoader มี per-request cache
// ปัญหา: หลัง mutation, cache อาจ stale

const resolvers = {
  Mutation: {
    updateUser: async (_, { id, input }, { prisma, loaders }) => {
      const user = await prisma.user.update({
        where: { id },
        data: input,
      });
      
      // Invalidate cache หลัง mutation
      loaders.userById.clear(id);
      
      // Or prime with new value
      loaders.userById.prime(id, user);
      
      return user;
    },
    
    deletePost: async (_, { id }, { prisma, loaders }) => {
      const post = await prisma.post.delete({ where: { id } });
      
      // Clear related caches
      loaders.postsByUserId.clear(post.authorId);
      loaders.postCountByUserId.clear(post.authorId);
      
      return true;
    },
  },
};
```

---

## 📌 Step 536: ใช้ DataLoader ใน Resolvers

```javascript
// ✅ ดี: ใช้ loaders
const resolvers = {
  Post: {
    author: (post, _, { loaders }) =>
      loaders.userById.load(post.authorId),
    
    comments: (post, _, { loaders }) =>
      loaders.commentsByPostId.load(post.id),
    
    likes: (post, _, { loaders }) =>
      loaders.likeCountByPostId.load(post.id),
  },
  
  User: {
    posts: (user, { limit = 10 }, { loaders }) =>
      // ❌ DataLoader ไม่รองรับ arguments ต่างกัน
      // ต้องสร้าง loader ใหม่หรือใช้ prisma ตรง
      loaders.postsByUserId.load(user.id),
      
    // สำหรับ pagination ใช้ prisma ตรง
    paginatedPosts: async (user, { first, after }, { prisma }) => {
      return prisma.post.findMany({
        where: { authorId: user.id },
        take: first + 1,
        cursor: after ? { id: after } : undefined,
        orderBy: { createdAt: 'desc' },
      });
    },
  },
};
```

---

## 📌 Step 537: DataLoader Testing

```javascript
// test/loaders.test.js
import { describe, it, expect, vi } from 'vitest';
import { createLoaders } from '../src/loaders/index.js';
import { prisma } from '../src/db.js';

vi.mock('../src/db.js', () => ({
  prisma: {
    user: {
      findMany: vi.fn(),
    },
  },
}));

describe('UserLoader', () => {
  it('should batch multiple loads into one query', async () => {
    const mockUsers = [
      { id: '1', name: 'Alice' },
      { id: '2', name: 'Bob' },
    ];
    
    prisma.user.findMany.mockResolvedValue(mockUsers);
    
    const { userById } = createLoaders();
    
    // Load 2 users simultaneously
    const [alice, bob] = await Promise.all([
      userById.load('1'),
      userById.load('2'),
    ]);
    
    // Should batch into single query
    expect(prisma.user.findMany).toHaveBeenCalledTimes(1);
    expect(prisma.user.findMany).toHaveBeenCalledWith({
      where: { id: { in: ['1', '2'] } },
    });
    
    expect(alice).toEqual({ id: '1', name: 'Alice' });
    expect(bob).toEqual({ id: '2', name: 'Bob' });
  });
  
  it('should return null for missing users', async () => {
    prisma.user.findMany.mockResolvedValue([]);
    
    const { userById } = createLoaders();
    const result = await userById.load('non-existent');
    
    expect(result).toBeNull();
  });
});
```

---

## 📌 Step 538: สรุป Part 037

### เนื้อหาที่เรียนรู้

✅ DataLoader internals  
✅ DataLoader factory pattern  
✅ Multi-key DataLoader  
✅ Cache invalidation  
✅ Integration กับ resolvers  
✅ DataLoader testing  

### Performance Tips

```
1. สร้าง loaders ใหม่ทุก request (ป้องกัน data leak)
2. ใช้ maxBatchSize ป้องกัน IN clause ใหญ่เกิน
3. Prime cache หลัง mutation
4. Clear cache หลัง delete
5. ระวัง loader ที่ใช้ arguments (ต้อง handle พิเศษ)
```

### ในส่วนถัดไป

➡️ **[Part 038](./part-038.md)** — Audit Logging & Compliance

---

*Part 037 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
