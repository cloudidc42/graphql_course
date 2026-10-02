# Part 038 — Audit Logging & Compliance 📋

> **ระดับ:** Advanced | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 571–610

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Audit trail คืออะไร
- GraphQL audit logging
- Sensitive data masking
- GDPR compliance
- Audit log schema
- Log aggregation
- Alert on suspicious activity
- Retention policies

---

## 📌 Step 571: ทำไมต้องมี Audit Log

```
Audit logging สำคัญสำหรับ:

1. Security: ตรวจสอบ unauthorized access
2. Compliance: GDPR, HIPAA, PCI-DSS, SOC2
3. Debugging: reproduce production issues
4. Business: tracking user behavior
5. Legal: evidence ถ้าเกิดคดี

ข้อมูลที่ควร log:
- ใคร (user ID, IP, user agent)
- ทำอะไร (operation, fields accessed)
- เมื่อไหร่ (timestamp)
- ผลลัพธ์ (success/failure)
- Data ที่เปลี่ยน (before/after)
```

---

## 📌 Step 572: Audit Log Schema

```prisma
// prisma/schema.prisma

model AuditLog {
  id          String   @id @default(cuid())
  
  // Who
  userId      String?
  sessionId   String?
  ipAddress   String
  userAgent   String?
  
  // What
  action      String   // graphql.query, graphql.mutation, auth.login
  resource    String?  // User, Post, Order
  resourceId  String?
  
  // Operation
  operationName String?
  query         String?  // redacted
  variables     Json?    // redacted
  
  // Result
  success     Boolean
  errorCode   String?
  
  // Changes (for mutations)
  before      Json?
  after       Json?
  
  // When
  timestamp   DateTime @default(now())
  duration    Int?     // milliseconds
  
  @@index([userId, timestamp])
  @@index([resource, resourceId])
  @@index([action, timestamp])
  @@index([ipAddress, timestamp])
}
```

---

## 📌 Step 573: Audit Log Plugin

```javascript
// src/plugins/audit.plugin.js
import { prisma } from '../db.js';
import { logger } from '../logger.js';

const SENSITIVE_FIELDS = ['password', 'token', 'secret', 'creditCard', 'ssn'];
const SENSITIVE_OPERATIONS = ['login', 'register', 'changePassword', 'deleteAccount'];

function redactSensitive(obj) {
  if (!obj || typeof obj !== 'object') return obj;
  
  const result = {};
  for (const [key, value] of Object.entries(obj)) {
    if (SENSITIVE_FIELDS.some(f => key.toLowerCase().includes(f))) {
      result[key] = '[REDACTED]';
    } else if (typeof value === 'object') {
      result[key] = redactSensitive(value);
    } else {
      result[key] = value;
    }
  }
  return result;
}

export const auditPlugin = {
  async requestDidStart({ request, contextValue }) {
    const startTime = Date.now();
    
    return {
      async willSendResponse({ response, contextValue }) {
        const duration = Date.now() - startTime;
        const { user, req } = contextValue;
        
        const hasErrors = !!response.body.singleResult?.errors?.length;
        const operationName = request.operationName;
        
        // ไม่ log introspection queries
        if (operationName === 'IntrospectionQuery') return;
        
        // Detect operation type
        const isMutation = request.query?.trim().startsWith('mutation');
        const isQuery = !isMutation;
        
        const auditEntry = {
          userId: user?.sub || null,
          ipAddress: req.ip || req.connection?.remoteAddress,
          userAgent: req.headers['user-agent'],
          action: isMutation ? 'graphql.mutation' : 'graphql.query',
          operationName,
          query: request.query?.slice(0, 500), // limit length
          variables: redactSensitive(request.variables),
          success: !hasErrors,
          errorCode: response.body.singleResult?.errors?.[0]?.extensions?.code,
          duration,
        };
        
        // Log ไป database (async, ไม่รอ)
        prisma.auditLog.create({ data: auditEntry }).catch(err => {
          logger.error({ err }, 'Failed to write audit log');
        });
        
        // Log ไป structured logger
        logger.info(auditEntry, 'GraphQL operation');
        
        // Alert สำหรับ sensitive operations
        if (isMutation && SENSITIVE_OPERATIONS.includes(operationName) && hasErrors) {
          logger.warn({ ...auditEntry }, `Sensitive operation failed: ${operationName}`);
        }
      },
    };
  },
};
```

---

## 📌 Step 574: Resolver-level Audit

```javascript
// src/middleware/auditMiddleware.js
// Audit specific field access

export function withAuditField(resolver, { resource, action }) {
  return async function auditedResolver(parent, args, context, info) {
    const result = await resolver(parent, args, context, info);
    
    if (result !== null && result !== undefined) {
      // Log sensitive field access
      await prisma.auditLog.create({
        data: {
          userId: context.user?.sub,
          ipAddress: context.req?.ip,
          action: `field.${resource}.${action}`,
          resource,
          resourceId: parent?.id,
          success: true,
        },
      }).catch(() => {}); // ไม่ fail resolver ถ้า audit log fail
    }
    
    return result;
  };
}

// ใช้งาน
const resolvers = {
  User: {
    // Log ทุกครั้งที่มีการอ่าน email
    email: withAuditField(
      (user) => user.email,
      { resource: 'User', action: 'read.email' }
    ),
    
    // Log ทุกครั้งที่มีการอ่าน phone
    phone: withAuditField(
      (user) => user.phone,
      { resource: 'User', action: 'read.phone' }
    ),
  },
};
```

---

## 📌 Step 575: Data Change Tracking

```javascript
// ติดตาม before/after สำหรับ mutations

const resolvers = {
  Mutation: {
    updateUser: async (_, { id, input }, { prisma, user }) => {
      // Get before state
      const before = await prisma.user.findUnique({
        where: { id },
        select: { name: true, email: true, role: true },
      });
      
      // Make change
      const after = await prisma.user.update({
        where: { id },
        data: input,
      });
      
      // Log the change
      await prisma.auditLog.create({
        data: {
          userId: user.sub,
          action: 'user.update',
          resource: 'User',
          resourceId: id,
          before: redactSensitive(before),
          after: redactSensitive({
            name: after.name,
            email: after.email,
            role: after.role,
          }),
          success: true,
        },
      });
      
      return after;
    },
  },
};
```

---

## 📌 Step 576: GDPR Compliance

```javascript
// GDPR: User มีสิทธิ์ดู, แก้ไข, และลบข้อมูลของตัวเอง

const resolvers = {
  Query: {
    // ดาวน์โหลดข้อมูลทั้งหมด (GDPR Right to Access)
    exportMyData: async (_, __, { user, prisma }) => {
      if (!user) throw new GraphQLError('Unauthenticated');
      
      const [userData, orders, posts, auditLogs] = await Promise.all([
        prisma.user.findUnique({ where: { id: user.sub } }),
        prisma.order.findMany({ where: { customerId: user.sub } }),
        prisma.post.findMany({ where: { authorId: user.sub } }),
        prisma.auditLog.findMany({
          where: { userId: user.sub },
          orderBy: { timestamp: 'desc' },
          take: 1000,
        }),
      ]);
      
      return {
        profile: userData,
        orders,
        posts,
        activityLog: auditLogs,
        exportedAt: new Date().toISOString(),
      };
    },
  },
  
  Mutation: {
    // ลบข้อมูลทั้งหมด (GDPR Right to Erasure)
    deleteMyAccount: async (_, { confirmPassword }, { user, prisma }) => {
      if (!user) throw new GraphQLError('Unauthenticated');
      
      // Verify password
      const dbUser = await prisma.user.findUnique({ where: { id: user.sub } });
      const valid = await bcrypt.compare(confirmPassword, dbUser.password);
      if (!valid) throw new GraphQLError('Invalid password');
      
      // Anonymize ข้อมูลแทนการลบ (ถ้ามี FK constraints)
      await prisma.user.update({
        where: { id: user.sub },
        data: {
          name: 'Deleted User',
          email: `deleted-${user.sub}@deleted.com`,
          phone: null,
          avatar: null,
          deletedAt: new Date(),
          isAnonymized: true,
        },
      });
      
      // Log the deletion
      await prisma.auditLog.create({
        data: {
          userId: user.sub,
          action: 'user.gdpr.delete',
          resource: 'User',
          resourceId: user.sub,
          success: true,
        },
      });
      
      return true;
    },
  },
};
```

---

## 📌 Step 577: สรุป Part 038

### เนื้อหาที่เรียนรู้

✅ Audit log schema  
✅ GraphQL audit plugin  
✅ Sensitive data redaction  
✅ Field-level audit  
✅ Before/after change tracking  
✅ GDPR compliance (access + erasure)  

### Compliance Checklist

```
✅ บันทึก who, what, when, result
✅ Redact passwords, tokens, sensitive data
✅ Retain logs ตาม policy (เช่น 1 ปี)
✅ ทำ GDPR data export endpoint
✅ ทำ account anonymization (ไม่ใช่ hard delete)
✅ Alert suspicious patterns
✅ Audit log ต้องไม่แก้ไขได้ (append-only)
```

### ในส่วนถัดไป

➡️ **[Part 039](./part-039.md)** — GraphQL Penetration Testing Methodology

---

*Part 038 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
