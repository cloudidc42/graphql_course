# Part 036 — Advanced RBAC & Permission Systems 🔐

> **ระดับ:** Advanced | **เวลาเรียน:** 120 นาที | **ขั้นตอนที่:** 491–530

---

## 🎯 สิ่งที่จะได้เรียนรู้

- RBAC (Role-Based Access Control)
- ABAC (Attribute-Based Access Control)
- Permission Policies
- GraphQL Shield
- Row-level security
- Field-level permissions
- Multi-tenant permissions
- Permission caching

---

## 📌 Step 491: RBAC Basics

```
Roles → Permissions → Resources

ตัวอย่าง:
ADMIN    → ทุกอย่าง
SELLER   → จัดการ products, orders ของตัวเอง
BUYER    → สั่งซื้อ, ดูคำสั่งตัวเอง
MODERATOR→ ลบ content
GUEST    → ดูเฉพาะ public content
```

---

## 📌 Step 492: Role & Permission Schema

```javascript
// Prisma schema
// model User {
//   id    String   @id @default(cuid())
//   roles UserRole[]
// }
//
// model UserRole {
//   userId  String
//   roleId  String
//   grantedAt DateTime
//   grantedBy String?
//   expiresAt DateTime?
// }
//
// model Role {
//   id          String       @id
//   name        String       @unique
//   permissions Permission[]
// }
//
// model Permission {
//   id      String @id
//   action  String   // read, write, delete
//   subject String   // Post, User, Order
//   roleId  String
// }

// src/auth/permissions.js
export const ROLES = {
  ADMIN: 'ADMIN',
  SELLER: 'SELLER',
  BUYER: 'BUYER',
  MODERATOR: 'MODERATOR',
};

export const PERMISSIONS = {
  // Users
  'user:read:own': ['BUYER', 'SELLER', 'MODERATOR', 'ADMIN'],
  'user:read:any': ['ADMIN', 'MODERATOR'],
  'user:update:own': ['BUYER', 'SELLER', 'ADMIN'],
  'user:update:any': ['ADMIN'],
  'user:delete:any': ['ADMIN'],
  
  // Products
  'product:read': ['BUYER', 'SELLER', 'MODERATOR', 'ADMIN'],
  'product:create': ['SELLER', 'ADMIN'],
  'product:update:own': ['SELLER', 'ADMIN'],
  'product:update:any': ['ADMIN'],
  'product:delete:own': ['SELLER', 'ADMIN'],
  'product:delete:any': ['ADMIN'],
  
  // Orders
  'order:read:own': ['BUYER', 'SELLER', 'ADMIN'],
  'order:read:any': ['ADMIN', 'MODERATOR'],
  'order:update:own': ['SELLER', 'ADMIN'],
  
  // Admin
  'admin:access': ['ADMIN'],
  'analytics:view': ['ADMIN', 'MODERATOR'],
};

// ตรวจสอบสิทธิ์
export function hasPermission(userRoles, permission) {
  const allowedRoles = PERMISSIONS[permission];
  if (!allowedRoles) return false;
  return userRoles.some(role => allowedRoles.includes(role));
}
```

---

## 📌 Step 493: GraphQL Shield

```bash
npm install graphql-shield
```

```javascript
// src/auth/shield.js
import { shield, rule, and, or, not, allow, deny } from 'graphql-shield';
import { GraphQLError } from 'graphql';

// Rules
const isAuthenticated = rule({ cache: 'contextual' })(
  (parent, args, ctx) => {
    if (!ctx.user) {
      throw new GraphQLError('Authentication required', {
        extensions: { code: 'UNAUTHENTICATED' }
      });
    }
    return true;
  }
);

const isAdmin = rule({ cache: 'contextual' })(
  (parent, args, ctx) => {
    return ctx.user?.roles?.includes('ADMIN') || false;
  }
);

const isSeller = rule({ cache: 'contextual' })(
  (parent, args, ctx) => {
    return ctx.user?.roles?.includes('SELLER') || false;
  }
);

const isOwner = rule({ cache: 'strict' })(
  async (parent, args, ctx, info) => {
    const userId = ctx.user?.sub;
    if (!userId) return false;
    
    // ดูจาก parent (ถ้ามี)
    if (parent?.userId === userId) return true;
    if (parent?.authorId === userId) return true;
    
    // ดูจาก args
    if (args.userId === userId) return true;
    
    return false;
  }
);

const isProductOwner = rule({ cache: 'strict' })(
  async (parent, args, ctx) => {
    const productId = args.id || args.productId;
    if (!productId) return false;
    
    const product = await ctx.prisma.product.findUnique({
      where: { id: productId },
      select: { sellerId: true },
    });
    
    return product?.sellerId === ctx.user?.sub;
  }
);

// Permissions map
export const permissions = shield({
  Query: {
    me: isAuthenticated,
    users: isAdmin,
    user: or(isOwner, isAdmin),
    
    products: allow,           // Public
    product: allow,            // Public
    myProducts: and(isAuthenticated, isSeller),
    
    orders: isAdmin,
    myOrders: isAuthenticated,
    order: and(isAuthenticated, or(isOwner, isAdmin)),
    
    analytics: and(isAuthenticated, isAdmin),
  },
  
  Mutation: {
    updateProfile: and(isAuthenticated, isOwner),
    
    createProduct: and(isAuthenticated, isSeller),
    updateProduct: and(isAuthenticated, or(isProductOwner, isAdmin)),
    deleteProduct: and(isAuthenticated, or(isProductOwner, isAdmin)),
    
    checkout: isAuthenticated,
    cancelOrder: and(isAuthenticated, or(isOwner, isAdmin)),
    
    // Admin only
    deleteUser: and(isAuthenticated, isAdmin),
    updateOrderStatus: and(isAuthenticated, isAdmin),
  },
  
  // Field-level permissions
  User: {
    email: or(isOwner, isAdmin),
    phone: or(isOwner, isAdmin),
    address: or(isOwner, isAdmin),
    createdAt: isAdmin,
  },
  
  Order: {
    paymentDetails: or(isOwner, isAdmin),
    billingAddress: or(isOwner, isAdmin),
  },
}, {
  allowExternalErrors: true,
  fallbackError: new GraphQLError('Not authorized', {
    extensions: { code: 'FORBIDDEN' }
  }),
  fallbackRule: allow,
});
```

---

## 📌 Step 494: ใช้ Shield ใน Server

```javascript
// src/server.js
import { makeExecutableSchema } from '@graphql-tools/schema';
import { applyMiddleware } from 'graphql-middleware';
import { permissions } from './auth/shield.js';

const schema = makeExecutableSchema({ typeDefs, resolvers });

// Apply Shield middleware
const schemaWithPermissions = applyMiddleware(schema, permissions);

const server = new ApolloServer({
  schema: schemaWithPermissions,
  // ...
});
```

---

## 📌 Step 495: Row-Level Security

```javascript
// ป้องกัน user อ่าน data ของ user อื่น

// ✅ ใน resolver
const resolvers = {
  Query: {
    order: async (_, { id }, { user, prisma }) => {
      const order = await prisma.order.findUnique({ where: { id } });
      
      if (!order) throw new GraphQLError('Order not found', {
        extensions: { code: 'NOT_FOUND' }
      });
      
      // Row-level check
      if (order.customerId !== user.sub && !user.roles.includes('ADMIN')) {
        throw new GraphQLError('Access denied', {
          extensions: { code: 'FORBIDDEN' }
        });
      }
      
      return order;
    },
    
    // ✅ ดีกว่า: กรองใน query (ป้องกัน timing attack)
    orders: async (_, { filter }, { user, prisma }) => {
      const where = user.roles.includes('ADMIN')
        ? filter  // Admin เห็นทุกอย่าง
        : { ...filter, customerId: user.sub }; // User เห็นแค่ของตัวเอง
      
      return prisma.order.findMany({ where });
    },
  },
};
```

---

## 📌 Step 496: ABAC (Attribute-Based Access Control)

```javascript
// ABAC ยืดหยุ่นกว่า RBAC: ตัดสินจาก attributes

// Policies
const policies = {
  canViewProduct: ({ user, resource }) => {
    if (resource.status === 'PUBLISHED') return true;
    if (resource.sellerId === user?.sub) return true;
    if (user?.roles.includes('ADMIN')) return true;
    return false;
  },
  
  canEditProduct: ({ user, resource }) => {
    if (!user) return false;
    if (user.roles.includes('ADMIN')) return true;
    if (resource.sellerId === user.sub) {
      // Seller สามารถแก้ไขได้ยกเว้น order ที่ pending
      return resource.pendingOrders === 0;
    }
    return false;
  },
  
  canViewOrder: ({ user, resource }) => {
    if (!user) return false;
    if (user.roles.includes('ADMIN')) return true;
    if (user.roles.includes('MODERATOR') && resource.hasDispute) return true;
    return resource.customerId === user.sub || resource.sellerId === user.sub;
  },
};

// ใช้ใน resolver
const resolvers = {
  Query: {
    product: async (_, { id }, { user, prisma }) => {
      const product = await prisma.product.findUnique({ where: { id } });
      
      if (!policies.canViewProduct({ user, resource: product })) {
        throw new GraphQLError('Access denied', { extensions: { code: 'FORBIDDEN' } });
      }
      
      return product;
    },
  },
};
```

---

## 📌 Step 497: Permission Caching

```javascript
// Cache permission checks เพื่อ performance

const resolvers = {
  Query: {
    orders: async (_, args, ctx) => {
      // Cache permission check ใน context (ต่อ request)
      if (!ctx._cache) ctx._cache = {};
      
      const cacheKey = `canListOrders:${ctx.user?.sub}`;
      
      if (!(cacheKey in ctx._cache)) {
        ctx._cache[cacheKey] = await checkPermission(ctx, 'order:read:any');
      }
      
      const canReadAny = ctx._cache[cacheKey];
      // ...
    },
  },
};
```

---

## 📌 Step 498: สรุป Part 036

### เนื้อหาที่เรียนรู้

✅ RBAC permission matrix  
✅ GraphQL Shield  
✅ Rule caching  
✅ Row-level security  
✅ ABAC policies  
✅ Permission caching  

### ในส่วนถัดไป

➡️ **[Part 037](./part-037.md)** — Advanced Batching & DataLoader Patterns

---

*Part 036 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
