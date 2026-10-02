# Part 020 — GraphQL Performance Optimization ⚡

> **ระดับ:** Intermediate | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 661–690

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Query complexity limiting
- Query depth limiting
- Field-level rate limiting
- Query cost analysis
- Schema design สำหรับ performance
- Database query optimization
- Connection pooling
- Monitoring และ profiling

---

## 📌 Step 661: Query Complexity

```javascript
// npm install graphql-query-complexity
import {
  createComplexityLimitRule,
  simpleEstimator,
  fieldExtensionsEstimator,
} from 'graphql-query-complexity';

const complexityLimitRule = createComplexityLimitRule(1000, {
  scalarCost: 1,
  objectCost: 2,
  listFactor: 10,
  
  onCost: (cost) => {
    console.log('Query cost:', cost);
  },
  
  formatErrorMessage: (cost) =>
    `Query is too complex: ${cost} (max: 1000)`,
  
  estimators: [
    fieldExtensionsEstimator(),
    simpleEstimator({ defaultComplexity: 1 }),
  ],
});

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [complexityLimitRule],
});
```

### กำหนด Complexity ใน Schema

```graphql
type Query {
  users(limit: Int): [User!]! @complexity(value: 5, multipliers: ["limit"])
  user(id: ID!): User @complexity(value: 1)
  searchUsers(term: String!): [User!]! @complexity(value: 10, multipliers: ["limit"])
}
```

---

## 📌 Step 662: Query Depth Limiting

```bash
npm install graphql-depth-limit
```

```javascript
import depthLimit from 'graphql-depth-limit';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    depthLimit(7),  // max depth = 7
  ],
});
```

### ตัวอย่าง Deep Query ที่จะถูก Block

```graphql
# Depth 7 — ผ่าน
query {
  user {                              # 1
    posts {                           # 2
      comments {                      # 3
        author {                      # 4
          posts {                     # 5
            comments {               # 6
              author { name }        # 7
            }
          }
        }
      }
    }
  }
}

# Depth 8 — ถูก block!
query {
  user {                              # 1
    posts {                           # 2
      comments {                      # 3
        author {                      # 4
          posts {                     # 5
            comments {               # 6
              author {               # 7
                profile { bio }      # 8 ← BLOCKED!
              }
            }
          }
        }
      }
    }
  }
}
```

---

## 📌 Step 663: Custom Complexity Rules

```javascript
// src/plugins/complexity.plugin.js
import { separateOperations } from 'graphql';
import {
  fieldExtensionsEstimator,
  getComplexity,
  simpleEstimator,
} from 'graphql-query-complexity';

export const complexityPlugin = {
  async requestDidStart() {
    return {
      async didResolveOperation({ request, document, schema }) {
        const complexity = getComplexity({
          schema,
          operationName: request.operationName,
          query: document,
          variables: request.variables,
          estimators: [
            fieldExtensionsEstimator(),
            simpleEstimator({ defaultComplexity: 1 }),
          ],
        });
        
        if (complexity > 1000) {
          throw new GraphQLError(
            `Query too complex: ${complexity}. Max allowed: 1000`,
            { extensions: { code: 'QUERY_TOO_COMPLEX', complexity } }
          );
        }
        
        // Log complexity ทุก query
        console.log(`Operation: ${request.operationName}, Complexity: ${complexity}`);
      },
    };
  },
};
```

---

## 📌 Step 664: Schema Design สำหรับ Performance

```graphql
# ❌ ไม่ดี — ต้องโหลดข้อมูลมากเกินไป
type Post {
  id: ID!
  author: User!           # ← โหลด user ทุกครั้ง
  comments: [Comment!]!  # ← โหลด comments ทั้งหมด
  likes: [User!]!         # ← โหลด all likers
}

# ✅ ดีกว่า — ใช้ pagination + counts
type Post {
  id: ID!
  authorId: ID!
  author: User!
  
  # Paginated
  comments(first: Int = 10, after: String): CommentConnection!
  
  # Count only (ไม่โหลด all users)
  likeCount: Int!
  
  # ดู likes เฉพาะเมื่อต้องการ
  likes(first: Int = 10): [User!]!
  
  # เพิ่มสำหรับ logged-in user
  isLikedByMe: Boolean!
}
```

---

## 📌 Step 665: Database Query Optimization

```javascript
// src/resolvers/post.resolver.js

// ❌ ไม่ดี — SELECT * ทุกครั้ง
const resolvers = {
  Query: {
    posts: () => Post.findAll(),
  },
};

// ✅ ดีกว่า — Select เฉพาะ fields ที่ต้องการ
const resolvers = {
  Query: {
    posts: async (_, __, ___, info) => {
      // Extract requested fields จาก info.fieldNodes
      const requestedFields = extractFields(info);
      const dbColumns = mapFieldsToColumns(requestedFields);
      
      return Post.findAll({
        attributes: dbColumns,
        limit: 10,
      });
    },
  },
};

// Helper: extract fields from GraphQL info
function extractFields(info) {
  const fieldNodes = info.fieldNodes[0].selectionSet.selections;
  return fieldNodes
    .filter(node => node.kind === 'Field')
    .map(node => node.name.value);
}

// ✅ ดีที่สุด — ใช้ Prisma + select
const resolvers = {
  Query: {
    posts: async (_, { limit = 10, offset = 0 }) => {
      return prisma.post.findMany({
        select: {
          id: true,
          title: true,
          createdAt: true,
          author: {
            select: { id: true, name: true, avatar: true },
          },
        },
        take: limit,
        skip: offset,
        orderBy: { createdAt: 'desc' },
      });
    },
  },
};
```

---

## 📌 Step 666: Connection Pooling

```javascript
// PostgreSQL connection pool
import { Pool } from 'pg';

const pool = new Pool({
  host: process.env.DB_HOST,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  port: parseInt(process.env.DB_PORT, 10),
  
  // Pool configuration
  min: 2,          // minimum connections
  max: 20,         // maximum connections
  idleTimeoutMillis: 30000,    // 30s
  connectionTimeoutMillis: 2000, // 2s timeout
  
  // SSL (production)
  ssl: process.env.NODE_ENV === 'production' ? {
    rejectUnauthorized: true,
    ca: process.env.DB_SSL_CERT,
  } : false,
});

// Monitor pool
pool.on('connect', () => console.log('DB connected'));
pool.on('remove', () => console.log('DB connection removed'));
pool.on('error', (err) => console.error('Pool error:', err));
```

---

## 📌 Step 667: Monitoring Performance

```javascript
// src/plugins/performance.plugin.js
import { performance } from 'perf_hooks';

export const performancePlugin = {
  async requestDidStart({ request }) {
    const startTime = performance.now();
    const operationName = request.operationName || 'anonymous';
    
    return {
      async willSendResponse({ response }) {
        const duration = performance.now() - startTime;
        
        console.log({
          operation: operationName,
          duration: `${duration.toFixed(2)}ms`,
          size: JSON.stringify(response.body).length,
        });
        
        // Slow query warning
        if (duration > 1000) {
          console.warn(`⚠️ Slow query: ${operationName} took ${duration.toFixed(0)}ms`);
        }
        
        // Add timing header
        response.http.headers.set('Server-Timing', `gql;dur=${duration.toFixed(2)}`);
      },
    };
  },
};
```

---

## 📌 Step 668: สรุป Part 020

### เนื้อหาที่เรียนรู้

✅ Query complexity limiting  
✅ Query depth limiting  
✅ Custom complexity rules  
✅ Schema design สำหรับ performance  
✅ Database query optimization  
✅ Connection pooling  
✅ Performance monitoring  

### Homework

1. เพิ่ม complexity + depth limiting ให้ server
2. Profile slow resolvers
3. Optimize N+1 queries ด้วย DataLoader

### ในส่วนถัดไป

➡️ **[Part 021](./part-021.md)** — Apollo Federation: Microservices Architecture

---

*Part 020 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
