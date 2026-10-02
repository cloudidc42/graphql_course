# Part 069 — Advanced GraphQL Security 🛡️

> **ระดับ:** Expert | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1661–1700

---

## 🎯 สิ่งที่จะได้เรียนรู้

- GraphQL Injection attacks & defenses
- Introspection attacks & disabling
- Query depth & complexity attacks
- Field suggestion leakage
- CSRF สำหรับ GraphQL
- Persisted queries (security)
- Authorization bypass patterns
- Security testing checklist

> ⚠️ **หมายเหตุ:** เนื้อหานี้เพื่อ **การศึกษาด้านป้องกัน** เท่านั้น
> การนำเทคนิคเหล่านี้ไปใช้กับระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

---

## 📌 Step 1661: Introspection Attack Prevention

```typescript
// ใน production: ปิด introspection
const server = new ApolloServer({
  typeDefs,
  resolvers,
  introspection: process.env.NODE_ENV !== 'production',
  
  // หรือ custom plugin สำหรับ whitelist
  plugins: [
    {
      async requestDidStart({ request }) {
        if (request.operationName === 'IntrospectionQuery') {
          const ip = request.http?.headers.get('x-forwarded-for');
          const isInternal = isInternalIP(ip ?? '');
          
          if (!isInternal && process.env.NODE_ENV === 'production') {
            throw new GraphQLError('Introspection disabled', {
              extensions: { code: 'INTROSPECTION_DISABLED' },
            });
          }
        }
      },
    },
  ],
});

// ซ่อน field suggestions
import { ApolloServerPluginDisableSchemaChange } from '@apollo/server/plugin/disableSchemaChange';

const server = new ApolloServer({
  plugins: [ApolloServerPluginDisableSchemaChange()],
  
  // ปิด did-you-mean suggestions
  formatError: (error) => {
    // Remove "Did you mean X?" from error messages
    return {
      ...error,
      message: error.message.replace(/Did you mean .+\?/g, ''),
      locations: undefined,   // ซ่อน line numbers
      path: error.path,
    };
  },
});
```

---

## 📌 Step 1662: Query Depth & Complexity Limits

```typescript
// npm install graphql-depth-limit graphql-validation-complexity
import depthLimit from 'graphql-depth-limit';
import { createComplexityLimitRule } from 'graphql-validation-complexity';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    // จำกัด depth สูงสุด
    depthLimit(10),
    
    // จำกัด complexity
    createComplexityLimitRule(5000, {
      // Custom complexity per field
      fieldConfigEstimator: () => 1,
      scalarCost: 1,
      objectCost: 2,
      listFactor: 10,    // list fields แพงกว่า
      
      onCost: (cost) => {
        console.log('Query cost:', cost);
      },
    }),
  ],
});

// Manual complexity calculation
function calculateComplexity(query: string): number {
  const doc = parse(query);
  let complexity = 0;
  
  visit(doc, {
    Field: {
      enter(node, key, parent, path, ancestors) {
        const depth = ancestors.filter(
          a => a?.kind === 'SelectionSet'
        ).length;
        
        complexity += Math.pow(2, depth);
      },
    },
    Argument: {
      enter(node) {
        // first/limit argument = potential amplification
        if (node.name.value === 'first' || node.name.value === 'limit') {
          const value = (node.value as IntValueNode).value;
          complexity += parseInt(value) * 0.1;
        }
      },
    },
  });
  
  return complexity;
}

// Plugin สำหรับ block expensive queries
const complexityPlugin: ApolloServerPlugin = {
  async requestDidStart({ request }) {
    if (!request.query) return {};
    
    const cost = calculateComplexity(request.query);
    
    if (cost > 1000) {
      throw new GraphQLError(`Query too complex (score: ${cost})`, {
        extensions: { code: 'QUERY_TOO_COMPLEX', cost },
      });
    }
    
    return {};
  },
};
```

---

## 📌 Step 1663: GraphQL Injection Prevention

```typescript
// SQL Injection via GraphQL
// ❌ NEVER: string interpolation ใน raw queries
const badResolver = async (_, { search }) => {
  // SQL INJECTION!
  return db.raw(`SELECT * FROM products WHERE name LIKE '%${search}%'`);
};

// ✅ GOOD: parameterized queries เสมอ
const goodResolver = async (_, { search }) => {
  return db.raw('SELECT * FROM products WHERE name LIKE ?', [`%${search}%`]);
};

// ✅ BETTER: ORM (Prisma)
const bestResolver = async (_, { search }) => {
  return prisma.product.findMany({
    where: { name: { contains: search, mode: 'insensitive' } },
  });
};

// ReDoS (Regular Expression Denial of Service)
// ❌ ระวัง regex ที่ซับซ้อนกับ user input
const badEmailRegex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;

// ✅ ใช้ library ที่ตรวจสอบแล้ว
import validator from 'validator';
const isEmail = (email: string) => validator.isEmail(email);

// SSRF (Server-Side Request Forgery)
// ❌ อย่า fetch URL จาก user input โดยตรง
const badFetch = async (_, { imageUrl }) => {
  return fetch(imageUrl).then(r => r.buffer());  // SSRF!
};

// ✅ Validate & allowlist URLs
const ALLOWED_DOMAINS = ['cdn.myapp.com', 'images.myapp.com'];

const goodFetch = async (_, { imageUrl }) => {
  const url = new URL(imageUrl);
  if (!ALLOWED_DOMAINS.includes(url.hostname)) {
    throw new GraphQLError('Domain not allowed');
  }
  return fetch(imageUrl).then(r => r.buffer());
};
```

---

## 📌 Step 1664: Authorization Security Patterns

```typescript
// Object-level authorization (BOLA/IDOR prevention)

// ❌ ไม่มี authorization check
const badResolver = async (_, { orderId }, ctx) => {
  return ctx.prisma.order.findUnique({ where: { id: orderId } });
};

// ✅ ตรวจสอบ ownership เสมอ
const goodResolver = async (_, { orderId }, ctx) => {
  requireAuth(ctx);
  
  const order = await ctx.prisma.order.findFirst({
    where: {
      id: orderId,
      // Ownership check: ต้องเป็น order ของ user นี้ หรือ admin
      OR: [
        { customerId: ctx.user!.id },
        { customer: { id: ctx.user!.id } },
      ],
    },
  });
  
  if (!order && ctx.user!.role !== 'ADMIN') {
    throw new GraphQLError('Order not found', {
      extensions: { code: 'NOT_FOUND' },
    });
  }
  
  return order;
};

// Field-level authorization
const resolvers = {
  User: {
    email: (user, _, ctx) => {
      // Only show email to: owner, admin
      if (ctx.user?.id !== user.id && ctx.user?.role !== 'ADMIN') {
        return null;  // หรือ mask: u***@domain.com
      }
      return user.email;
    },
    
    // Internal field: admin only
    internalNotes: (user, _, ctx) => {
      if (ctx.user?.role !== 'ADMIN') {
        throw new ForbiddenError('Not authorized');
      }
      return user.internalNotes;
    },
  },
};

// Batch authorization (avoid N+1 checks)
const orderLoader = new DataLoader(async (orderIds: readonly string[]) => {
  const orders = await prisma.order.findMany({
    where: {
      id: { in: [...orderIds] },
      // Apply auth at batch level
      customerId: currentUserId,  // from context closure
    },
  });
  
  return orderIds.map(id => orders.find(o => o.id === id) ?? null);
});
```

---

## 📌 Step 1665: CSRF & Persisted Queries Security

```typescript
// CSRF Protection สำหรับ GraphQL

// GraphQL over HTTP: CSRF attack ทำได้ถ้า:
// 1. ใช้ cookies สำหรับ auth
// 2. Server accept application/x-www-form-urlencoded หรือ text/plain

// ✅ ใช้ Bearer token แทน cookies (CSRF-safe)
// ✅ ถ้าใช้ cookies: ต้องมี CSRF token หรือ SameSite=Strict

// ✅ Content-Type check
app.use('/graphql', (req, res, next) => {
  const contentType = req.headers['content-type'] ?? '';
  
  // GraphQL only accepts: application/json, multipart/form-data
  if (!contentType.startsWith('application/json') && 
      !contentType.startsWith('multipart/form-data')) {
    return res.status(415).json({
      errors: [{ message: 'Unsupported Media Type' }],
    });
  }
  
  next();
});

// ✅ Persisted Queries สำหรับ security
// แทนที่จะรับ arbitrary queries → รับเฉพาะ registered queries
import { createPersistedQueryPlugin } from '@apollo/server-plugin-query-cache';

const server = new ApolloServer({
  plugins: [
    createPersistedQueryPlugin({
      cache: redisCache,
      
      // ใน production: ไม่อนุญาต ad-hoc queries
      async resolveQueryFromHash(queryHash, context) {
        const query = await redisCache.get(`pq:${queryHash}`);
        if (!query && process.env.NODE_ENV === 'production') {
          throw new GraphQLError('Only registered queries are allowed', {
            extensions: { code: 'PERSISTED_QUERY_NOT_FOUND' },
          });
        }
        return query;
      },
    }),
  ],
});
```

---

## 📌 Step 1666: Security Testing Checklist

```
GraphQL Security Audit Checklist:

Authentication:
□ ทุก mutation ต้องการ auth
□ ทุก query ที่มี private data ต้องการ auth
□ Token expiry ทำงานถูกต้อง
□ Invalid tokens ถูก reject
□ Rate limiting บน login

Authorization:
□ Object-level: ดึง resource ของคนอื่นไม่ได้
□ Field-level: fields ที่ sensitive ซ่อนอยู่
□ Role-based: admin features ต้องการ admin role
□ Horizontal privilege escalation ทำไม่ได้
□ Vertical privilege escalation ทำไม่ได้

Input Validation:
□ SQL injection ทุก field ที่เข้า DB
□ XSS: sanitize text ก่อน store/display
□ SSRF: validate URLs ใน inputs
□ ReDoS: ตรวจสอบ regex patterns

DoS Prevention:
□ Query depth limit
□ Query complexity limit
□ Rate limiting (per IP, per user)
□ Timeout สำหรับ long-running queries
□ Request size limit

Information Disclosure:
□ Introspection disabled ใน production
□ Error messages ไม่ leak stack traces
□ Field suggestions ปิด ใน production
□ Server version ไม่ expose ใน headers
□ Debug info ไม่แสดงใน prod

Infrastructure:
□ HTTPS only
□ CORS configured correctly
□ Content-Security-Policy headers
□ Dependencies up-to-date (no known vulnerabilities)
```

---

## 📌 Step 1667: สรุป Part 069

### เนื้อหาที่เรียนรู้

✅ Introspection attack prevention  
✅ Query depth & complexity limits  
✅ SQL injection prevention  
✅ SSRF prevention  
✅ Authorization security (IDOR/BOLA)  
✅ CSRF protection  
✅ Persisted queries security  
✅ Security audit checklist  

### ในส่วนถัดไป

➡️ **[Part 070](./part-070.md)** — Production-Grade Error Handling

---

*Part 069 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
