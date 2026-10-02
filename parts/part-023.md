# Part 023 — PostgreSQL + Prisma Integration 🐘

> **ระดับ:** Intermediate-Advanced | **เวลาเรียน:** 150 นาที | **ขั้นตอนที่:** 781–830

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Prisma setup
- Schema definition
- Migrations
- CRUD operations
- Relations (one-to-many, many-to-many)
- Raw queries
- Transactions
- Connection pooling (PgBouncer)
- Full-text search

---

## 📌 Step 781: ติดตั้ง Prisma

```bash
npm install prisma --save-dev
npm install @prisma/client
npx prisma init
```

### .env

```env
DATABASE_URL="postgresql://user:password@localhost:5432/myapp?schema=public"
```

---

## 📌 Step 782: Prisma Schema

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String    @id @default(cuid())
  email     String    @unique
  name      String
  password  String
  role      UserRole  @default(USER)
  avatar    String?
  bio       String?
  isActive  Boolean   @default(true)
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  
  // Relations
  posts     Post[]
  comments  Comment[]
  orders    Order[]
  
  @@index([email])
  @@index([role])
  @@map("users")
}

model Post {
  id          String    @id @default(cuid())
  title       String
  slug        String    @unique
  content     String
  excerpt     String?
  status      PostStatus @default(DRAFT)
  publishedAt DateTime?
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  
  authorId    String
  author      User      @relation(fields: [authorId], references: [id], onDelete: Cascade)
  
  comments    Comment[]
  tags        PostTag[]
  
  // Full-text search
  searchVector Unsupported("tsvector")?
  
  @@index([authorId])
  @@index([status, publishedAt])
  @@index([slug])
  @@map("posts")
}

model Comment {
  id        String   @id @default(cuid())
  content   String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  postId    String
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  
  authorId  String
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  
  parentId  String?
  parent    Comment?  @relation("CommentReplies", fields: [parentId], references: [id])
  replies   Comment[] @relation("CommentReplies")
  
  @@index([postId])
  @@index([authorId])
  @@map("comments")
}

model Tag {
  id    String    @id @default(cuid())
  name  String    @unique
  slug  String    @unique
  posts PostTag[]
  
  @@map("tags")
}

model PostTag {
  postId String
  post   Post   @relation(fields: [postId], references: [id], onDelete: Cascade)
  
  tagId String
  tag   Tag    @relation(fields: [tagId], references: [id], onDelete: Cascade)
  
  @@id([postId, tagId])
  @@map("post_tags")
}

enum UserRole {
  ADMIN
  USER
}

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}
```

---

## 📌 Step 783: Migration

```bash
# Create migration
npx prisma migrate dev --name init

# Apply migration ใน production
npx prisma migrate deploy

# Reset database (dev only!)
npx prisma migrate reset

# View studio
npx prisma studio
```

---

## 📌 Step 784: Prisma Client ใน GraphQL Context

```javascript
// src/context.js
import { PrismaClient } from '@prisma/client';

// Singleton
const prisma = new PrismaClient({
  log: process.env.NODE_ENV === 'development'
    ? ['query', 'error', 'warn']
    : ['error'],
});

// Handle graceful shutdown
process.on('SIGINT', async () => {
  await prisma.$disconnect();
  process.exit(0);
});

export async function createContext({ req }) {
  return {
    prisma,
    user: await verifyToken(req),
  };
}
```

---

## 📌 Step 785: CRUD Resolvers

```javascript
// src/resolvers/post.resolver.js
export const postResolvers = {
  Query: {
    post: async (_, { id }, { prisma }) => {
      return prisma.post.findUnique({
        where: { id },
        include: {
          author: true,
          comments: {
            take: 10,
            orderBy: { createdAt: 'asc' },
            include: { author: true },
          },
          tags: { include: { tag: true } },
        },
      });
    },
    
    posts: async (_, { limit = 10, offset = 0, status }, { prisma }) => {
      const where = status ? { status } : { status: 'PUBLISHED' };
      
      const [posts, total] = await prisma.$transaction([
        prisma.post.findMany({
          where,
          take: limit,
          skip: offset,
          orderBy: { publishedAt: 'desc' },
          include: {
            author: { select: { id: true, name: true, avatar: true } },
            tags: { include: { tag: true } },
            _count: { select: { comments: true } },
          },
        }),
        prisma.post.count({ where }),
      ]);
      
      return { posts, total, hasMore: offset + limit < total };
    },
    
    // Full-text search
    searchPosts: async (_, { term, limit = 10 }, { prisma }) => {
      return prisma.$queryRaw`
        SELECT p.*, ts_rank(p.search_vector, query) AS rank
        FROM posts p, plainto_tsquery('thai', ${term}) query
        WHERE p.search_vector @@ query
        ORDER BY rank DESC
        LIMIT ${limit}
      `;
    },
  },
  
  Mutation: {
    createPost: async (_, { input }, { prisma, user }) => {
      if (!user) throw new GraphQLError('Unauthorized');
      
      const slug = generateSlug(input.title);
      
      return prisma.post.create({
        data: {
          ...input,
          slug,
          authorId: user.sub,
          status: 'DRAFT',
          // สร้าง tags ถ้ามี
          tags: input.tagIds ? {
            create: input.tagIds.map(tagId => ({ tagId })),
          } : undefined,
        },
        include: {
          author: true,
          tags: { include: { tag: true } },
        },
      });
    },
    
    updatePost: async (_, { id, input }, { prisma, user }) => {
      if (!user) throw new GraphQLError('Unauthorized');
      
      // ตรวจสอบเป็น owner
      const post = await prisma.post.findUnique({ where: { id } });
      if (!post) throw new GraphQLError('Post not found');
      if (post.authorId !== user.sub && user.role !== 'ADMIN') {
        throw new GraphQLError('Forbidden');
      }
      
      // Update tags ถ้ามี
      const tagsUpdate = input.tagIds ? {
        tags: {
          deleteMany: {},
          create: input.tagIds.map(tagId => ({ tagId })),
        },
      } : {};
      
      return prisma.post.update({
        where: { id },
        data: {
          ...omit(input, ['tagIds']),
          ...tagsUpdate,
        },
        include: {
          author: true,
          tags: { include: { tag: true } },
        },
      });
    },
    
    publishPost: async (_, { id }, { prisma, user }) => {
      const post = await prisma.post.findUnique({ where: { id } });
      if (!post) throw new GraphQLError('Post not found');
      if (post.authorId !== user.sub && user.role !== 'ADMIN') {
        throw new GraphQLError('Forbidden');
      }
      
      return prisma.post.update({
        where: { id },
        data: { status: 'PUBLISHED', publishedAt: new Date() },
      });
    },
    
    deletePost: async (_, { id }, { prisma, user }) => {
      const post = await prisma.post.findUnique({ where: { id } });
      if (!post) throw new GraphQLError('Post not found');
      if (post.authorId !== user.sub && user.role !== 'ADMIN') {
        throw new GraphQLError('Forbidden');
      }
      
      await prisma.post.delete({ where: { id } });
      return true;
    },
  },
  
  Post: {
    // tags field resolver — map PostTag → Tag
    tags: (post) => {
      if (post.tags) {
        return post.tags.map(pt => pt.tag);
      }
      return [];
    },
    
    commentCount: (post) => post._count?.comments ?? 0,
  },
};
```

---

## 📌 Step 786: Transactions

```javascript
const resolvers = {
  Mutation: {
    createOrderWithItems: async (_, { input }, { prisma, user }) => {
      // ทุก operations ใน $transaction เป็น atomic
      return prisma.$transaction(async (tx) => {
        // 1. สร้าง Order
        const order = await tx.order.create({
          data: {
            customerId: user.sub,
            status: 'PENDING',
            total: 0, // จะอัพเดททีหลัง
          },
        });
        
        let total = 0;
        const orderItems = [];
        
        // 2. สร้าง OrderItems + ลด stock
        for (const item of input.items) {
          const product = await tx.product.findUnique({
            where: { id: item.productId },
          });
          
          if (!product) throw new Error(`Product ${item.productId} not found`);
          if (product.stock < item.quantity) {
            throw new Error(`Insufficient stock for ${product.name}`);
          }
          
          // ลด stock
          await tx.product.update({
            where: { id: item.productId },
            data: { stock: { decrement: item.quantity } },
          });
          
          const itemTotal = product.price * item.quantity;
          total += itemTotal;
          
          orderItems.push(
            tx.orderItem.create({
              data: {
                orderId: order.id,
                productId: item.productId,
                quantity: item.quantity,
                price: product.price,
                total: itemTotal,
              },
            })
          );
        }
        
        // 3. สร้าง order items ทั้งหมด
        await Promise.all(orderItems);
        
        // 4. อัพเดท total
        return tx.order.update({
          where: { id: order.id },
          data: { total },
          include: { items: { include: { product: true } } },
        });
      });
    },
  },
};
```

---

## 📌 Step 787: สรุป Part 023

### เนื้อหาที่เรียนรู้

✅ Prisma setup + schema  
✅ Migrations  
✅ Prisma Client ใน context  
✅ CRUD operations  
✅ Relations (one-to-many, many-to-many)  
✅ Full-text search  
✅ Transactions  

### Homework

1. สร้าง blog API ด้วย Prisma + PostgreSQL
2. implement full-text search
3. ทำ transaction สำหรับ order creation

### ในส่วนถัดไป

➡️ **[Part 024](./part-024.md)** — MongoDB + Mongoose Integration

---

*Part 023 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
