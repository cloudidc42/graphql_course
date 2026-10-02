# Part 001 — Introduction to GraphQL

> **ระดับ:** Beginner | **เวลาเรียน:** 45 นาที | **ขั้นตอนที่:** 1–15

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

- GraphQL คืออะไร และทำไมถึงสำคัญ
- ประวัติและที่มาของ GraphQL
- GraphQL ทำงานอย่างไร (overview)
- ข้อดีของ GraphQL เทียบกับ REST
- Use cases จริงที่บริษัทใหญ่ใช้
- ติดตั้งและทดลอง query แรกของคุณ

---

## 📌 Step 1: GraphQL คืออะไร?

**GraphQL** (Graph Query Language) คือภาษาสำหรับ query ข้อมูล และ runtime สำหรับ execute query เหล่านั้น โดย Facebook (ปัจจุบัน Meta) พัฒนาขึ้นในปี 2012 และ open source ในปี 2015

### คำจำกัดความอย่างเป็นทางการ

> "GraphQL is a query language for your API, and a server-side runtime for executing queries using a type system you define for your data."
> — graphql.org

쉽게 말하면ง่ายๆ ว่า:

```
GraphQL = วิธีการขอข้อมูลจาก Server ที่ยืดหยุ่น และมีประสิทธิภาพสูงกว่า REST
```

---

## 📌 Step 2: ปัญหาที่ GraphQL แก้ไข

### ปัญหาที่ 1: Over-fetching (ได้ข้อมูลมากเกินไป)

ใน REST API เมื่อคุณเรียก `/users/1` คุณจะได้รับข้อมูลทั้งหมดของ user แม้จะต้องการแค่ชื่อ:

```json
// REST Response — คุณต้องการแค่ชื่อ แต่ได้ทั้งหมด
{
  "id": 1,
  "name": "สมชาย",
  "email": "somchai@example.com",
  "phone": "0812345678",
  "address": "กรุงเทพฯ",
  "createdAt": "2024-01-01",
  "updatedAt": "2024-06-15",
  "role": "admin",
  "preferences": { ... },
  "metadata": { ... }
}
```

ใน GraphQL คุณขอเฉพาะที่ต้องการ:

```graphql
# GraphQL Query — ขอแค่ชื่อ
query {
  user(id: 1) {
    name
  }
}
```

```json
// GraphQL Response — ได้แค่ที่ขอ
{
  "data": {
    "user": {
      "name": "สมชาย"
    }
  }
}
```

### ปัญหาที่ 2: Under-fetching (ได้ข้อมูลน้อยเกินไป — N+1 problem)

ใน REST ถ้าต้องการโพสต์ของ user พร้อมความคิดเห็น ต้องเรียก API หลายครั้ง:

```
GET /users/1           → ได้ข้อมูล user
GET /users/1/posts     → ได้รายการโพสต์
GET /posts/1/comments  → ได้ comment ของโพสต์แรก
GET /posts/2/comments  → ได้ comment ของโพสต์ที่สอง
...
```

ใน GraphQL เรียกครั้งเดียวได้ทุกอย่าง:

```graphql
query {
  user(id: 1) {
    name
    posts {
      title
      comments {
        text
        author {
          name
        }
      }
    }
  }
}
```

### ปัญหาที่ 3: Versioning Hell

```
REST:
/api/v1/users
/api/v2/users   ← version ใหม่เมื่อ schema เปลี่ยน
/api/v3/users   ← version อีกอัน

GraphQL:
/graphql        ← endpoint เดียว ตลอดไป
```

---

## 📌 Step 3: ประวัติของ GraphQL

```
2012 → Facebook พัฒนา GraphQL สำหรับแอพมือถือ iOS ภายใน
       (แก้ปัญหา slow news feed บน 2G networks)

2015 → Facebook open-source GraphQL specification

2016 → GitHub ประกาศใช้ GraphQL สำหรับ API v4
       Shopify, Twitter ก็เริ่มใช้

2018 → GraphQL Foundation ก่อตั้งภายใต้ Linux Foundation
       (ทำให้ไม่ขึ้นกับ Facebook/Meta)

2019 → GraphQL spec มีการ update ครั้งใหญ่
       Apollo Federation เปิดตัว

2021 → GitHub รายงานว่า GraphQL API รับ requests 15 billion+ ต่อวัน

2023 → GraphQL HTTP spec เสร็จสมบูรณ์
       Incremental Delivery (defer/stream) ถูกบรรจุ

ปัจจุบัน → ใช้ที่: Meta, GitHub, Shopify, Twitter, Netflix, 
           Airbnb, Pinterest, PayPal, Lyft...
```

---

## 📌 Step 4: องค์ประกอบหลักของ GraphQL

GraphQL มี 3 ส่วนหลัก:

### 1. Schema (โครงสร้างข้อมูล)
```graphql
# บอกว่า API มีข้อมูลอะไรบ้าง
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
}
```

### 2. Query Language (ภาษาสำหรับขอข้อมูล)
```graphql
# Client ส่ง request แบบนี้
query GetUser {
  user(id: "1") {
    name
    email
    posts {
      title
    }
  }
}
```

### 3. Runtime (ส่วนประมวลผล)
```javascript
// Server ประมวลผลและส่ง response กลับ
{
  "data": {
    "user": {
      "name": "สมชาย",
      "email": "somchai@example.com",
      "posts": [
        { "title": "บทความแรก" },
        { "title": "บทความที่สอง" }
      ]
    }
  }
}
```

---

## 📌 Step 5: GraphQL Operations 3 ประเภท

```graphql
# 1. Query — ดึงข้อมูล (Read)
query {
  users {
    name
    email
  }
}

# 2. Mutation — เปลี่ยนแปลงข้อมูล (Create/Update/Delete)
mutation {
  createUser(input: { name: "สมหญิง", email: "som@example.com" }) {
    id
    name
  }
}

# 3. Subscription — รับข้อมูล Real-time
subscription {
  userCreated {
    id
    name
  }
}
```

---

## 📌 Step 6: การติดตั้ง Environment

### ความต้องการของระบบ

```bash
# ตรวจสอบ Node.js version (ต้องการ 18+)
node --version
# v20.x.x ✓

# ตรวจสอบ npm
npm --version
# 10.x.x ✓
```

### สร้าง Project แรก

```bash
# สร้างโฟลเดอร์
mkdir graphql-hello-world
cd graphql-hello-world

# เริ่มต้น project
npm init -y

# ติดตั้ง dependencies
npm install graphql

# ติดตั้ง Apollo Server (server ยอดนิยม)
npm install @apollo/server

# ติดตั้ง dev dependencies
npm install -D nodemon
```

### โครงสร้างไฟล์

```
graphql-hello-world/
├── package.json
├── src/
│   ├── index.js       ← จุดเริ่มต้น
│   ├── schema.js      ← กำหนด Types
│   └── resolvers.js   ← Logic การดึงข้อมูล
└── node_modules/
```

---

## 📌 Step 7: Hello World — GraphQL Server แรก

### ไฟล์ `src/schema.js`

```javascript
// กำหนดโครงสร้างข้อมูลของ API
export const typeDefs = `#graphql
  # Query type — บอกว่า API มี query อะไรบ้าง
  type Query {
    hello: String
    greeting(name: String): String
  }
`;
```

### ไฟล์ `src/resolvers.js`

```javascript
// Resolvers — ฟังก์ชันที่ดึงข้อมูลจริงๆ
export const resolvers = {
  Query: {
    // เมื่อ client query `hello` จะได้ string นี้
    hello: () => "สวัสดี GraphQL! 🎉",
    
    // เมื่อ client query `greeting(name: "สมชาย")` 
    greeting: (_, { name }) => `สวัสดี ${name || "ทุกคน"}!`,
  },
};
```

### ไฟล์ `src/index.js`

```javascript
import { ApolloServer } from "@apollo/server";
import { startStandaloneServer } from "@apollo/server/standalone";
import { typeDefs } from "./schema.js";
import { resolvers } from "./resolvers.js";

// สร้าง Apollo Server
const server = new ApolloServer({
  typeDefs,
  resolvers,
});

// เริ่มต้น server
const { url } = await startStandaloneServer(server, {
  listen: { port: 4000 },
});

console.log(`🚀 GraphQL Server พร้อมใช้งานที่: ${url}`);
console.log(`📊 เปิด GraphQL Explorer ที่: ${url}`);
```

### ไฟล์ `package.json`

```json
{
  "name": "graphql-hello-world",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js"
  },
  "dependencies": {
    "@apollo/server": "^4.10.0",
    "graphql": "^16.8.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.0"
  }
}
```

---

## 📌 Step 8: รัน Server และทดสอบ

```bash
# รัน server
npm run dev

# Output:
# 🚀 GraphQL Server พร้อมใช้งานที่: http://localhost:4000/
# 📊 เปิด GraphQL Explorer ที่: http://localhost:4000/
```

เปิด Browser ไปที่ `http://localhost:4000/` จะเห็น **Apollo Sandbox** (GraphQL playground)

---

## 📌 Step 9: Query แรกของคุณ

ใน Apollo Sandbox พิมพ์:

```graphql
# Query 1: Hello World
query {
  hello
}
```

**Response:**
```json
{
  "data": {
    "hello": "สวัสดี GraphQL! 🎉"
  }
}
```

```graphql
# Query 2: Greeting with argument
query {
  greeting(name: "สมชาย")
}
```

**Response:**
```json
{
  "data": {
    "greeting": "สวัสดี สมชาย!"
  }
}
```

---

## 📌 Step 10: ทดลองเพิ่ม Type ซับซ้อนขึ้น

แก้ไข `src/schema.js`:

```javascript
export const typeDefs = `#graphql
  type User {
    id: ID!
    name: String!
    email: String!
    age: Int
    isActive: Boolean!
  }

  type Query {
    hello: String
    greeting(name: String): String
    # ดึง user คนเดียว
    user(id: ID!): User
    # ดึง users ทั้งหมด
    users: [User!]!
  }
`;
```

แก้ไข `src/resolvers.js`:

```javascript
// ข้อมูล mock (จำลองฐานข้อมูล)
const users = [
  { id: "1", name: "สมชาย ใจดี", email: "somchai@example.com", age: 28, isActive: true },
  { id: "2", name: "สมหญิง สวยงาม", email: "somying@example.com", age: 25, isActive: true },
  { id: "3", name: "มานะ พยายาม", email: "mana@example.com", age: 32, isActive: false },
];

export const resolvers = {
  Query: {
    hello: () => "สวัสดี GraphQL! 🎉",
    greeting: (_, { name }) => `สวัสดี ${name || "ทุกคน"}!`,
    
    // ค้นหา user ตาม id
    user: (_, { id }) => users.find((u) => u.id === id),
    
    // ดึง users ทั้งหมด
    users: () => users,
  },
};
```

---

## 📌 Step 11: ทดสอบ Queries ใหม่

```graphql
# ดึง users ทั้งหมด
query GetAllUsers {
  users {
    id
    name
    email
    isActive
  }
}
```

**Response:**
```json
{
  "data": {
    "users": [
      { "id": "1", "name": "สมชาย ใจดี", "email": "somchai@example.com", "isActive": true },
      { "id": "2", "name": "สมหญิง สวยงาม", "email": "somying@example.com", "isActive": true },
      { "id": "3", "name": "มานะ พยายาม", "email": "mana@example.com", "isActive": false }
    ]
  }
}
```

```graphql
# ดึง user คนเดียว
query GetUser {
  user(id: "1") {
    name
    email
    age
  }
}
```

**Response:**
```json
{
  "data": {
    "user": {
      "name": "สมชาย ใจดี",
      "email": "somchai@example.com",
      "age": 28
    }
  }
}
```

---

## 📌 Step 12: GraphQL Introspection — สำรวจ Schema

GraphQL สามารถตรวจสอบ schema ของตัวเองได้:

```graphql
# ดู types ทั้งหมดใน schema
query IntrospectSchema {
  __schema {
    types {
      name
      kind
    }
  }
}
```

```graphql
# ดูรายละเอียดของ Type เฉพาะ
query IntrospectUser {
  __type(name: "User") {
    name
    fields {
      name
      type {
        name
        kind
      }
    }
  }
}
```

> **⚠️ Security Note:** Introspection ควรปิดใน Production! (เรียนรู้เพิ่มเติมใน Part 044)

---

## 📌 Step 13: GraphQL vs REST — เปรียบเทียบ

| Feature | REST | GraphQL |
|---------|------|---------|
| Endpoint | หลาย endpoints | 1 endpoint |
| Data fetching | Fixed response | Flexible (ขอเฉพาะที่ต้องการ) |
| Over-fetching | มีปัญหา | ไม่มีปัญหา |
| Under-fetching | มีปัญหา (N+1) | ไม่มีปัญหา |
| Versioning | ต้องทำ `/v1`, `/v2` | วิวัฒนาการ schema ได้ |
| Type safety | ไม่บังคับ | บังคับด้วย Schema |
| Real-time | ต้องใช้ Polling/SSE | Subscriptions built-in |
| Caching | HTTP Caching ง่าย | ซับซ้อนกว่า |
| Learning curve | ง่าย | ปานกลาง |

---

## 📌 Step 14: บริษัทใหญ่ที่ใช้ GraphQL

```
📱 Meta (Facebook)
   - ใช้ใน News Feed, Instagram, WhatsApp
   - ต้นกำเนิดของ GraphQL

🐙 GitHub
   - GitHub API v4 เป็น GraphQL ทั้งหมด
   - 15+ billion requests/วัน

🛒 Shopify
   - Storefront API, Admin API
   - สำคัญมากสำหรับ e-commerce ecosystem

🐦 Twitter/X
   - ใช้ GraphQL ภายใน

🎬 Netflix
   - Studio API, Partner API

🏠 Airbnb
   - Booking, Listing APIs

💳 PayPal
   - Payment processing APIs

🚗 Lyft
   - Ride-sharing APIs

📦 Amazon
   - AppSync (Managed GraphQL service)
```

---

## 📌 Step 15: สรุปและ Next Steps

### สิ่งที่เรียนรู้ใน Part 001

✅ GraphQL คืออะไร และทำไมถึงสร้างขึ้น  
✅ ปัญหาหลักที่ GraphQL แก้ไข (over-fetching, under-fetching)  
✅ 3 Operations หลัก: Query, Mutation, Subscription  
✅ ติดตั้งและรัน GraphQL Server แรก  
✅ ทดลอง Query จริง  

### Homework

1. ลอง query ทุก field ของ `User` type
2. ลอง query เฉพาะบาง field (สังเกตว่า response เปลี่ยนอย่างไร)
3. ลองเพิ่ม field ใหม่ใน User type เช่น `createdAt: String`
4. ลองเพิ่ม user ใหม่ใน mock data

### ในส่วนถัดไป

➡️ **[Part 002](./part-002.md)** — GraphQL vs REST: การเปรียบเทียบเชิงลึกและการเลือกใช้

---

## 📚 แหล่งอ้างอิง

- [GraphQL Official Specification](https://spec.graphql.org/)
- [GraphQL.org — Learn GraphQL](https://graphql.org/learn/)
- [Apollo GraphQL Documentation](https://www.apollographql.com/docs/)
- [How to GraphQL](https://www.howtographql.com/)

---

*Part 001 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
