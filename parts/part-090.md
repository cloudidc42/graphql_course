# Part 090 — Advanced DataLoader Patterns 🔄

> **ระดับ:** Expert | **เวลาเรียน:** 80 นาที | **ขั้นตอนที่:** 2526–2570

---

## 🎯 สิ่งที่จะได้เรียนรู้

- DataLoader internals
- Batching strategies
- Tiered caching ด้วย DataLoader
- DataLoader กับ Prisma
- Conditional batching
- Prime cache patterns
- Error handling ใน batch
- Distributed DataLoader

---

## 📌 Step 2526: DataLoader Internals

```
DataLoader ทำงานอย่างไร:

1. Resolver เรียก loader.load(key)
2. DataLoader เก็บ key ไว้ใน queue
3. ใน event loop tick ถัดไป:
   - รวม keys ทั้งหมด → batch
   - เรียก batchFn([key1, key2, key3])
   - แจก results กลับไปแต่ละ Resolver

Timeline:
  tick 1: load('user-1')  → queue: ['user-1']
  tick 1: load('user-2')  → queue: ['user-1', 'user-2']
  tick 1: load('user-1')  → cache hit, queue: ['user-1', 'user-2']
  ── event loop tick ──
  tick 2: batchFn(['user-1', 'user-2']) → 1 DB query
  tick 2: resolve all promises

Performance:
  Without DataLoader: N queries (N+1 problem)
  With DataLoader: 1 query per batch size
```

---

## 📌 Step 2527: Tiered Caching DataLoader

```typescript
// src/loaders/tiered-cache.loader.ts
// L1: in-memory (request scope)
// L2: Redis (shared, longer TTL)

class TieredCacheLoader<K, V> {
  private memoryCache = new Map<string, V>();
  private loader: DataLoader<K, V | null>;
  
  constructor(
    private batchFn: (keys: readonly K[]) => Promise<(V | null)[]>,
    private redis: Redis,
    private options: {
      cacheKeyFn: (key: K) => string;
      redisTTL: number;     // วินาที
    }
  ) {
    this.loader = new DataLoader(this.tieredBatchFn.bind(this), {
      cacheKeyFn: options.cacheKeyFn,
    });
  }
  
  async load(key: K): Promise<V | null> {
    return this.loader.load(key);
  }
  
  private async tieredBatchFn(keys: readonly K[]): Promise<(V | null)[]> {
    const keyStrings = keys.map(this.options.cacheKeyFn);
    const results = new Map<string, V | null>();
    
    // 1. Check memory cache (L1)
    const missingFromMemory: K[] = [];
    for (let i = 0; i < keys.length; i++) {
      const ks = keyStrings[i];
      if (this.memoryCache.has(ks)) {
        results.set(ks, this.memoryCache.get(ks)!);
      } else {
        missingFromMemory.push(keys[i]);
      }
    }
    
    if (missingFromMemory.length === 0) {
      return keys.map(k => results.get(this.options.cacheKeyFn(k)) ?? null);
    }
    
    // 2. Check Redis cache (L2)
    const missingFromRedis: K[] = [];
    const redisPipeline = this.redis.pipeline();
    
    for (const key of missingFromMemory) {
      redisPipeline.get(`loader:${this.options.cacheKeyFn(key)}`);
    }
    
    const redisResults = await redisPipeline.exec();
    
    for (let i = 0; i < missingFromMemory.length; i++) {
      const ks = this.options.cacheKeyFn(missingFromMemory[i]);
      const cached = redisResults?.[i]?.[1];
      
      if (cached) {
        const value = JSON.parse(cached as string) as V;
        results.set(ks, value);
        this.memoryCache.set(ks, value);  // Populate L1
      } else {
        missingFromRedis.push(missingFromMemory[i]);
      }
    }
    
    // 3. DB fetch (L0)
    if (missingFromRedis.length > 0) {
      const dbResults = await this.batchFn(missingFromRedis);
      const redisSave = this.redis.pipeline();
      
      for (let i = 0; i < missingFromRedis.length; i++) {
        const ks = this.options.cacheKeyFn(missingFromRedis[i]);
        const value = dbResults[i] ?? null;
        results.set(ks, value);
        
        // Save to L1 and L2
        this.memoryCache.set(ks, value!);
        if (value !== null) {
          redisSave.setex(
            `loader:${ks}`,
            this.options.redisTTL,
            JSON.stringify(value)
          );
        }
      }
      
      await redisSave.exec();
    }
    
    return keys.map(k => results.get(this.options.cacheKeyFn(k)) ?? null);
  }
  
  // Prime cache หลังจาก create/update
  prime(key: K, value: V) {
    const ks = this.options.cacheKeyFn(key);
    this.memoryCache.set(ks, value);
    this.redis.setex(`loader:${ks}`, this.options.redisTTL, JSON.stringify(value));
    this.loader.prime(key, value);
  }
  
  // Invalidate cache หลัง update
  async invalidate(key: K) {
    const ks = this.options.cacheKeyFn(key);
    this.memoryCache.delete(ks);
    await this.redis.del(`loader:${ks}`);
    this.loader.clear(key);
  }
}
```

---

## 📌 Step 2528: DataLoader กับ Prisma

```typescript
// src/loaders/prisma-loaders.ts

export function createUserLoader(prisma: PrismaClient): DataLoader<string, User | null> {
  return new DataLoader(
    async (ids: readonly string[]) => {
      const users = await prisma.user.findMany({
        where: { id: { in: [...ids] } },
        // ดึงเฉพาะ fields ที่จะใช้
        select: {
          id: true,
          email: true,
          name: true,
          role: true,
          avatarUrl: true,
        },
      });
      
      const map = new Map(users.map(u => [u.id, u]));
      return ids.map(id => map.get(id) ?? null);
    },
    {
      maxBatchSize: 100,
      // Cache per request (default: true)
      cache: true,
    }
  );
}

// Loader สำหรับ one-to-many
export function createOrdersByUserLoader(prisma: PrismaClient): DataLoader<string, Order[]> {
  return new DataLoader(
    async (userIds: readonly string[]) => {
      const orders = await prisma.order.findMany({
        where: {
          userId: { in: [...userIds] },
          // Filter ที่ต้องการ
        },
        orderBy: { createdAt: 'desc' },
        take: 10,  // limit per user
      });
      
      // Group by userId
      const ordersByUser = new Map<string, Order[]>();
      for (const order of orders) {
        const arr = ordersByUser.get(order.userId) ?? [];
        arr.push(order);
        ordersByUser.set(order.userId, arr);
      }
      
      return userIds.map(id => ordersByUser.get(id) ?? []);
    },
    { maxBatchSize: 50 }
  );
}

// Loader สำหรับ complex composite key
interface ProductVariantKey {
  productId: string;
  color: string;
  size: string;
}

export function createProductVariantLoader(prisma: PrismaClient) {
  return new DataLoader<ProductVariantKey, ProductVariant | null>(
    async (keys) => {
      // Composite key loader
      const variants = await prisma.productVariant.findMany({
        where: {
          OR: keys.map(k => ({
            productId: k.productId,
            color: k.color,
            size: k.size,
          })),
        },
      });
      
      const map = new Map(
        variants.map(v => [`${v.productId}:${v.color}:${v.size}`, v])
      );
      
      return keys.map(k => map.get(`${k.productId}:${k.color}:${k.size}`) ?? null);
    },
    {
      cacheKeyFn: (k) => `${k.productId}:${k.color}:${k.size}`,
    }
  );
}
```

---

## 📌 Step 2529: Prime Cache Pattern

```typescript
// src/resolvers/product.resolver.ts
// Prime DataLoader cache หลัง create/update
// เพื่อหลีกเลี่ยง DB round-trip ทันที

const productResolvers = {
  Mutation: {
    createProduct: async (
      _: unknown,
      { input }: CreateProductArgs,
      { prisma, loaders, user }: AppContext
    ) => {
      requireRole(user, 'ADMIN');
      
      const product = await prisma.product.create({ data: input });
      
      // Prime DataLoader cache ทันที
      // Queries ที่ตามมาหลัง mutation จะได้ผลจาก cache
      loaders.productById.prime(product.id, product);
      
      return product;
    },
    
    updateProduct: async (
      _: unknown,
      { id, input }: UpdateProductArgs,
      { prisma, loaders, user }: AppContext
    ) => {
      requireRole(user, 'ADMIN');
      
      const product = await prisma.product.update({
        where: { id },
        data: input,
      });
      
      // Clear old cache + prime new value
      loaders.productById.clear(id);
      loaders.productById.prime(id, product);
      
      return product;
    },
    
    deleteProduct: async (
      _: unknown,
      { id }: { id: string },
      { prisma, loaders, user }: AppContext
    ) => {
      requireRole(user, 'ADMIN');
      await prisma.product.delete({ where: { id } });
      
      // Invalidate cache
      loaders.productById.clear(id);
      
      return { success: true };
    },
  },
};
```

---

## 📌 Step 2530: Error Handling ใน Batch

```typescript
// src/loaders/safe-batch-loader.ts
// Handle errors ใน batch function อย่าง safe

function createSafeBatchLoader<K, V>(
  batchFn: (keys: readonly K[]) => Promise<(V | Error | null)[]>
): DataLoader<K, V | null> {
  return new DataLoader(async (keys) => {
    try {
      const results = await batchFn(keys);
      
      return results.map((result, index) => {
        if (result instanceof Error) {
          // Log แต่ไม่ crash batch ทั้งหมด
          logger.error({ error: result, key: keys[index] }, 'Loader batch error for key');
          return null;  // หรือ return Error เพื่อให้ propagate
        }
        return result;
      });
    } catch (err) {
      // Total batch failure: return Error สำหรับทุก key
      logger.error({ error: err }, 'Loader batch function failed');
      return keys.map(() => null);
    }
  });
}

// Loader with circuit breaker
function createResilientLoader<K extends string, V>(
  name: string,
  batchFn: (keys: readonly K[]) => Promise<(V | null)[]>,
  breaker: CircuitBreaker
): DataLoader<K, V | null> {
  return new DataLoader(async (keys) => {
    try {
      return await breaker.fire(keys) as (V | null)[];
    } catch (err) {
      if ((err as Error).message === 'Breaker is open') {
        logger.warn(`${name} loader: circuit open, returning nulls`);
        return keys.map(() => null);
      }
      throw err;
    }
  });
}
```

---

## 📌 Step 2531: สรุป Part 090

### เนื้อหาที่เรียนรู้

✅ DataLoader internals  
✅ Tiered caching (L1 memory + L2 Redis)  
✅ Prisma batch loaders  
✅ Composite key loaders  
✅ Prime cache pattern  
✅ Error handling ใน batch  

### DataLoader Best Practices

```
Do:
✅ สร้าง loaders ใหม่ต่อ request
✅ Prime cache หลัง mutation
✅ Clear cache หลัง update/delete
✅ Return null (ไม่ใช่ Error) สำหรับ not-found
✅ maxBatchSize ที่เหมาะสมกับ DB

Don't:
❌ Reuse loaders ข้าม requests
❌ ลืม handle missing keys (null)
❌ Batch size เยอะเกิน (DB in-clause limits)
❌ Share loaders ระหว่าง users

Performance Tips:
□ Select เฉพาะ fields ที่จะใช้
□ Database indexes บน batch keys
□ Redis TTL ตาม data freshness
□ Monitor DataLoader hit rate
```

### ในส่วนถัดไป

➡️ **[Part 091](./part-091.md)** — GraphQL กับ Message Queues

---

*Part 090 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
