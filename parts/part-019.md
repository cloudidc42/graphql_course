# Part 019 — Caching Strategies 🗄️

> **ระดับ:** Intermediate | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 631–660

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Caching layers
- In-memory caching ด้วย Map/LRU
- Redis caching
- Response caching
- Persisted queries
- Apollo Client cache
- Cache invalidation strategies
- CDN caching

---

## 📌 Step 631: Caching Layers

```
Caching Layers สำหรับ GraphQL:

1. CDN Cache (CloudFlare, Fastly)
   → เฉพาะ GET requests, public data
   
2. Response Cache (Redis, Memcached)
   → Cache full response
   → Per-operation caching
   
3. Resolver Cache (DataLoader)
   → Cache per-request
   → ป้องกัน N+1
   
4. Database Query Cache
   → ORM level (Sequelize, Prisma)
   
5. Apollo Client Cache
   → Client-side normalized cache
```

---

## 📌 Step 632: In-Memory LRU Cache

```javascript
// src/utils/lru-cache.js
export class LRUCache {
  constructor(maxSize = 1000, ttlMs = 5 * 60 * 1000) {
    this.cache = new Map();
    this.maxSize = maxSize;
    this.ttlMs = ttlMs;
  }
  
  get(key) {
    const item = this.cache.get(key);
    if (!item) return undefined;
    
    if (Date.now() > item.expiresAt) {
      this.cache.delete(key);
      return undefined;
    }
    
    // Move to end (most recently used)
    this.cache.delete(key);
    this.cache.set(key, item);
    
    return item.value;
  }
  
  set(key, value) {
    // Evict oldest if at capacity
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    this.cache.set(key, {
      value,
      expiresAt: Date.now() + this.ttlMs,
    });
  }
  
  delete(key) {
    this.cache.delete(key);
  }
  
  clear() {
    this.cache.clear();
  }
  
  get size() {
    return this.cache.size;
  }
}
```

---

## 📌 Step 633: Redis Resolver Cache

```javascript
// src/cache/resolver-cache.js
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();

export async function cachedResolver(key, ttlSeconds, resolver) {
  const cached = await redis.get(key);
  if (cached) {
    return JSON.parse(cached);
  }
  
  const result = await resolver();
  await redis.setEx(key, ttlSeconds, JSON.stringify(result));
  
  return result;
}

// ใช้ใน resolver
const resolvers = {
  Query: {
    categories: async () => {
      return cachedResolver(
        'categories:all',
        3600, // 1 hour
        () => Category.findAll({ order: [['name', 'ASC']] })
      );
    },
    
    featuredProducts: async () => {
      return cachedResolver(
        'products:featured',
        300, // 5 minutes
        () => Product.findAll({ where: { isFeatured: true }, limit: 10 })
      );
    },
    
    user: async (_, { id }) => {
      return cachedResolver(
        `user:${id}`,
        60, // 1 minute
        () => User.findById(id)
      );
    },
  },
};

// Cache invalidation
export async function invalidateUserCache(userId) {
  await redis.del(`user:${userId}`);
}

// Pattern-based invalidation
export async function invalidateProductsCache() {
  const keys = await redis.keys('products:*');
  if (keys.length > 0) {
    await redis.del(keys);
  }
}
```

---

## 📌 Step 634: Response Caching Plugin

```javascript
// src/plugins/response-cache.plugin.js
import { createClient } from 'redis';
import { createHash } from 'crypto';

const redis = createClient({ url: process.env.REDIS_URL });

export const responseCachePlugin = {
  async requestDidStart() {
    return {
      async responseForOperation({ request, contextValue }) {
        // ไม่ cache mutations หรือ subscriptions
        const isQuery = request.query?.includes('query ') || 
          !request.query?.includes('mutation ');
        
        if (!isQuery) return null;
        
        // ไม่ cache requests ที่มี user token (personalized)
        if (contextValue.user) return null;
        
        // สร้าง cache key จาก query + variables
        const cacheKey = `graphql:${createHash('sha256')
          .update(JSON.stringify({
            query: request.query,
            variables: request.variables,
          }))
          .digest('hex')}`;
        
        const cached = await redis.get(cacheKey);
        if (cached) {
          return JSON.parse(cached);
        }
        
        return null;
      },
      
      async willSendResponse({ request, response, contextValue }) {
        // Cache successful responses
        if (!response.body.singleResult.errors &&
            response.body.singleResult.data &&
            !contextValue.user) {
          
          const cacheKey = `graphql:${createHash('sha256')
            .update(JSON.stringify({
              query: request.query,
              variables: request.variables,
            }))
            .digest('hex')}`;
          
          await redis.setEx(
            cacheKey,
            300, // 5 minutes
            JSON.stringify(response.body.singleResult)
          );
        }
      },
    };
  },
};
```

---

## 📌 Step 635: Apollo Client Cache — Normalized

```javascript
// InMemoryCache configuration
const cache = new InMemoryCache({
  typePolicies: {
    // Customize how User objects are stored
    User: {
      keyFields: ['id'],
      fields: {
        // Merge nested arrays
        friends: {
          merge(existing = [], incoming) {
            return [...existing, ...incoming];
          },
        },
      },
    },
    
    // Paginated query
    Query: {
      fields: {
        users: {
          keyArgs: ['filter', 'sort'],  // args ที่ใช้แยก cache
          
          merge(existing = { edges: [] }, incoming) {
            return {
              ...incoming,
              edges: [...(existing.edges || []), ...incoming.edges],
            };
          },
          
          read(existing, { args }) {
            if (!existing) return undefined;
            
            // Slice สำหรับ offset pagination
            if (args?.offset !== undefined) {
              return {
                ...existing,
                edges: existing.edges.slice(args.offset, args.offset + args.limit),
              };
            }
            
            return existing;
          },
        },
      },
    },
  },
});
```

---

## 📌 Step 636: Cache Invalidation

```javascript
// Apollo Client — Manual cache update
const [deletePost] = useMutation(DELETE_POST, {
  update(cache, { data: { deletePost } }) {
    // ลบ post ออกจาก cache
    cache.evict({ id: cache.identify({ __typename: 'Post', id: deletePost.id }) });
    cache.gc(); // garbage collect
  },
});

// หรือ invalidate field
cache.evict({ fieldName: 'posts' });

// หรือ refetch
const [createPost] = useMutation(CREATE_POST, {
  refetchQueries: [{ query: GET_POSTS }],
  awaitRefetchQueries: true,
});
```

---

## 📌 Step 637: สรุป Part 019

### เนื้อหาที่เรียนรู้

✅ Caching layers  
✅ LRU cache implementation  
✅ Redis resolver cache  
✅ Response caching plugin  
✅ Apollo Client normalized cache  
✅ Cache invalidation strategies  

### Homework

1. implement Redis caching สำหรับ categories API
2. implement Apollo Client cache ด้วย type policies
3. เปรียบเทียบ performance ก่อนและหลัง caching

### ในส่วนถัดไป

➡️ **[Part 020](./part-020.md)** — GraphQL Performance Optimization

---

*Part 019 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
