# Part 002 — GraphQL vs REST API

> **ระดับ:** Beginner | **เวลาเรียน:** 60 นาที | **ขั้นตอนที่:** 16–35

---

## 🎯 สิ่งที่จะได้เรียนรู้

- เปรียบเทียบ GraphQL และ REST อย่างละเอียด
- เมื่อไหร่ควรใช้ GraphQL เมื่อไหร่ควรใช้ REST
- การ migrate จาก REST ไป GraphQL
- ตัวอย่างจริง: สร้าง API เดียวกันด้วย REST และ GraphQL

---

## 📌 Step 16: REST API คืออะไร (ทบทวน)

REST (Representational State Transfer) เป็น architectural style สำหรับสร้าง API:

```
HTTP Methods:
GET    /users          → ดึง users ทั้งหมด
GET    /users/1        → ดึง user คนเดียว
POST   /users          → สร้าง user ใหม่
PUT    /users/1        → แก้ไข user ทั้งหมด
PATCH  /users/1        → แก้ไข user บางส่วน
DELETE /users/1        → ลบ user
```

### ตัวอย่าง REST API

```javascript
// Express.js REST API
const express = require('express');
const app = express();

// Mock data
const users = [
  { id: 1, name: 'สมชาย', email: 'somchai@example.com', age: 28 },
  { id: 2, name: 'สมหญิง', email: 'somying@example.com', age: 25 },
];

const posts = [
  { id: 1, userId: 1, title: 'บทความแรก', content: 'เนื้อหา...' },
  { id: 2, userId: 1, title: 'บทความที่สอง', content: 'เนื้อหา...' },
  { id: 3, userId: 2, title: 'บทความของสมหญิง', content: 'เนื้อหา...' },
];

// GET /users
app.get('/users', (req, res) => {
  res.json(users);
});

// GET /users/:id
app.get('/users/:id', (req, res) => {
  const user = users.find(u => u.id === parseInt(req.params.id));
  if (!user) return res.status(404).json({ error: 'ไม่พบ user' });
  res.json(user);
});

// GET /users/:id/posts
app.get('/users/:id/posts', (req, res) => {
  const userPosts = posts.filter(p => p.userId === parseInt(req.params.id));
  res.json(userPosts);
});

app.listen(3000, () => console.log('REST API running on port 3000'));
```

---

## 📌 Step 17: ปัญหาหลักของ REST — Over-fetching

### สถานการณ์: แอพมือถือต้องการแสดงรายชื่อ users

**REST Response** เมื่อเรียก `GET /users`:
```json
[
  {
    "id": 1,
    "name": "สมชาย ใจดี",
    "email": "somchai@example.com",
    "phone": "0812345678",
    "address": {
      "street": "123 ถนนสุขุมวิท",
      "city": "กรุงเทพฯ",
      "zipCode": "10110"
    },
    "profilePicture": "https://example.com/pic1.jpg",
    "bio": "นักพัฒนาซอฟต์แวร์...",
    "preferences": {
      "language": "th",
      "theme": "dark",
      "notifications": true
    },
    "stats": {
      "postsCount": 42,
      "followersCount": 1234,
      "followingCount": 567
    },
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-06-15T12:00:00Z"
  }
]
```

**แต่แอพต้องการแค่:**
```json
[
  {
    "id": 1,
    "name": "สมชาย ใจดี",
    "profilePicture": "https://example.com/pic1.jpg"
  }
]
```

**ข้อมูลที่โหลดมาโดยไม่จำเป็น = bandwidth เสีย + battery drain บนมือถือ**

**GraphQL แก้ปัญหานี้:**
```graphql
query {
  users {
    id
    name
    profilePicture
  }
}
```

---

## 📌 Step 18: ปัญหาหลักของ REST — Under-fetching (N+1)

### สถานการณ์: แสดงโปรไฟล์ user พร้อมโพสต์และ comments

**ต้องเรียก API หลายครั้ง:**
```
Request 1: GET /users/1
→ ได้ข้อมูล user

Request 2: GET /users/1/posts
→ ได้ posts: [1, 2, 3]

Request 3: GET /posts/1/comments
Request 4: GET /posts/2/comments  
Request 5: GET /posts/3/comments
→ ได้ comments แต่ละโพสต์

Request 6: GET /users/5   (comment author ของโพสต์ 1)
Request 7: GET /users/8   (comment author ของโพสต์ 2)
...

รวม: 7+ HTTP requests สำหรับหน้าเดียว!
```

**ผลเสีย:**
- Latency เพิ่มขึ้น (waterfall requests)
- Network overhead
- Server load สูง
- UX แย่ (loading spinner นาน)

**GraphQL แก้ด้วย 1 request:**
```graphql
query UserProfile {
  user(id: 1) {
    name
    email
    profilePicture
    posts {
      id
      title
      createdAt
      comments {
        text
        author {
          name
          profilePicture
        }
      }
    }
  }
}
```

---

## 📌 Step 19: ปัญหาของ REST — API Versioning

```
# Versioning ทำให้ codebase ซับซ้อน
/api/v1/users        ← version เก่า (deprecated)
/api/v2/users        ← version ปัจจุบัน
/api/v3/users        ← version ใหม่ (beta)

# ปัญหา:
- ต้อง maintain หลาย version พร้อมกัน
- Documentation ซับซ้อน
- Clients ต้อง migrate
- ทดสอบยาก
```

**GraphQL ใช้ Schema Evolution แทน:**
```graphql
type User {
  id: ID!
  name: String!
  email: String!
  
  # เพิ่ม field ใหม่ได้เลย — backward compatible
  phoneNumber: String      # เพิ่มใน v2
  address: Address         # เพิ่มใน v3
  
  # Deprecated field แต่ยังใช้ได้
  username: String @deprecated(reason: "ใช้ name แทน")
}
```

Client เก่าไม่ query field ใหม่ → ยังทำงานได้ปกติ  
Client ใหม่ query field ใหม่ได้ → ได้ features ใหม่

---

## 📌 Step 20: ข้อดีของ REST ที่ยังดีกว่า

REST ก็มีข้อดีที่ GraphQL ด้อยกว่า:

### 1. HTTP Caching
```
REST:
GET /users/1
→ Browser/CDN cache ได้ทันที (URL เป็น cache key)
→ ETag, Last-Modified headers ทำงานได้ดี

GraphQL:
POST /graphql
→ POST requests ไม่ cache โดย default
→ ต้องใช้ persisted queries หรือ GET method
→ Cache key ซับซ้อนกว่า
```

### 2. File Upload
```
REST:
POST /upload
Content-Type: multipart/form-data
→ ง่ายมาก

GraphQL:
→ ต้องใช้ multipart request spec หรือ library พิเศษ
→ ซับซ้อนกว่า
```

### 3. HTTP Status Codes
```
REST:
200 OK, 201 Created, 400 Bad Request, 401 Unauthorized, 404 Not Found
→ ชัดเจน, standardized

GraphQL:
200 OK เสมอ (แม้จะ error)
→ Error อยู่ใน response body
→ Monitoring/alerting ซับซ้อนกว่า
```

### 4. Learning Curve
```
REST:
- ใช้ HTTP methods ที่คุ้นเคย
- เรียนรู้ง่าย
- Documentation มาก

GraphQL:
- ต้องเรียน Query Language ใหม่
- ต้องเข้าใจ Schema, Resolvers
- Setup ซับซ้อนกว่า
```

---

## 📌 Step 21: เมื่อไหร่ควรใช้อะไร

```
ใช้ GraphQL เมื่อ:
✅ มี multiple clients (web, mobile, 3rd party)
✅ Clients ต้องการ data ต่างกัน
✅ Data มี relationships ซับซ้อน
✅ ต้องการ real-time (subscriptions)
✅ ทีม frontend ต้องการ independence
✅ API public/partner API ที่หลาย developers ใช้
✅ Mobile apps (bandwidth สำคัญ)
✅ Rapid prototyping

ใช้ REST เมื่อ:
✅ Simple CRUD operations
✅ API สำหรับ single client อย่างง่าย
✅ Caching สำคัญมาก (CDN, browser cache)
✅ ทีมคุ้นเคยกับ REST
✅ File upload เป็นหลัก
✅ Public API ที่ต้องการ simplicity
✅ Streaming large files
✅ ระบบที่ต้องการ HTTP status codes
```

---

## 📌 Step 22: ตัวอย่างจริง — สร้าง Blog API ด้วยทั้งสองแบบ

### Schema ของข้อมูล

```
Users:
- id, name, email, createdAt

Posts:
- id, title, content, authorId, createdAt, tags

Comments:
- id, text, authorId, postId, createdAt
```

### REST API Design

```
# Users
GET    /users                  → list users
GET    /users/:id              → get user
POST   /users                  → create user
PUT    /users/:id              → update user
DELETE /users/:id              → delete user

# Posts
GET    /posts                  → list posts
GET    /posts?tag=graphql      → filter by tag
GET    /posts/:id              → get post
POST   /posts                  → create post
PUT    /posts/:id              → update post
DELETE /posts/:id              → delete post

# Comments
GET    /posts/:id/comments     → list comments
POST   /posts/:id/comments     → create comment
DELETE /comments/:id           → delete comment

# Combined (custom endpoints)
GET    /posts/:id/full         → post + author + comments + comment authors
```

**ปัญหา:** ต้องสร้าง endpoint พิเศษ `/posts/:id/full` หรือ client ต้องเรียกหลายครั้ง

### GraphQL Schema Design

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  createdAt: String!
  posts: [Post!]!
  comments: [Comment!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments: [Comment!]!
  tags: [String!]!
  createdAt: String!
}

type Comment {
  id: ID!
  text: String!
  author: User!
  post: Post!
  createdAt: String!
}

type Query {
  users: [User!]!
  user(id: ID!): User
  posts(tag: String): [Post!]!
  post(id: ID!): Post
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
  
  createPost(input: CreatePostInput!): Post!
  updatePost(id: ID!, input: UpdatePostInput!): Post!
  deletePost(id: ID!): Boolean!
  
  createComment(postId: ID!, text: String!): Comment!
  deleteComment(id: ID!): Boolean!
}

input CreateUserInput {
  name: String!
  email: String!
}

input UpdateUserInput {
  name: String
  email: String
}

input CreatePostInput {
  title: String!
  content: String!
  tags: [String!]
}

input UpdatePostInput {
  title: String
  content: String
  tags: [String!]
}
```

---

## 📌 Step 23: สร้าง GraphQL Blog Server

### `src/data.js` — Mock Database

```javascript
export const users = [
  {
    id: "1",
    name: "สมชาย ใจดี",
    email: "somchai@example.com",
    createdAt: "2024-01-01T00:00:00Z",
  },
  {
    id: "2",
    name: "สมหญิง สวยงาม",
    email: "somying@example.com",
    createdAt: "2024-02-01T00:00:00Z",
  },
];

export const posts = [
  {
    id: "1",
    title: "เริ่มต้นกับ GraphQL",
    content: "GraphQL เป็นภาษา query สำหรับ API...",
    authorId: "1",
    tags: ["graphql", "api"],
    createdAt: "2024-03-01T00:00:00Z",
  },
  {
    id: "2",
    title: "Node.js Best Practices",
    content: "สิ่งที่ควรรู้เกี่ยวกับ Node.js...",
    authorId: "1",
    tags: ["nodejs", "javascript"],
    createdAt: "2024-04-01T00:00:00Z",
  },
  {
    id: "3",
    title: "React Hooks ทำความเข้าใจ",
    content: "Hooks เปลี่ยนวิธีเขียน React...",
    authorId: "2",
    tags: ["react", "javascript"],
    createdAt: "2024-05-01T00:00:00Z",
  },
];

export const comments = [
  {
    id: "1",
    text: "บทความดีมากเลย!",
    authorId: "2",
    postId: "1",
    createdAt: "2024-03-02T00:00:00Z",
  },
  {
    id: "2",
    text: "ขอบคุณสำหรับการแชร์",
    authorId: "1",
    postId: "3",
    createdAt: "2024-05-02T00:00:00Z",
  },
];
```

### `src/schema.js`

```javascript
export const typeDefs = `#graphql
  type User {
    id: ID!
    name: String!
    email: String!
    createdAt: String!
    posts: [Post!]!
    comments: [Comment!]!
  }

  type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
    comments: [Comment!]!
    tags: [String!]!
    createdAt: String!
  }

  type Comment {
    id: ID!
    text: String!
    author: User!
    post: Post!
    createdAt: String!
  }

  type Query {
    users: [User!]!
    user(id: ID!): User
    posts(tag: String): [Post!]!
    post(id: ID!): Post
  }

  type Mutation {
    createUser(name: String!, email: String!): User!
    createPost(title: String!, content: String!, tags: [String!]): Post!
    createComment(postId: ID!, text: String!): Comment!
  }
`;
```

### `src/resolvers.js`

```javascript
import { users, posts, comments } from './data.js';

// IDs สำหรับ mock data ใหม่
let nextUserId = 3;
let nextPostId = 4;
let nextCommentId = 3;

export const resolvers = {
  Query: {
    users: () => users,
    user: (_, { id }) => users.find(u => u.id === id),
    posts: (_, { tag }) => {
      if (tag) {
        return posts.filter(p => p.tags.includes(tag));
      }
      return posts;
    },
    post: (_, { id }) => posts.find(p => p.id === id),
  },

  Mutation: {
    createUser: (_, { name, email }) => {
      const user = {
        id: String(nextUserId++),
        name,
        email,
        createdAt: new Date().toISOString(),
      };
      users.push(user);
      return user;
    },
    
    createPost: (_, { title, content, tags = [] }, context) => {
      // ใน production ใช้ context.userId จาก auth
      const post = {
        id: String(nextPostId++),
        title,
        content,
        authorId: "1", // mock: ใช้ user 1 เสมอ
        tags,
        createdAt: new Date().toISOString(),
      };
      posts.push(post);
      return post;
    },
    
    createComment: (_, { postId, text }) => {
      const comment = {
        id: String(nextCommentId++),
        text,
        authorId: "1", // mock
        postId,
        createdAt: new Date().toISOString(),
      };
      comments.push(comment);
      return comment;
    },
  },

  // Resolvers สำหรับ nested fields
  User: {
    posts: (user) => posts.filter(p => p.authorId === user.id),
    comments: (user) => comments.filter(c => c.authorId === user.id),
  },

  Post: {
    author: (post) => users.find(u => u.id === post.authorId),
    comments: (post) => comments.filter(c => c.postId === post.id),
  },

  Comment: {
    author: (comment) => users.find(u => u.id === comment.authorId),
    post: (comment) => posts.find(p => p.id === comment.postId),
  },
};
```

---

## 📌 Step 24: ทดสอบ Blog API

```graphql
# Query 1: ดู posts ทั้งหมด
query AllPosts {
  posts {
    id
    title
    tags
    author {
      name
    }
  }
}
```

```graphql
# Query 2: ดู post เดียว พร้อม comments และ authors
query PostWithComments {
  post(id: "1") {
    title
    content
    author {
      name
      email
    }
    comments {
      text
      author {
        name
      }
      createdAt
    }
  }
}
```

```graphql
# Query 3: Filter posts by tag
query PostsByTag {
  posts(tag: "graphql") {
    title
    tags
  }
}
```

```graphql
# Mutation: สร้าง user ใหม่
mutation CreateUser {
  createUser(name: "วิชัย เก่งมาก", email: "wichai@example.com") {
    id
    name
    email
    createdAt
  }
}
```

```graphql
# Mutation: สร้าง comment
mutation AddComment {
  createComment(postId: "1", text: "ชอบบทความนี้มาก!") {
    id
    text
    author {
      name
    }
    post {
      title
    }
  }
}
```

---

## 📌 Step 25: เปรียบเทียบ Requests — REST vs GraphQL

### Use Case: หน้า "โปรไฟล์ผู้ใช้" (User Profile Page)

**ต้องการ:**
- ชื่อ, email ของ user
- โพสต์ล่าสุด 3 อัน (title เท่านั้น)
- จำนวน comments ทั้งหมด

**REST — 3 requests:**
```bash
# Request 1
GET /users/1
Response: { id, name, email, phone, address, bio, ... }  # over-fetch

# Request 2  
GET /users/1/posts?limit=3
Response: [{ id, title, content, authorId, tags, ... }]  # over-fetch

# Request 3
GET /users/1/comments/count
Response: { count: 5 }
```

**GraphQL — 1 request:**
```graphql
query UserProfilePage {
  user(id: "1") {
    name
    email
    posts {
      title
    }
    # ต้องเพิ่ม field commentsCount ใน schema
    # หรือใช้ length ของ comments array
  }
}
```

---

## 📌 Step 26: GraphQL ทำงานอย่างไร — Deep Dive

```
Client                      Server
  │                            │
  │   1. ส่ง Query/Mutation    │
  │ ─────────────────────────→ │
  │   POST /graphql             │
  │   { query: "...",           │
  │     variables: {...} }      │
  │                            │
  │                    2. Parse & Validate
  │                    (ตรวจสอบ syntax)
  │                            │
  │                    3. Execute
  │                    (เรียก Resolvers)
  │                            │
  │   4. ส่ง Response กลับ     │
  │ ←───────────────────────── │
  │   { data: {...},            │
  │     errors: [...] }         │
```

### Execution Process

```javascript
// เมื่อ client ส่ง:
// POST /graphql
// { "query": "query { user(id: \"1\") { name posts { title } } }" }

// 1. Parse → AST (Abstract Syntax Tree)
{
  kind: "Document",
  definitions: [{
    kind: "OperationDefinition",
    operation: "query",
    selectionSet: {
      selections: [{
        kind: "Field",
        name: { value: "user" },
        arguments: [{ name: "id", value: "1" }],
        selectionSet: {
          selections: [
            { kind: "Field", name: { value: "name" } },
            {
              kind: "Field",
              name: { value: "posts" },
              selectionSet: {
                selections: [
                  { kind: "Field", name: { value: "title" } }
                ]
              }
            }
          ]
        }
      }]
    }
  }]
}

// 2. Validate → ตรวจว่า fields มีอยู่ใน schema

// 3. Execute → เรียก resolvers
// Query.user(root, { id: "1" }, context, info)
// → ได้ user object
// User.posts(user, {}, context, info)
// → ได้ posts array
// (ไม่เรียก Post.title เพราะ title เป็น scalar)
```

---

## 📌 Step 27: Transport Layer

GraphQL ไม่ผูกติดกับ HTTP — แต่ส่วนใหญ่ใช้:

```
Standard GraphQL over HTTP:
POST /graphql
Content-Type: application/json

{
  "query": "query { users { name } }",
  "variables": { "userId": "1" },
  "operationName": "GetUsers"
}

Response:
{
  "data": { ... },
  "errors": [ ... ]  // optional
}
```

```
GraphQL over GET (สำหรับ queries เท่านั้น):
GET /graphql?query={users{name}}&operationName=GetUsers

(ใช้สำหรับ caching)
```

```
GraphQL over WebSocket (สำหรับ subscriptions):
ws://localhost:4000/graphql

Protocol: graphql-ws หรือ subscriptions-transport-ws
```

---

## 📌 Step 28: Introspection — Self-documenting API

หนึ่งในข้อดีของ GraphQL คือ schema เป็น self-documenting:

```graphql
# ดู types ทั้งหมด
query {
  __schema {
    types {
      name
      description
      kind
    }
  }
}

# ดู queries ที่มีทั้งหมด
query {
  __schema {
    queryType {
      fields {
        name
        description
        args {
          name
          type {
            name
            kind
          }
        }
      }
    }
  }
}
```

Tools ที่ใช้ Introspection:
- **Apollo Sandbox** — GraphQL IDE
- **GraphiQL** — open source playground
- **Postman** — รองรับ GraphQL
- **Insomnia** — API client
- **graphql-inspector** — schema diff tool

---

## 📌 Step 29: GraphQL Clients

```javascript
// 1. Vanilla fetch (ง่ายที่สุด)
const response = await fetch('http://localhost:4000/graphql', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    query: `
      query {
        users {
          name
          email
        }
      }
    `
  })
});
const { data, errors } = await response.json();
```

```javascript
// 2. Apollo Client (ยอดนิยมสำหรับ React)
import { ApolloClient, InMemoryCache, gql } from '@apollo/client';

const client = new ApolloClient({
  uri: 'http://localhost:4000/graphql',
  cache: new InMemoryCache(),
});

const { data } = await client.query({
  query: gql`
    query {
      users {
        name
        email
      }
    }
  `
});
```

```javascript
// 3. urql (lightweight alternative)
import { createClient, cacheExchange, fetchExchange } from 'urql';

const client = createClient({
  url: 'http://localhost:4000/graphql',
  exchanges: [cacheExchange, fetchExchange],
});
```

```javascript
// 4. React Query + custom fetcher
import { useQuery } from '@tanstack/react-query';

function useGraphQL(query, variables) {
  return useQuery({
    queryKey: [query, variables],
    queryFn: async () => {
      const res = await fetch('/graphql', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ query, variables }),
      });
      return res.json();
    },
  });
}
```

---

## 📌 Step 30: Migration Strategy — REST to GraphQL

### แนวทางที่ 1: The Strangler Fig Pattern

```
เริ่มต้น: REST API เต็ม 100%

Phase 1: เพิ่ม GraphQL layer ทับ REST
┌─────────────────────────────┐
│  GraphQL Server             │
│  (wraps REST APIs)          │
└─────────────────────────────┘
         │
┌────────┴─────────────────────┐
│  REST APIs (ยังทำงานอยู่)    │
└──────────────────────────────┘

Phase 2: ย้าย logic ทีละ service
Phase 3: เลิกใช้ REST endpoints เก่า
```

### แนวทางที่ 2: Parallel Run

```javascript
// สร้าง GraphQL server ที่เรียก REST APIs ภายใน
const resolvers = {
  Query: {
    users: async () => {
      // เรียก REST API ภายใน
      const response = await fetch('http://rest-api/users');
      return response.json();
    },
    
    user: async (_, { id }) => {
      const response = await fetch(`http://rest-api/users/${id}`);
      return response.json();
    },
  },
};
```

### แนวทางที่ 3: Schema-First Migration

```graphql
# 1. ออกแบบ GraphQL schema ก่อน
type User {
  id: ID!
  name: String!
  # ...
}

# 2. สร้าง resolvers ที่ยังใช้ REST data source
# 3. ทดสอบให้ครบ
# 4. Migrate resolvers ทีละตัวให้ใช้ DB โดยตรง
# 5. เลิกใช้ REST layer
```

---

## 📌 Step 31: GraphQL Tools Ecosystem

```
Schema Design:
├── graphql-js          — Reference implementation
├── type-graphql        — TypeScript decorators
└── nexus               — Code-first schema

Servers:
├── @apollo/server      — Most popular
├── graphql-yoga        — Flexible, modern
├── mercurius           — Fastify plugin
└── graphql-http        — Reference server

Clients:
├── @apollo/client      — React, full-featured
├── urql                — Lightweight
├── relay               — Facebook's client
└── graphql-request     — Simple fetcher

Tools:
├── graphiql            — Browser IDE
├── apollo sandbox      — Modern IDE
├── graphql-inspector   — Schema validation
├── graphql-code-generator — Type generation
└── rover               — Apollo CLI

Testing:
├── jest + supertest    — Integration tests
└── @graphql-tools/mock — Schema mocking
```

---

## 📌 Step 32: Performance Considerations

### Query Complexity

```graphql
# Query ที่อาจทำให้ server ช้า
query Dangerous {
  users {
    posts {
      comments {
        author {
          posts {
            comments {
              # ลึกมากๆ = expensive!
            }
          }
        }
      }
    }
  }
}
```

### Solutions

```javascript
// 1. Depth Limiting
import depthLimit from 'graphql-depth-limit';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [depthLimit(5)], // max depth = 5
});

// 2. Query Complexity
import { createComplexityLimitRule } from 'graphql-validation-complexity';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    createComplexityLimitRule(1000) // max complexity = 1000
  ],
});
```

---

## 📌 Step 33: Real-world Considerations

### เมื่อ GraphQL ทำให้ช้าลง

```
สถานการณ์: E-commerce catalog API

REST:
GET /products?category=electronics&sort=price&page=2
→ 1 SQL query ที่ optimize แล้ว
→ ~50ms response time

GraphQL (naive):
query {
  products(category: "electronics") {
    name
    price
    category {   ← N+1!
      name
    }
    reviews {    ← N+1!
      rating
    }
  }
}
→ 1 query + N queries สำหรับ categories + N queries สำหรับ reviews
→ 200ms+ response time
```

**แก้ด้วย DataLoader (เรียนใน Part 018)**

---

## 📌 Step 34: Checklist — GraphQL vs REST Decision

```
ถามตัวเอง:

1. มี clients หลายตัว (web, mobile, 3rd party)?
   → ใช่ → GraphQL

2. Clients ต้องการ data ต่างกัน?
   → ใช่ → GraphQL

3. Data มี relationships ซับซ้อน?
   → ใช่ → GraphQL

4. ต้องการ real-time?
   → ใช่ → GraphQL (subscriptions)

5. HTTP caching สำคัญมาก (CDN, public API)?
   → ใช่ → REST

6. Simple CRUD ไม่มี relationships?
   → ใช่ → REST

7. ทีมเล็ก, timeline สั้น?
   → REST ก่อน แล้วค่อย migrate

8. ต้องการ self-documenting API?
   → GraphQL (introspection)
```

---

## 📌 Step 35: สรุป Part 002

### เนื้อหาที่เรียนรู้

✅ Over-fetching และ Under-fetching ใน REST  
✅ GraphQL แก้ปัญหา versioning  
✅ ข้อดีของ REST ที่ยังดีกว่า (HTTP caching, simplicity)  
✅ สร้าง Blog API ด้วย GraphQL  
✅ Migration strategies  
✅ เมื่อไหร่ควรใช้อะไร  

### Homework

1. สร้าง REST Blog API ด้วย Express.js และเปรียบเทียบกับ GraphQL version
2. นับจำนวน HTTP requests ที่ต้องใช้สำหรับหน้า "user profile" ใน REST
3. เพิ่ม `tags` filter ใน GraphQL posts query

### ในส่วนถัดไป

➡️ **[Part 003](./part-003.md)** — ติดตั้งและตั้งค่า GraphQL Server แบบสมบูรณ์

---

*Part 002 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
