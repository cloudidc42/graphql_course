# Part 052 — Batch Operations & Bulk Mutations ⚡

> **ระดับ:** Advanced | **เวลาเรียน:** 75 นาที | **ขั้นตอนที่:** 991–1030

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Bulk create/update/delete mutations
- Upsert operations
- Transaction handling
- Partial success patterns
- Import/export via GraphQL
- Batch result types
- Optimistic locking
- Idempotent mutations

---

## 📌 Step 991: Bulk Mutation Schema

```graphql
# src/schema/bulk.graphql

type Mutation {
  # Bulk create
  createProducts(inputs: [CreateProductInput!]!): BulkCreateResult!
  
  # Bulk update
  updateProducts(updates: [ProductUpdate!]!): BulkUpdateResult!
  
  # Bulk delete
  deleteProducts(ids: [ID!]!): BulkDeleteResult!
  
  # Upsert
  upsertProduct(input: UpsertProductInput!): Product!
  upsertProducts(inputs: [UpsertProductInput!]!): BulkUpsertResult!
  
  # Import from CSV/JSON
  importProducts(data: [ProductImportRow!]!): ImportResult!
}

input ProductUpdate {
  id: ID!
  data: UpdateProductData!
}

input UpsertProductInput {
  id: ID          # ถ้ามี id = update, ไม่มี = create
  externalId: String  # alternative key
  name: String!
  price: Float!
  categoryId: ID!
  stock: Int!
}

input ProductImportRow {
  externalId: String!
  name: String!
  price: Float!
  category: String!
  stock: Int!
}

# Bulk result types
type BulkCreateResult {
  succeeded: [Product!]!
  failed: [BulkError!]!
  successCount: Int!
  failureCount: Int!
}

type BulkUpdateResult {
  succeeded: [Product!]!
  failed: [BulkError!]!
  successCount: Int!
  failureCount: Int!
}

type BulkDeleteResult {
  deletedIds: [ID!]!
  failedIds: [ID!]!
  successCount: Int!
  failureCount: Int!
}

type BulkUpsertResult {
  created: [Product!]!
  updated: [Product!]!
  failed: [BulkError!]!
}

type BulkError {
  index: Int!     # index ใน input array
  id: ID          # id ที่ fail (ถ้ามี)
  message: String!
  code: String!
}

type ImportResult {
  imported: Int!
  skipped: Int!
  errors: [ImportError!]!
  preview: [Product!]!
}

type ImportError {
  row: Int!
  column: String
  message: String!
}
```

---

## 📌 Step 992: Bulk Create Resolver

```typescript
// src/resolvers/bulk.resolver.ts
import { GraphQLError } from 'graphql';

const bulkResolvers = {
  Mutation: {
    createProducts: async (_, { inputs }, ctx) => {
      requireAuth(ctx);
      requireRole(ctx, 'ADMIN', 'SELLER');
      
      // Validate limit
      if (inputs.length > 500) {
        throw new GraphQLError('Maximum 500 items per bulk operation', {
          extensions: { code: 'BAD_USER_INPUT' }
        });
      }
      
      const succeeded: Product[] = [];
      const failed: BulkError[] = [];
      
      // ใช้ transaction สำหรับ all-or-nothing
      // หรือ per-item error สำหรับ partial success
      
      // Option A: Partial success (แต่ละ item fail แยก)
      for (let i = 0; i < inputs.length; i++) {
        try {
          const input = inputs[i];
          
          // Validate
          const validated = CreateProductSchema.parse(input);
          
          // Check duplicate
          const existing = await ctx.prisma.product.findFirst({
            where: { name: validated.name, categoryId: validated.categoryId }
          });
          if (existing) {
            failed.push({
              index: i,
              id: null,
              message: `Product "${validated.name}" already exists in this category`,
              code: 'DUPLICATE_PRODUCT',
            });
            continue;
          }
          
          const product = await ctx.prisma.product.create({
            data: { ...validated, sellerId: ctx.user!.sub }
          });
          succeeded.push(product);
          
        } catch (error) {
          failed.push({
            index: i,
            id: null,
            message: error instanceof Error ? error.message : 'Unknown error',
            code: 'CREATE_FAILED',
          });
        }
      }
      
      return {
        succeeded,
        failed,
        successCount: succeeded.length,
        failureCount: failed.length,
      };
    },
    
    // Option B: All-or-nothing transaction
    createProductsAtomic: async (_, { inputs }, ctx) => {
      requireAuth(ctx);
      
      if (inputs.length > 100) {
        throw new GraphQLError('Maximum 100 items for atomic bulk create');
      }
      
      // Validate all first
      const validated = inputs.map((input: unknown, i: number) => {
        try {
          return CreateProductSchema.parse(input);
        } catch (error) {
          throw new GraphQLError(`Validation failed at index ${i}: ${(error as Error).message}`);
        }
      });
      
      // Single transaction
      const products = await ctx.prisma.$transaction(
        validated.map((data: { name: string; price: number; categoryId: string; stock: number }) =>
          ctx.prisma.product.create({ data })
        )
      );
      
      return {
        succeeded: products,
        failed: [],
        successCount: products.length,
        failureCount: 0,
      };
    },
    
    updateProducts: async (_, { updates }, ctx) => {
      requireAuth(ctx);
      
      if (updates.length > 200) {
        throw new GraphQLError('Maximum 200 items per bulk update');
      }
      
      const succeeded: Product[] = [];
      const failed: BulkError[] = [];
      
      // ตรวจสอบ ownership ก่อน (ป้องกัน IDOR)
      const ids = updates.map((u: { id: string }) => u.id);
      const existingProducts = await ctx.prisma.product.findMany({
        where: { id: { in: ids } },
        select: { id: true, sellerId: true },
      });
      
      const productMap = new Map(existingProducts.map(p => [p.id, p]));
      
      for (let i = 0; i < updates.length; i++) {
        const { id, data } = updates[i];
        const product = productMap.get(id);
        
        if (!product) {
          failed.push({ index: i, id, message: 'Product not found', code: 'NOT_FOUND' });
          continue;
        }
        
        // ตรวจสอบ ownership
        if (ctx.user!.role !== 'ADMIN' && product.sellerId !== ctx.user!.sub) {
          failed.push({ index: i, id, message: 'Unauthorized', code: 'UNAUTHORIZED' });
          continue;
        }
        
        try {
          const updated = await ctx.prisma.product.update({
            where: { id },
            data,
          });
          succeeded.push(updated);
        } catch (error) {
          failed.push({
            index: i, id,
            message: error instanceof Error ? error.message : 'Update failed',
            code: 'UPDATE_FAILED',
          });
        }
      }
      
      return { succeeded, failed, successCount: succeeded.length, failureCount: failed.length };
    },
    
    deleteProducts: async (_, { ids }, ctx) => {
      requireAuth(ctx);
      requireRole(ctx, 'ADMIN', 'SELLER');
      
      if (ids.length > 200) throw new GraphQLError('Maximum 200 items per bulk delete');
      
      // Check ownership
      const products = await ctx.prisma.product.findMany({
        where: { id: { in: ids } },
        select: { id: true, sellerId: true },
      });
      
      const authorizedIds = ctx.user!.role === 'ADMIN'
        ? ids
        : products.filter(p => p.sellerId === ctx.user!.sub).map(p => p.id);
      
      const unauthorizedIds = ids.filter(id => !authorizedIds.includes(id));
      
      await ctx.prisma.product.deleteMany({
        where: { id: { in: authorizedIds } },
      });
      
      return {
        deletedIds: authorizedIds,
        failedIds: unauthorizedIds,
        successCount: authorizedIds.length,
        failureCount: unauthorizedIds.length,
      };
    },
  },
};
```

---

## 📌 Step 993: Upsert Operations

```typescript
// Upsert: create ถ้าไม่มี, update ถ้ามี

upsertProduct: async (_, { input }, ctx) => {
  requireAuth(ctx);
  
  // ค้นหาจาก id หรือ externalId
  const where = input.id
    ? { id: input.id }
    : { externalId: input.externalId };
  
  return ctx.prisma.product.upsert({
    where: where as Prisma.ProductWhereUniqueInput,
    create: {
      ...input,
      sellerId: ctx.user!.sub,
    },
    update: {
      name: input.name,
      price: input.price,
      stock: input.stock,
      updatedAt: new Date(),
    },
  });
},

// Bulk upsert ด้วย createMany + updateMany
upsertProducts: async (_, { inputs }, ctx) => {
  requireAuth(ctx);
  
  if (inputs.length > 500) throw new GraphQLError('Maximum 500 items');
  
  const created: Product[] = [];
  const updated: Product[] = [];
  const failed: BulkError[] = [];
  
  // แยก items เป็น create กับ update groups
  const externalIds = inputs
    .filter((i: UpsertProductInput) => i.externalId)
    .map((i: UpsertProductInput) => i.externalId!);
  
  const existingProducts = await ctx.prisma.product.findMany({
    where: { externalId: { in: externalIds } },
    select: { id: true, externalId: true },
  });
  
  const existingMap = new Map(existingProducts.map(p => [p.externalId, p.id]));
  
  for (let i = 0; i < inputs.length; i++) {
    const input = inputs[i];
    
    try {
      const existingId = input.id ?? existingMap.get(input.externalId);
      
      if (existingId) {
        // Update
        const product = await ctx.prisma.product.update({
          where: { id: existingId },
          data: { name: input.name, price: input.price, stock: input.stock },
        });
        updated.push(product);
      } else {
        // Create
        const product = await ctx.prisma.product.create({
          data: { ...input, sellerId: ctx.user!.sub },
        });
        created.push(product);
      }
    } catch (error) {
      failed.push({
        index: i, id: input.id,
        message: error instanceof Error ? error.message : 'Upsert failed',
        code: 'UPSERT_FAILED',
      });
    }
  }
  
  return { created, updated, failed };
},
```

---

## 📌 Step 994: Import สำหรับ Products

```typescript
importProducts: async (_, { data }, ctx) => {
  requireAuth(ctx);
  requireRole(ctx, 'ADMIN', 'SELLER');
  
  const MAX_IMPORT = 10000;
  if (data.length > MAX_IMPORT) {
    throw new GraphQLError(`Maximum ${MAX_IMPORT} rows per import`);
  }
  
  const errors: ImportError[] = [];
  const validItems: Product[] = [];
  
  // Resolve categories ก่อน (batch lookup)
  const categoryNames = [...new Set(data.map((r: ProductImportRow) => r.category))];
  const categories = await ctx.prisma.category.findMany({
    where: { name: { in: categoryNames } },
    select: { id: true, name: true },
  });
  const categoryMap = new Map(categories.map(c => [c.name, c.id]));
  
  // Validate rows
  for (let row = 0; row < data.length; row++) {
    const item = data[row];
    
    if (!item.name?.trim()) {
      errors.push({ row: row + 1, column: 'name', message: 'Name is required' });
      continue;
    }
    
    if (typeof item.price !== 'number' || item.price < 0) {
      errors.push({ row: row + 1, column: 'price', message: 'Price must be a non-negative number' });
      continue;
    }
    
    const categoryId = categoryMap.get(item.category);
    if (!categoryId) {
      errors.push({ row: row + 1, column: 'category', message: `Category "${item.category}" not found` });
      continue;
    }
    
    validItems.push({
      externalId: item.externalId,
      name: item.name,
      price: item.price,
      stock: item.stock ?? 0,
      categoryId,
      sellerId: ctx.user!.sub,
    } as unknown as Product);
  }
  
  // Upsert ทั้งหมด
  let imported = 0;
  let skipped = 0;
  
  if (validItems.length > 0) {
    const result = await ctx.prisma.product.createMany({
      data: validItems as Product[],
      skipDuplicates: true, // skip ถ้า externalId ซ้ำ
    });
    imported = result.count;
    skipped = validItems.length - result.count;
  }
  
  // Preview: แสดง 5 items แรกที่ imported
  const preview = await ctx.prisma.product.findMany({
    where: { sellerId: ctx.user!.sub },
    orderBy: { createdAt: 'desc' },
    take: 5,
  });
  
  return {
    imported,
    skipped,
    errors,
    preview,
  };
},
```

---

## 📌 Step 995: Idempotent Mutations

```typescript
// Idempotent mutation: ส่ง request ซ้ำได้ โดยไม่สร้าง side effects เพิ่ม
// ใช้ idempotency key (UUID จาก client)

// Schema
// type Mutation {
//   checkout(cartId: ID!, idempotencyKey: String!): CheckoutResult!
// }

const checkoutResolver = async (_, { cartId, idempotencyKey }, ctx) => {
  requireAuth(ctx);
  
  // ตรวจสอบว่าเคย process idempotency key นี้แล้วหรือยัง
  const cachedResult = await ctx.redis.get(`idempotency:${idempotencyKey}`);
  if (cachedResult) {
    return JSON.parse(cachedResult);
  }
  
  // Lock เพื่อป้องกัน concurrent requests
  const lockKey = `lock:checkout:${idempotencyKey}`;
  const lockAcquired = await ctx.redis.set(lockKey, '1', 'NX', 'EX', 30);
  if (!lockAcquired) {
    throw new GraphQLError('Checkout in progress, please wait', {
      extensions: { code: 'CONCURRENT_REQUEST' }
    });
  }
  
  try {
    // Process checkout
    const result = await processCheckout(cartId, ctx);
    
    // Cache result สำหรับ idempotency (24 ชั่วโมง)
    await ctx.redis.setex(
      `idempotency:${idempotencyKey}`,
      86400,
      JSON.stringify(result)
    );
    
    return result;
  } finally {
    await ctx.redis.del(lockKey);
  }
};
```

---

## 📌 Step 996: สรุป Part 052

### เนื้อหาที่เรียนรู้

✅ Bulk create/update/delete mutations  
✅ Partial success pattern (BulkError)  
✅ Upsert operations  
✅ Import functionality  
✅ Idempotent mutations  
✅ Transaction handling  

### Bulk Operation Best Practices

```
1. กำหนด limit สำหรับทุก bulk operation (500/200/100 ขึ้นกับ op)
2. ตรวจสอบ ownership ก่อน batch update/delete
3. Partial success > all-or-nothing สำหรับ UX ที่ดีกว่า
4. Idempotency key สำหรับ critical mutations
5. ใช้ createMany + skipDuplicates สำหรับ import
6. Log bulk operations สำหรับ audit
7. Background job สำหรับ > 1000 items
```

### ในส่วนถัดไป

➡️ **[Part 053](./part-053.md)** — Optimistic Updates & Apollo Cache

---

*Part 052 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
