# Part 046 — Query Depth & Complexity Security 🛡️

> **ระดับ:** Expert | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 701–730

---

## 🎯 สิ่งที่จะได้เรียนรู้

- DoS ผ่าน deep queries
- DoS ผ่าน expensive queries
- Query depth limiting
- Query complexity analysis
- Query timeout
- Persisted queries whitelist
- Disabling introspection ใน production
- Complete security checklist

---

## 📌 Step 701: DoS via Deeply Nested Queries

```graphql
# ❌ อาจทำให้ server ช้ามาก
query {
  user(id: "1") {
    friends {
      friends {
        friends {
          friends {
            friends {
              friends { name }  # ← depth 7
            }
          }
        }
      }
    }
  }
}
```

### ป้องกัน Depth

```bash
npm install graphql-depth-limit
```

```javascript
import depthLimit from 'graphql-depth-limit';
import { ApolloServer } from '@apollo/server';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    depthLimit(5),  // max 5 levels deep
  ],
});
```

---

## 📌 Step 702: Complexity Analysis

```bash
npm install graphql-query-complexity
```

```javascript
import {
  createComplexityLimitRule,
  simpleEstimator,
  fieldExtensionsEstimator,
} from 'graphql-query-complexity';

// กำหนด complexity limit
const complexityLimitRule = createComplexityLimitRule(500, {
  estimators: [
    // ใช้ @complexity directive ใน schema
    fieldExtensionsEstimator(),
    
    // Default: แต่ละ field = 1
    simpleEstimator({ defaultComplexity: 1 }),
  ],
  
  onCost(cost) {
    console.log(`Query complexity: ${cost}`);
  },
  
  formatErrorMessage: (cost) =>
    `Query too complex (${cost}). Max allowed: 500.`,
});

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    depthLimit(5),
    complexityLimitRule,
  ],
});
```

### กำหนด Complexity ใน Schema

```graphql
type Query {
  # ง่าย
  user(id: ID!): User @complexity(value: 1)
  
  # list = multiplier
  users(limit: Int): [User!]! @complexity(value: 2, multipliers: ["limit"])
  
  # expensive operations
  searchUsers(term: String!): [User!]! @complexity(value: 10)
  
  # aggregate
  userStats: UserStats! @complexity(value: 20)
}
```

---

## 📌 Step 703: Query Timeout

```javascript
// src/plugins/timeout.plugin.js
export const timeoutPlugin = {
  async requestDidStart() {
    return {
      async executionDidStart() {
        const timeout = setTimeout(() => {
          throw new GraphQLError('Query timeout: 10 seconds exceeded', {
            extensions: { code: 'QUERY_TIMEOUT' }
          });
        }, 10000); // 10 seconds
        
        return {
          executionDidEnd: () => clearTimeout(timeout),
        };
      },
    };
  },
};
```

---

## 📌 Step 704: Persisted Queries Whitelist

```javascript
// ใน production — อนุญาตเฉพาะ queries ที่รู้จัก
import { ApolloServer } from '@apollo/server';
import { ApolloServerPluginLandingPageDisabled } from '@apollo/server/plugin/disabled';
import sha256 from 'crypto-js/sha256.js';

// Map of known query hashes
const ALLOWED_QUERIES = new Map([
  ['abc123...', 'query GetUser($id: ID!) { user(id: $id) { name email } }'],
  ['def456...', 'query GetProducts { products { id name price } }'],
]);

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    ApolloServerPluginLandingPageDisabled(),  // ปิด Playground ใน production
  ],
  
  // ป้องกัน arbitrary queries
  allowBatchedHttpRequests: false,
});

// Custom middleware สำหรับ whitelist
app.use('/graphql', (req, res, next) => {
  if (process.env.NODE_ENV !== 'production') return next();
  
  const { query, extensions } = req.body;
  
  // ถ้าไม่มี query hash = reject
  const queryHash = extensions?.persistedQuery?.sha256Hash;
  
  if (!queryHash && query) {
    // ในโหมด strict — ไม่รับ ad-hoc queries
    return res.status(400).json({
      errors: [{ message: 'Only persisted queries are allowed' }]
    });
  }
  
  if (queryHash && !ALLOWED_QUERIES.has(queryHash)) {
    return res.status(400).json({
      errors: [{ message: 'Unknown query' }]
    });
  }
  
  next();
});
```

---

## 📌 Step 705: Complete Security Config

```javascript
// src/server-secure.js — Production-ready configuration
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import depthLimit from 'graphql-depth-limit';
import { createComplexityLimitRule } from 'graphql-query-complexity';
import { NoSchemaIntrospectionCustomRule } from 'graphql';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  
  // Disable introspection in production
  introspection: process.env.NODE_ENV !== 'production',
  
  validationRules: [
    // Depth limit
    depthLimit(6),
    
    // Complexity limit
    createComplexityLimitRule(1000, {
      estimators: [simpleEstimator({ defaultComplexity: 1 })],
    }),
    
    // No introspection in production
    ...(process.env.NODE_ENV === 'production'
      ? [NoSchemaIntrospectionCustomRule]
      : []),
  ],
  
  plugins: [
    // Disable playground in production
    process.env.NODE_ENV === 'production'
      ? ApolloServerPluginLandingPageDisabled()
      : ApolloServerPluginLandingPageLocalDefault(),
    
    // Timeout plugin
    timeoutPlugin,
    
    // Metrics
    metricsPlugin,
  ],
  
  formatError: (formattedError, error) => {
    logger.error({ error }, 'GraphQL error');
    
    // Mask internal errors
    if (process.env.NODE_ENV === 'production') {
      const safeCode = formattedError.extensions?.code;
      const safeCodes = ['UNAUTHENTICATED', 'FORBIDDEN', 'NOT_FOUND', 'BAD_USER_INPUT', 'VALIDATION_ERROR'];
      
      if (!safeCodes.includes(safeCode)) {
        return {
          message: 'An error occurred',
          extensions: { code: 'INTERNAL_SERVER_ERROR' },
        };
      }
    }
    
    return formattedError;
  },
});

// Express middleware security headers
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:', 'https:'],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true,
  },
}));
```

---

## 📌 Step 706: Security Testing

```javascript
// test/security/dos.test.js
describe('DoS Protection', () => {
  it('should block deeply nested queries', async () => {
    const deepQuery = `
      query {
        user(id: "1") {
          friends { friends { friends { friends { friends { friends { friends { name } } } } } } }
        }
      }
    `;
    
    const response = await request(app)
      .post('/graphql')
      .send({ query: deepQuery });
    
    expect(response.body.errors[0].message).toContain('exceeds maximum');
  });
  
  it('should block high complexity queries', async () => {
    // Generate a very complex query
    const complexQuery = `
      query {
        ${Array.from({ length: 100 }, (_, i) => `u${i}: user(id: "${i}") { name email }`).join('\n')}
      }
    `;
    
    const response = await request(app)
      .post('/graphql')
      .send({ query: complexQuery });
    
    expect(response.body.errors[0].message).toContain('too complex');
  });
});
```

---

## 📌 Step 707: สรุป Part 046

### เนื้อหาที่เรียนรู้

✅ DoS via deep/complex queries  
✅ Query depth limiting  
✅ Complexity analysis  
✅ Query timeout  
✅ Persisted queries whitelist  
✅ Complete production security config  

### Security Checklist

```
✅ Disable introspection in production
✅ Limit query depth (max 6-7 levels)
✅ Limit query complexity (max 1000)
✅ Query timeout (10-30 seconds)
✅ Rate limiting per IP + per user
✅ Input validation
✅ Error masking
✅ CORS configuration
✅ Security headers (helmet)
✅ Persisted queries in production
✅ Logging and monitoring
✅ Regular security audits
```

### ในส่วนถัดไป

➡️ **[Part 047](./part-047.md)** — Advanced Authentication: OAuth, SSO

---

*Part 046 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
