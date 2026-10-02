# Part 015 — DataLoader & N+1 Problem 🚀

> **ระดับ:** Intermediate | **เวลาเรียน:** 120 นาที | **ขั้นตอนที่:** 461–510

---

## 🎯 สิ่งที่จะได้เรียนรู้

- N+1 Problem คืออะไร
- DataLoader pattern
- Batching และ Caching
- ติดตั้งและใช้ dataloader
- DataLoaders หลายชนิด
- Per-request context DataLoader
- Test DataLoader
- Monitoring และ Debug

---

## 📌 Step 461: N+1 Problem

```graphql
# Query นี้ดูเหมือนธรรมดา
query {
  posts(limit: 10) {
    title
    author {   # ← นี่คือปัญหา!
      name
    }
  }
}
```

### ปัญหาที่เกิด

```
SQL queries ที่เกิดขึ้น:
1. SELECT * FROM posts LIMIT 10;        ← 1 query
2. SELECT * FROM users WHERE id = 1;    ← post 1
3. SELECT * FROM users WHERE id = 2;    ← post 2
4. SELECT * FROM users WHERE id = 1;    ← post 3 (author เดิม!)
5. SELECT * FROM users WHERE id = 3;    ← post 4
6. SELECT * FROM users WHERE id = 2;    ← post 5 (author ซ้ำ!)
...
11. SELECT * FROM users WHERE id = 5;   ← post 10

รวม: 1 + 10 = 11 queries!
(สำหรับ 10 posts → 10 author queries)
```

---

## 📌 Step 462: DataLoader คืออะไร

```
DataLoader แก้ N+1 ด้วย:
1. Batching: รวม queries ที่เกิดในรอบเดียวกัน
2. Caching: cache ผลลัพธ์ภายใน request เดียว

แทนที่จะ:
→ query user 1
→ query user 2
→ query user 1 (again!)

DataLoader ทำ:
→ collect user ids: [1, 2, 1]  (รอรอบเดียว)
→ batch: SELECT * FROM users WHERE id IN (1, 2)
→ return cached result สำหรับ id=1 ครั้งที่ 2
```

---

## 📌 Step 463: ติดตั้ง DataLoader

```bash
npm install dataloader
```

### DataLoader พื้นฐาน

```javascript
import DataLoader from 'dataloader';

// Batch function — รับ array ของ keys
// ต้อง return array ของ values ในลำดับเดียวกับ keys
async function batchUsers(userIds) {
  console.log('Batching users:', userIds); // → [1, 2, 3]
  
  const users = await User.findAll({
    where: { id: userIds },
  });
  
  // Map: id → user
  const userMap = new Map(users.map(u => [u.id, u]));
  
  // Return values ในลำดับเดียวกับ userIds
  return userIds.map(id => userMap.get(id) || null);
}

const userLoader = new DataLoader(batchUsers);
```

### ทำไมต้อง return ตามลำดับ

```
keys:   [1, 2, 3]
values: [user1, user2, user3]  ← ต้อง match ลำดับ!

ถ้า DB return [user3, user1, user2] ต้อง re-order ให้ถูก:
return keys.map(id => users.find(u => u.id === id))
```

---

## 📌 Step 464: ใช้ DataLoader ใน Context

```javascript
// src/context.js
import DataLoader from 'dataloader';

async function batchUsers(ids) {
  const users = await User.findAll({ where: { id: ids } });
  const userMap = new Map(users.map(u => [String(u.id), u]));
  return ids.map(id => userMap.get(String(id)) || null);
}

async function batchPostsByAuthor(authorIds) {
  const posts = await Post.findAll({
    where: { authorId: authorIds },
  });
  
  // Group by authorId
  const postsByAuthor = new Map();
  for (const post of posts) {
    const list = postsByAuthor.get(String(post.authorId)) || [];
    list.push(post);
    postsByAuthor.set(String(post.authorId), list);
  }
  
  return authorIds.map(id => postsByAuthor.get(String(id)) || []);
}

async function batchCategories(ids) {
  const categories = await Category.findAll({ where: { id: ids } });
  const map = new Map(categories.map(c => [String(c.id), c]));
  return ids.map(id => map.get(String(id)) || null);
}

// สร้าง DataLoaders ใหม่ทุก request (important!)
export function createDataLoaders() {
  return {
    userLoader: new DataLoader(batchUsers),
    postsByAuthorLoader: new DataLoader(batchPostsByAuthor),
    categoryLoader: new DataLoader(batchCategories),
  };
}

export async function createContext({ req }) {
  // สร้าง DataLoaders ใหม่ทุก request
  // (อย่าใช้ singleton — cache ต้องแยกกันระหว่าง users!)
  const loaders = createDataLoaders();
  
  return {
    ...authContext,
    loaders,
  };
}
```

---

## 📌 Step 465: ใช้ DataLoader ใน Resolvers

```javascript
const resolvers = {
  Query: {
    posts: async (_, { limit = 10 }) => {
      return Post.findAll({ limit });
    },
  },
  
  Post: {
    // ✅ ใช้ DataLoader แทน direct query
    author: async (post, _, { loaders }) => {
      return loaders.userLoader.load(post.authorId);
    },
    
    category: async (post, _, { loaders }) => {
      return loaders.categoryLoader.load(post.categoryId);
    },
  },
  
  User: {
    // ✅ Batch posts by author
    posts: async (user, { limit = 10 }, { loaders }) => {
      const posts = await loaders.postsByAuthorLoader.load(user.id);
      return posts.slice(0, limit);
    },
  },
};
```

### ผล: จาก 11 queries เหลือ 2 queries!

```
ก่อน DataLoader:
  1 query สำหรับ posts
  10 queries สำหรับ authors (N+1)
  = 11 queries

หลัง DataLoader:
  1 query สำหรับ posts
  1 batch query สำหรับ authors: IN (1,2,3,...)
  = 2 queries
```

---

## 📌 Step 466: DataLoader Options

```javascript
const userLoader = new DataLoader(batchUsers, {
  // Cache: default true
  // ป้องกัน load ซ้ำ ใน request เดียวกัน
  cache: true,
  
  // CacheKeyFn: default identity (===)
  // ใช้เมื่อ key ไม่ใช่ primitive
  cacheKeyFn: (key) => String(key),
  
  // Batch: default true
  batch: true,
  
  // MaxBatchSize: กี่ ids ต่อ batch
  maxBatchSize: 100,
  
  // BatchScheduleFn: ควบคุม timing ของ batch
  // default: process.nextTick
});
```

---

## 📌 Step 467: DataLoader สำหรับ Foreign Keys จาก Many Side

```javascript
// ต้องการ: comments ของแต่ละ post
async function batchCommentsByPost(postIds) {
  const comments = await Comment.findAll({
    where: { postId: postIds },
    order: [['createdAt', 'ASC']],
  });
  
  // Group by postId
  const commentsByPost = postIds.reduce((acc, id) => {
    acc.set(String(id), []);
    return acc;
  }, new Map());
  
  for (const comment of comments) {
    const list = commentsByPost.get(String(comment.postId));
    if (list) list.push(comment);
  }
  
  return postIds.map(id => commentsByPost.get(String(id)) || []);
}

const commentsByPostLoader = new DataLoader(batchCommentsByPost);

const resolvers = {
  Post: {
    comments: async (post, { limit = 10 }, { loaders }) => {
      const comments = await loaders.commentsByPostLoader.load(post.id);
      return comments.slice(0, limit);
    },
  },
};
```

---

## 📌 Step 468: DataLoader Caching

```javascript
// Cache ทำงานอย่างไร
const userLoader = new DataLoader(batchUsers);

// Request 1
const user1 = await userLoader.load('1');  // → DB query
const user2 = await userLoader.load('2');  // → DB query (batched with 1)

// Request 2 (ใน request เดียวกัน)
const user1Again = await userLoader.load('1');  // → from cache! ไม่ query DB

// Clear cache manually (ถ้าข้อมูลเปลี่ยน)
userLoader.clear('1');
userLoader.clearAll();

// Prime cache (pre-populate)
userLoader.prime('3', userData);  // เพิ่มข้อมูลเข้า cache โดยไม่ query
```

### Cache ต้อง Per-Request

```javascript
// ❌ อันตราย — shared loader ระหว่าง requests
const sharedUserLoader = new DataLoader(batchUsers);

// ✅ ถูกต้อง — สร้างใหม่ทุก request
export function createContext({ req }) {
  return {
    loaders: {
      userLoader: new DataLoader(batchUsers),  // ← ใหม่ทุกครั้ง
    },
  };
}
```

---

## 📌 Step 469: Test DataLoader

```javascript
// test/dataloaders/userLoader.test.js
describe('UserLoader', () => {
  let userLoader;
  
  beforeEach(() => {
    userLoader = new DataLoader(batchUsers);
    jest.spyOn(User, 'findAll');
  });
  
  it('should batch multiple loads', async () => {
    const [user1, user2, user3] = await Promise.all([
      userLoader.load('1'),
      userLoader.load('2'),
      userLoader.load('3'),
    ]);
    
    // ต้อง call findAll แค่ครั้งเดียว
    expect(User.findAll).toHaveBeenCalledTimes(1);
    expect(User.findAll).toHaveBeenCalledWith({
      where: { id: ['1', '2', '3'] },
    });
    
    expect(user1.id).toBe('1');
    expect(user2.id).toBe('2');
    expect(user3.id).toBe('3');
  });
  
  it('should cache repeated loads', async () => {
    await userLoader.load('1');
    await userLoader.load('1');
    
    expect(User.findAll).toHaveBeenCalledTimes(1);
  });
  
  it('should return null for missing users', async () => {
    const result = await userLoader.load('999');
    expect(result).toBeNull();
  });
});
```

---

## 📌 Step 470: Monitoring N+1 ด้วย Apollo Plugin

```javascript
// src/plugins/query-monitor.plugin.js
export const queryMonitorPlugin = {
  requestDidStart() {
    return {
      willSendResponse({ request, response, contextValue }) {
        const loaders = contextValue.loaders;
        
        if (!loaders) return;
        
        // Log batch stats (ถ้า extend DataLoader เพื่อ track)
        console.log('DataLoader stats:', {
          operation: request.operationName,
          batchCount: loaders.userLoader.stats?.batchCount,
          cacheHits: loaders.userLoader.stats?.cacheHits,
        });
      },
    };
  },
};
```

---

## 📌 Step 471: สรุป Part 015

### เนื้อหาที่เรียนรู้

✅ N+1 problem คืออะไร และทำไมถึงช้า  
✅ DataLoader pattern — batching + caching  
✅ implement DataLoader สำหรับ user, category, comments  
✅ Per-request context DataLoader  
✅ DataLoader options (cache, maxBatchSize)  
✅ DataLoader สำหรับ one-to-many relationships  
✅ Testing DataLoader  

### Homework

1. หา N+1 ใน Blog API schema แล้วแก้ด้วย DataLoader
2. วัด performance ก่อนและหลัง DataLoader
3. สร้าง DataLoader สำหรับ tags ของ post

### ในส่วนถัดไป

➡️ **[Part 016](./part-016.md)** — Testing GraphQL APIs

---

*Part 015 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
