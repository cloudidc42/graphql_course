# Part 031 — GraphQL Security: Rate Limiting & DoS Prevention 🛡️

> **ระดับ:** Intermediate-Advanced | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 301–340

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Rate limiting ประเภทต่างๆ
- IP-based rate limiting
- User-based rate limiting
- Operation-based rate limiting
- Sliding window algorithm
- Token bucket algorithm
- Redis-backed rate limiting
- GraphQL-specific DoS patterns
- ป้องกัน batching abuse

---

## 📌 Step 301: ทำไม Rate Limiting สำคัญมาก

```
⚠️ GraphQL มี attack surface ที่ต่างจาก REST:

1. Single endpoint → ทุก request ไปที่ /graphql
2. Flexible queries → attacker เลือก field เองได้
3. Batching → ส่ง 100 queries ในครั้งเดียว
4. Aliases → duplicate fields ในคำสั่งเดียว
5. Nested queries → recursive loading

ผลกระทบ:
- Server CPU spike
- Database connection exhaustion
- Memory overflow
- Service downtime
```

---

## 📌 Step 302: Basic Rate Limiting ด้วย express-rate-limit

```bash
npm install express-rate-limit
```

```javascript
// src/middleware/rateLimit.js
import rateLimit from 'express-rate-limit';

// Global rate limit
export const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,                  // max 100 requests per window
  standardHeaders: true,
  legacyHeaders: false,
  
  message: {
    errors: [{ message: 'Too many requests, please try again later' }]
  },
  
  // Custom key: IP + User-Agent
  keyGenerator: (req) => {
    return `${req.ip}-${req.headers['user-agent']}`;
  },
  
  skip: (req) => {
    // ไม่ rate limit health check
    return req.path === '/health';
  },
});

// Strict limiter สำหรับ auth operations
export const authLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 10,                   // max 10 login attempts per hour
  skipSuccessfulRequests: true, // นับเฉพาะ failed requests
  message: {
    errors: [{ message: 'Too many authentication attempts' }]
  },
});
```

```javascript
// src/index.js
import { globalLimiter, authLimiter } from './middleware/rateLimit.js';

app.use('/graphql', globalLimiter);
```

---

## 📌 Step 303: Redis-backed Rate Limiting

```bash
npm install rate-limiter-flexible ioredis
```

```javascript
// src/middleware/redisRateLimit.js
import { RateLimiterRedis } from 'rate-limiter-flexible';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

// Per-IP rate limiter
const ipRateLimiter = new RateLimiterRedis({
  storeClient: redis,
  keyPrefix: 'rl:ip',
  points: 60,          // 60 requests
  duration: 60,        // per 60 seconds (1 minute)
  blockDuration: 300,  // block 5 minutes if exceeded
});

// Per-user rate limiter (authenticated)
const userRateLimiter = new RateLimiterRedis({
  storeClient: redis,
  keyPrefix: 'rl:user',
  points: 200,         // 200 requests
  duration: 60,        // per minute
  blockDuration: 60,   // block 1 minute
});

// Per-operation rate limiter
const operationRateLimiter = new RateLimiterRedis({
  storeClient: redis,
  keyPrefix: 'rl:op',
  points: 10,          // 10 mutation requests
  duration: 60,        // per minute
});

export async function graphqlRateLimitMiddleware(req, res, next) {
  try {
    const userId = req.user?.sub;
    const ip = req.ip;
    
    // Always check IP limit
    await ipRateLimiter.consume(ip);
    
    // If authenticated, use user limit (more generous)
    if (userId) {
      await userRateLimiter.consume(userId);
    }
    
    // Check operation-specific limit for mutations
    const body = req.body;
    if (body?.query?.trim().startsWith('mutation')) {
      const key = userId || ip;
      await operationRateLimiter.consume(`mutation:${key}`);
    }
    
    next();
  } catch (rejRes) {
    const secs = Math.ceil(rejRes.msBeforeNext / 1000) || 1;
    res.set('Retry-After', String(secs));
    res.set('X-RateLimit-Reset', new Date(Date.now() + rejRes.msBeforeNext).toISOString());
    
    res.status(429).json({
      errors: [{
        message: 'Rate limit exceeded',
        extensions: {
          code: 'RATE_LIMITED',
          retryAfter: secs,
        }
      }]
    });
  }
}
```

---

## 📌 Step 304: Sliding Window Algorithm

```javascript
// ใช้ Redis Sorted Set สำหรับ sliding window
async function slidingWindowRateLimit(redis, key, limit, windowMs) {
  const now = Date.now();
  const windowStart = now - windowMs;
  
  const pipeline = redis.pipeline();
  
  // ลบ entries เก่ากว่า window
  pipeline.zremrangebyscore(key, 0, windowStart);
  
  // นับ requests ใน window ปัจจุบัน
  pipeline.zcard(key);
  
  // เพิ่ม request ปัจจุบัน
  pipeline.zadd(key, now, `${now}-${Math.random()}`);
  
  // Set expiry
  pipeline.pexpire(key, windowMs);
  
  const results = await pipeline.exec();
  const requestCount = results[1][1];
  
  return {
    allowed: requestCount < limit,
    remaining: Math.max(0, limit - requestCount - 1),
    reset: windowStart + windowMs,
  };
}

// ใน resolver
async function createPostResolver(parent, { input }, { user, redis }) {
  if (!user) throw new GraphQLError('Unauthenticated');
  
  const { allowed, remaining } = await slidingWindowRateLimit(
    redis,
    `posts:${user.sub}`,
    5,      // max 5 posts
    3600000 // per hour
  );
  
  if (!allowed) {
    throw new GraphQLError('Post creation rate limit exceeded', {
      extensions: { code: 'RATE_LIMITED', remaining }
    });
  }
  
  return prisma.post.create({ data: { ...input, authorId: user.sub } });
}
```

---

## 📌 Step 305: Batching Attack Prevention

```javascript
// ป้องกัน query batching abuse
// attacker ส่ง array ของ queries ในครั้งเดียว

// ❌ Enable batching = อันตราย
const server = new ApolloServer({
  allowBatchedHttpRequests: true, // ← อย่าเปิดโดยไม่มี protection!
});

// ✅ จำกัด batch size
app.use('/graphql', (req, res, next) => {
  if (Array.isArray(req.body)) {
    if (req.body.length > 5) {
      return res.status(400).json({
        errors: [{ message: 'Maximum batch size is 5' }]
      });
    }
  }
  next();
});

// ✅ Rate limit ทั้ง batch
app.use('/graphql', (req, res, next) => {
  const batchSize = Array.isArray(req.body) ? req.body.length : 1;
  req._batchSize = batchSize;
  next();
});
```

---

## 📌 Step 306: Alias-based DoS Prevention

```javascript
// attacker ใช้ aliases เพื่อ multiply load:
// query {
//   a1: expensiveField
//   a2: expensiveField
//   a3: expensiveField
//   ... (100 times)
// }

// ป้องกัน: นับ aliases
import { visit, Kind } from 'graphql';

export function maxAliasesRule(maxAliases = 15) {
  return function(context) {
    let aliasCount = 0;
    
    return {
      Field(node) {
        if (node.alias) {
          aliasCount++;
          if (aliasCount > maxAliases) {
            context.reportError(
              new GraphQLError(`Too many aliases: ${aliasCount}. Max: ${maxAliases}`)
            );
          }
        }
      },
    };
  };
}

// ใน server
const server = new ApolloServer({
  validationRules: [maxAliasesRule(15)],
});
```

---

## 📌 Step 307: @rateLimit Directive

```graphql
directive @rateLimit(
  max: Int!
  window: Int!  # seconds
  message: String
) on FIELD_DEFINITION

type Mutation {
  login(email: String!, password: String!): AuthPayload
    @rateLimit(max: 5, window: 900, message: "Too many login attempts")
  
  sendEmailVerification(email: String!): Boolean
    @rateLimit(max: 3, window: 3600, message: "Wait before requesting another email")
  
  createPost(input: CreatePostInput!): Post
    @rateLimit(max: 10, window: 3600)
}
```

```javascript
// Implementation using directive transformer
import { mapSchema, getDirective, MapperKind } from '@graphql-tools/utils';
import { defaultFieldResolver } from 'graphql';

export function rateLimitDirectiveTransformer(schema, redis) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const directive = getDirective(schema, fieldConfig, 'rateLimit')?.[0];
      if (!directive) return fieldConfig;
      
      const { max, window, message } = directive;
      const { resolve = defaultFieldResolver } = fieldConfig;
      
      fieldConfig.resolve = async function(source, args, context, info) {
        const userId = context.user?.sub || context.ip;
        const key = `rl:${info.parentType.name}:${info.fieldName}:${userId}`;
        
        const { allowed, remaining } = await slidingWindowRateLimit(
          redis, key, max, window * 1000
        );
        
        if (!allowed) {
          throw new GraphQLError(
            message || 'Rate limit exceeded',
            { extensions: { code: 'RATE_LIMITED', remaining } }
          );
        }
        
        return resolve(source, args, context, info);
      };
      
      return fieldConfig;
    },
  });
}
```

---

## 📌 Step 308: Rate Limit Response Headers

```javascript
// ส่ง rate limit info ใน headers
export const rateLimitPlugin = {
  async requestDidStart({ contextValue }) {
    return {
      async willSendResponse({ response, contextValue }) {
        const rateLimitInfo = contextValue.rateLimitInfo;
        if (!rateLimitInfo) return;
        
        // เพิ่ม rate limit headers
        if (response.http) {
          response.http.headers.set('X-RateLimit-Limit', String(rateLimitInfo.limit));
          response.http.headers.set('X-RateLimit-Remaining', String(rateLimitInfo.remaining));
          response.http.headers.set('X-RateLimit-Reset', String(rateLimitInfo.reset));
        }
      },
    };
  },
};
```

---

## 📌 Step 309: Testing Rate Limits

```javascript
// test/rateLimit.test.js
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import app from '../src/app.js';

describe('Rate Limiting', () => {
  it('should allow requests within limit', async () => {
    for (let i = 0; i < 5; i++) {
      const res = await request(app)
        .post('/graphql')
        .send({ query: '{ health }' });
      expect(res.status).toBe(200);
    }
  });
  
  it('should block requests over limit', async () => {
    // Make 11 rapid requests (limit = 10 per minute)
    const requests = Array.from({ length: 11 }, () =>
      request(app)
        .post('/graphql')
        .set('X-Forwarded-For', '1.2.3.4') // same IP
        .send({ query: '{ health }' })
    );
    
    const results = await Promise.all(requests);
    const blocked = results.filter(r => r.status === 429);
    
    expect(blocked.length).toBeGreaterThan(0);
    expect(blocked[0].body.errors[0].message).toContain('Rate limit');
  });
  
  it('should return Retry-After header', async () => {
    // Exhaust limit then check header
    const res = await exhaustRateLimit();
    expect(res.headers['retry-after']).toBeDefined();
  });
});
```

---

## 📌 Step 310: สรุป Part 031

### เนื้อหาที่เรียนรู้

✅ Rate limiting แบบต่างๆ  
✅ Redis sliding window  
✅ Batching attack prevention  
✅ Alias-based DoS prevention  
✅ @rateLimit directive  
✅ Rate limit response headers  
✅ Testing rate limits  

### Best Practices

```
1. ใช้ Redis สำหรับ distributed rate limiting
2. Rate limit per-IP AND per-user
3. ใช้ sliding window (ไม่ใช่ fixed window)
4. Block อย่างน้อย 5 นาทีหลัง exceed limit
5. ส่ง Retry-After header เสมอ
6. Log rate limit violations
7. Alert เมื่อมีการ abuse
```

### ในส่วนถัดไป

➡️ **[Part 032](./part-032.md)** — Input Validation & Sanitization

---

*Part 031 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
