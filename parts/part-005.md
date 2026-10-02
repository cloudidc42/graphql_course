# Part 005 — GraphQL Queries: ดึงข้อมูลทุกรูปแบบ

> **ระดับ:** Beginner | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 91–120

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Query syntax ทุกรูปแบบ
- Arguments และ Variables
- Named queries
- Aliases
- Fragments
- Directives @skip และ @include
- Multiple queries
- Field selection

---

## 📌 Step 91: Query Syntax พื้นฐาน

### Shorthand Query (ไม่มีชื่อ)

```graphql
# รูปแบบย่อ — ใช้ได้เฉพาะ query
{
  users {
    name
    email
  }
}
```

### Explicit Query

```graphql
# รูปแบบเต็ม — แนะนำ
query {
  users {
    name
    email
  }
}
```

### Named Query (Best Practice)

```graphql
# Named — ดีที่สุด เพราะ debug ง่าย
query GetUsers {
  users {
    name
    email
  }
}
```

---

## 📌 Step 92: Field Selection

GraphQL ให้คุณเลือก fields ที่ต้องการ:

```graphql
# ขอเฉพาะ name
query {
  users {
    name
  }
}

# ขอหลาย fields
query {
  users {
    id
    name
    email
    isActive
    role
  }
}

# ขอ nested fields
query {
  users {
    id
    name
    posts {
      title
      createdAt
    }
  }
}

# ขอ deeply nested
query {
  users {
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

---

## 📌 Step 93: Arguments

```graphql
# Argument แบบ literal
query {
  user(id: "1") {
    name
    email
  }
}

# Multiple arguments
query {
  posts(status: PUBLISHED, limit: 5, offset: 0) {
    title
    createdAt
  }
}

# Nested arguments
query {
  users {
    name
    # field ที่มี argument
    avatar(size: LARGE)
    posts(status: PUBLISHED) {
      title
    }
  }
}
```

---

## 📌 Step 94: Variables

Variables ทำให้ queries ใช้ซ้ำได้และปลอดภัยกว่า:

```graphql
# ✅ ดี — ใช้ Variables
query GetUser($userId: ID!) {
  user(id: $userId) {
    name
    email
  }
}

# Variables (JSON)
{
  "userId": "1"
}

# ❌ ไม่ดี — Hardcode ค่า (SQL injection risk)
query {
  user(id: "1") {
    name
  }
}
```

### Variable Types

```graphql
# Non-null variable
query GetUser($id: ID!) { ... }        # id ต้องส่งมาเสมอ

# Nullable variable
query GetPosts($tag: String) { ... }   # tag อาจไม่ส่งก็ได้

# Default value
query GetPosts($limit: Int = 10) { ... }  # default = 10

# List variable
query GetUsers($ids: [ID!]!) { ... }   # ต้องส่ง array of IDs

# Complex variables
query SearchPosts(
  $filter: PostFilterInput!
  $sort: PostSortField = CREATED_AT
  $order: SortOrder = DESC
  $page: Int = 1
  $perPage: Int = 10
) {
  posts(
    filter: $filter
    sort: $sort
    order: $order
    page: $page
    perPage: $perPage
  ) {
    id
    title
  }
}

# Variables
{
  "filter": {
    "status": "PUBLISHED",
    "tag": "graphql"
  },
  "sort": "CREATED_AT",
  "order": "DESC",
  "page": 1,
  "perPage": 5
}
```

---

## 📌 Step 95: Aliases

Aliases ช่วยให้ query field เดียวกันหลายครั้งด้วยชื่อต่างกัน:

```graphql
# ขอ published และ draft posts พร้อมกัน
query GetPosts {
  publishedPosts: posts(status: PUBLISHED) {
    id
    title
    createdAt
  }
  
  draftPosts: posts(status: DRAFT) {
    id
    title
    createdAt
  }
}

# Response
{
  "data": {
    "publishedPosts": [
      { "id": "1", "title": "...", "createdAt": "..." }
    ],
    "draftPosts": [
      { "id": "3", "title": "...", "createdAt": "..." }
    ]
  }
}
```

```graphql
# ดึง users หลายคนพร้อมกัน
query GetMultipleUsers {
  firstUser: user(id: "1") {
    name
    email
  }
  
  secondUser: user(id: "2") {
    name
    email
  }
}
```

```graphql
# Alias ใน nested fields
query UserDashboard {
  user(id: "1") {
    name
    recentPosts: posts(status: PUBLISHED, limit: 5) {
      title
    }
    draftCount: posts(status: DRAFT) {
      id
    }
  }
}
```

---

## 📌 Step 96: Fragments

Fragments ช่วยให้ใช้ field sets ซ้ำได้:

```graphql
# กำหนด Fragment
fragment UserBasicInfo on User {
  id
  name
  email
  isActive
}

fragment UserFullInfo on User {
  ...UserBasicInfo   # ← spread fragment
  age
  role
  createdAt
  profile {
    bio
    website
  }
}

fragment PostInfo on Post {
  id
  title
  createdAt
  author {
    ...UserBasicInfo   # ← ใช้ fragment ซ้ำ
  }
}

# ใช้ fragments ใน query
query GetUsersAndPosts {
  users {
    ...UserBasicInfo
  }
  
  posts {
    ...PostInfo
  }
}

query GetUserDetails {
  user(id: "1") {
    ...UserFullInfo
    posts {
      ...PostInfo
    }
  }
}
```

---

## 📌 Step 97: Inline Fragments

Inline fragments ใช้กับ Interface และ Union types:

```graphql
union SearchResult = User | Post | Comment

query Search {
  search(term: "graphql") {
    __typename   # บอกว่า object นี้เป็น type อะไร
    
    ... on User {
      id
      name
      email
    }
    
    ... on Post {
      id
      title
      author {
        name
      }
    }
    
    ... on Comment {
      id
      text
      post {
        title
      }
    }
  }
}
```

```graphql
# ใช้กับ Interface
interface Node {
  id: ID!
}

query GetNode {
  node(id: "user_1") {
    id   # มีทุก Node
    
    # fields เฉพาะ type
    ... on User {
      name
      email
    }
    ... on Post {
      title
    }
  }
}
```

---

## 📌 Step 98: Directives @skip และ @include

```graphql
# @skip(if: Boolean) — skip field ถ้า true
query UserProfile($hideEmail: Boolean!) {
  user(id: "1") {
    name
    email @skip(if: $hideEmail)      # ← ข้าม email ถ้า hideEmail = true
    phone @skip(if: $hideEmail)
  }
}

# @include(if: Boolean) — include field ถ้า true
query UserProfile($showPosts: Boolean!) {
  user(id: "1") {
    name
    posts @include(if: $showPosts) {  # ← รวม posts ถ้า showPosts = true
      title
    }
  }
}
```

```graphql
# ตัวอย่างจริง: dashboard query ที่ปรับได้
query Dashboard(
  $showAnalytics: Boolean!
  $showRecentOrders: Boolean!
  $showTopProducts: Boolean!
  $isAdmin: Boolean!
) {
  # แสดงสถิติ
  analytics @include(if: $showAnalytics) {
    totalUsers
    totalRevenue
    dailyActiveUsers
  }
  
  # คำสั่งซื้อล่าสุด
  recentOrders @include(if: $showRecentOrders) {
    id
    total
    status
  }
  
  # สินค้าขายดี
  topProducts @include(if: $showTopProducts) {
    name
    salesCount
  }
  
  # Admin-only data
  systemHealth @include(if: $isAdmin) {
    cpuUsage
    memoryUsage
  }
}
```

---

## 📌 Step 99: Multiple Queries

สามารถส่งหลาย queries พร้อมกัน (Batching):

```graphql
# Query 1
query GetUser {
  user(id: "1") {
    name
  }
}

# Query 2
query GetPosts {
  posts {
    title
  }
}
```

ปกติ GraphQL จะ execute ทีละ operation ไม่สามารถส่งหลาย operations ในครั้งเดียวได้โดยตรง ต้องใช้ `operationName` หรือ batching:

```javascript
// Client-side batching
const response = await fetch('/graphql', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify([
    { query: '{ user(id: "1") { name } }' },
    { query: '{ posts { title } }' },
  ])
});
```

---

## 📌 Step 100: Query ขั้นสูง — Pagination

### Offset-based Pagination

```graphql
query GetPosts($page: Int!, $perPage: Int!) {
  posts(page: $page, perPage: $perPage) {
    id
    title
    createdAt
  }
}

# Variables
{ "page": 2, "perPage": 10 }
```

### Cursor-based Pagination (Relay Style)

```graphql
query GetPostsFeed($first: Int!, $after: String) {
  postsConnection(first: $first, after: $after) {
    edges {
      node {
        id
        title
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}

# Variables (first page)
{ "first": 10, "after": null }

# Variables (next page — ใช้ endCursor จาก response ก่อน)
{ "first": 10, "after": "cursor_value_from_previous_response" }
```

---

## 📌 Step 101: Query ขั้นสูง — Search

```graphql
query SearchContent(
  $term: String!
  $type: SearchType
  $filters: SearchFilters
) {
  search(term: $term, type: $type, filters: $filters) {
    results {
      ... on User {
        id
        name
        email
        _score: relevanceScore   # alias
      }
      ... on Post {
        id
        title
        excerpt
        _score: relevanceScore
      }
    }
    totalCount
    took        # milliseconds
    suggestions # "Did you mean...?"
  }
}
```

---

## 📌 Step 102: เขียน Resolvers สำหรับ Queries ซับซ้อน

### Setup ข้อมูลครบสมบูรณ์

```javascript
// src/data/index.js
export const db = {
  users: [
    {
      id: '1',
      name: 'สมชาย ใจดี',
      email: 'somchai@example.com',
      age: 28,
      isActive: true,
      role: 'ADMIN',
      createdAt: '2024-01-01T00:00:00Z',
    },
    {
      id: '2',
      name: 'สมหญิง สวยงาม',
      email: 'somying@example.com',
      age: 25,
      isActive: true,
      role: 'USER',
      createdAt: '2024-02-01T00:00:00Z',
    },
    {
      id: '3',
      name: 'มานะ พยายาม',
      email: 'mana@example.com',
      age: 32,
      isActive: false,
      role: 'USER',
      createdAt: '2024-03-01T00:00:00Z',
    },
  ],

  posts: [
    {
      id: '1',
      title: 'เริ่มต้นกับ GraphQL',
      content: 'GraphQL เป็นภาษา query ที่ยืดหยุ่นและมีประสิทธิภาพ...',
      authorId: '1',
      tags: ['graphql', 'api', 'tutorial'],
      status: 'PUBLISHED',
      viewCount: 250,
      createdAt: '2024-03-01T00:00:00Z',
    },
    {
      id: '2',
      title: 'Node.js Best Practices',
      content: 'สิ่งที่ developer ทุกคนควรรู้เกี่ยวกับ Node.js...',
      authorId: '1',
      tags: ['nodejs', 'javascript', 'backend'],
      status: 'PUBLISHED',
      viewCount: 180,
      createdAt: '2024-04-01T00:00:00Z',
    },
    {
      id: '3',
      title: 'React Hooks ทำความเข้าใจ',
      content: 'Hooks เปลี่ยนวิธีที่เราเขียน React components...',
      authorId: '2',
      tags: ['react', 'javascript', 'frontend'],
      status: 'DRAFT',
      viewCount: 0,
      createdAt: '2024-05-01T00:00:00Z',
    },
    {
      id: '4',
      title: 'TypeScript Tips',
      content: 'Tips สำหรับ TypeScript ที่ทำให้โค้ดดีขึ้น...',
      authorId: '2',
      tags: ['typescript', 'javascript'],
      status: 'PUBLISHED',
      viewCount: 120,
      createdAt: '2024-06-01T00:00:00Z',
    },
  ],

  comments: [
    {
      id: '1',
      text: 'บทความดีมากเลย!',
      authorId: '2',
      postId: '1',
      createdAt: '2024-03-02T00:00:00Z',
    },
    {
      id: '2',
      text: 'ขอบคุณสำหรับข้อมูล',
      authorId: '3',
      postId: '1',
      createdAt: '2024-03-03T00:00:00Z',
    },
    {
      id: '3',
      text: 'อยากให้เพิ่มตัวอย่างมากกว่านี้',
      authorId: '1',
      postId: '2',
      createdAt: '2024-04-02T00:00:00Z',
    },
  ],
};
```

### Resolvers ครบสมบูรณ์

```javascript
// src/resolvers/query.js
import { db } from '../data/index.js';

export const queryResolvers = {
  Query: {
    // ดึง users พร้อม pagination
    users: (_, { page = 1, perPage = 10, status, role }) => {
      let results = db.users;
      
      // Filter
      if (status) {
        results = results.filter(u => 
          (status === 'ACTIVE') === u.isActive
        );
      }
      if (role) {
        results = results.filter(u => u.role === role);
      }
      
      // Pagination
      const total = results.length;
      const start = (page - 1) * perPage;
      const end = start + perPage;
      const users = results.slice(start, end);
      
      return {
        users,
        total,
        page,
        perPage,
        hasNextPage: end < total,
      };
    },

    // ดึง user คนเดียว
    user: (_, { id }) => {
      const user = db.users.find(u => u.id === id);
      if (!user) return null;
      return user;
    },

    // ดึง posts พร้อม filters
    posts: (_, { tag, status, authorId, page = 1, perPage = 10 }) => {
      let results = db.posts;
      
      if (tag) results = results.filter(p => p.tags.includes(tag));
      if (status) results = results.filter(p => p.status === status);
      if (authorId) results = results.filter(p => p.authorId === authorId);
      
      // Sort by createdAt DESC
      results = results.sort((a, b) => 
        new Date(b.createdAt) - new Date(a.createdAt)
      );
      
      const total = results.length;
      const start = (page - 1) * perPage;
      const end = start + perPage;
      
      return {
        posts: results.slice(start, end),
        total,
        page,
        perPage,
        hasNextPage: end < total,
      };
    },

    // ดึง post คนเดียว
    post: (_, { id }) => db.posts.find(p => p.id === id) || null,

    // Search
    search: (_, { term }) => {
      const lowerTerm = term.toLowerCase();
      const results = [];
      
      // Search users
      db.users
        .filter(u => 
          u.name.toLowerCase().includes(lowerTerm) ||
          u.email.toLowerCase().includes(lowerTerm)
        )
        .forEach(u => results.push({ ...u, __typename: 'User' }));
      
      // Search posts
      db.posts
        .filter(p => 
          p.title.toLowerCase().includes(lowerTerm) ||
          p.content.toLowerCase().includes(lowerTerm)
        )
        .forEach(p => results.push({ ...p, __typename: 'Post' }));
      
      return results;
    },
  },

  // Nested resolvers
  User: {
    posts: (user, { status, limit = 10 }) => {
      let posts = db.posts.filter(p => p.authorId === user.id);
      if (status) posts = posts.filter(p => p.status === status);
      return posts.slice(0, limit);
    },
    
    comments: (user) => 
      db.comments.filter(c => c.authorId === user.id),
  },

  Post: {
    author: (post) => db.users.find(u => u.id === post.authorId),
    
    comments: (post) => 
      db.comments.filter(c => c.postId === post.id),
    
    // Computed field
    excerpt: (post) => 
      post.content.substring(0, 150) + 
      (post.content.length > 150 ? '...' : ''),
  },

  Comment: {
    author: (comment) => db.users.find(u => u.id === comment.authorId),
    post: (comment) => db.posts.find(p => p.id === comment.postId),
  },
};
```

---

## 📌 Step 103: ทดสอบทุก Query Pattern

```graphql
# Test 1: Basic query
query BasicQuery {
  users {
    users {
      id
      name
      email
    }
    total
  }
}
```

```graphql
# Test 2: With variables
query GetUserById($id: ID!) {
  user(id: $id) {
    name
    email
    isActive
    role
  }
}
# Variables: { "id": "1" }
```

```graphql
# Test 3: Nested query
query UserWithEverything($userId: ID!) {
  user(id: $userId) {
    name
    email
    posts {
      title
      status
      tags
      comments {
        text
        author {
          name
        }
      }
    }
  }
}
# Variables: { "userId": "1" }
```

```graphql
# Test 4: Aliases
query CompareUsers {
  admin: user(id: "1") {
    name
    role
  }
  regular: user(id: "2") {
    name
    role
  }
}
```

```graphql
# Test 5: Fragments
fragment PostSummary on Post {
  id
  title
  status
  createdAt
}

query AllPostsWithFragments {
  posts {
    posts {
      ...PostSummary
      author {
        name
      }
    }
  }
}
```

```graphql
# Test 6: @include directive
query ConditionalData(
  $showPosts: Boolean!
  $showComments: Boolean!
) {
  user(id: "1") {
    name
    email
    posts @include(if: $showPosts) {
      title
    }
  }
  
  posts {
    posts {
      title
      comments @include(if: $showComments) {
        text
      }
    }
  }
}
# Variables: { "showPosts": true, "showComments": false }
```

---

## 📌 Step 104: Query Optimization

### ปัญหา N+1

```javascript
// ❌ ปัญหา: N+1 queries
const resolvers = {
  Post: {
    // ทุก post จะเรียก getUserById แยกกัน
    // 10 posts = 10 database queries!
    author: (post) => db.getUserById(post.authorId),
  },
};
```

```javascript
// ✅ แก้ด้วย DataLoader (เรียนรู้เพิ่มเติมใน Part 018)
import DataLoader from 'dataloader';

const userLoader = new DataLoader(async (ids) => {
  // 1 query สำหรับทุก ids
  const users = await db.getUsersByIds(ids);
  return ids.map(id => users.find(u => u.id === id));
});

const resolvers = {
  Post: {
    // ตอนนี้ batch 10 posts = 1 query!
    author: (post) => userLoader.load(post.authorId),
  },
};
```

---

## 📌 Step 105: Field Resolver Execution Order

```graphql
query {
  user(id: "1") {    # 1. Query.user resolver เรียกก่อน
    name             # 2. User.name (scalar — ไม่มี resolver เฉพาะ)
    posts {          # 3. User.posts resolver
      title          # 4. Post.title (scalar)
      author {       # 5. Post.author resolver
        name         # 6. User.name (scalar)
      }
    }
  }
}
```

```javascript
// Resolver execution ทำงานแบบ tree traversal (DFS)
// GraphQL execute resolvers แบบ parallel สำหรับ sibling fields

// เช่น:
const resolvers = {
  Query: {
    user: (_, { id }) => {
      console.log('1. Resolving Query.user');
      return db.users.find(u => u.id === id);
    },
  },
  User: {
    posts: (user) => {
      console.log('2. Resolving User.posts');
      return db.posts.filter(p => p.authorId === user.id);
    },
  },
  Post: {
    author: (post) => {
      console.log('3. Resolving Post.author');
      return db.users.find(u => u.id === post.authorId);
    },
  },
};
```

---

## 📌 Step 106: Context ใน Resolvers

```javascript
// Context ถูกส่งไปทุก resolver
const resolvers = {
  Query: {
    me: async (_, __, context) => {
      // context มาจาก contextValue ที่กำหนดตอน setup server
      if (!context.currentUser) {
        throw new Error('กรุณา login ก่อน');
      }
      return context.dataSources.users.getUserById(context.currentUser.id);
    },
  },
};

// Setup context
const server = new ApolloServer({
  typeDefs,
  resolvers,
});

startStandaloneServer(server, {
  context: async ({ req }) => {
    // Extract user จาก JWT token
    const token = req.headers.authorization?.replace('Bearer ', '');
    const currentUser = token ? verifyToken(token) : null;
    
    return {
      currentUser,
      dataSources: {
        users: new UserDataSource(),
        posts: new PostDataSource(),
      },
    };
  },
});
```

---

## 📌 Step 107: Resolver Arguments (4 ตัว)

```javascript
const resolvers = {
  Query: {
    user: (
      parent,    // parent object (null สำหรับ root Query)
      args,      // arguments ที่ client ส่งมา
      context,   // shared context
      info       // query execution info
    ) => {
      console.log('parent:', parent);    // null
      console.log('args:', args);         // { id: "1" }
      console.log('context:', context);   // { currentUser, dataSources }
      console.log('info.fieldName:', info.fieldName);  // "user"
      
      return db.users.find(u => u.id === args.id);
    },
  },

  User: {
    posts: (
      parent,   // user object จาก parent resolver
      args,
      context,
      info
    ) => {
      console.log('parent.id:', parent.id);  // user id
      return db.posts.filter(p => p.authorId === parent.id);
    },
  },
};
```

---

## 📌 Step 108: Query Complexity ใน Client

### ใน React

```javascript
import { useQuery, gql } from '@apollo/client';

// กำหนด query
const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      name
      email
      posts {
        id
        title
      }
    }
  }
`;

function UserProfile({ userId }) {
  const { data, loading, error } = useQuery(GET_USER, {
    variables: { id: userId },
  });
  
  if (loading) return <p>กำลังโหลด...</p>;
  if (error) return <p>เกิดข้อผิดพลาด: {error.message}</p>;
  
  return (
    <div>
      <h1>{data.user.name}</h1>
      <p>{data.user.email}</p>
      <ul>
        {data.user.posts.map(post => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

### ใน Vanilla JavaScript

```javascript
async function fetchUser(userId) {
  const response = await fetch('http://localhost:4000/graphql', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      query: `
        query GetUser($id: ID!) {
          user(id: $id) {
            name
            email
            posts {
              title
            }
          }
        }
      `,
      variables: { id: userId },
    }),
  });
  
  const { data, errors } = await response.json();
  
  if (errors) {
    throw new Error(errors[0].message);
  }
  
  return data.user;
}

// ใช้งาน
const user = await fetchUser('1');
console.log(user.name); // สมชาย ใจดี
```

---

## 📌 Step 109: ดึงข้อมูลหลายรูปแบบพร้อมกัน

```graphql
# Dashboard query — ดึงหลายอย่างพร้อมกัน
query Dashboard {
  # Stats
  userStats: users {
    total
  }
  
  # Recent posts
  recentPosts: posts(page: 1, perPage: 5) {
    posts {
      id
      title
      createdAt
      author {
        name
      }
    }
  }
  
  # Featured users
  admins: users(role: ADMIN) {
    users {
      name
      email
    }
  }
}
```

---

## 📌 Step 110: Query Debugging

### การ Debug ด้วย info parameter

```javascript
const resolvers = {
  Query: {
    users: (_, args, context, info) => {
      // ดู fields ที่ client ขอ
      const requestedFields = info.fieldNodes[0].selectionSet.selections
        .map(s => s.name.value);
      
      console.log('Requested fields:', requestedFields);
      // ['id', 'name', 'email'] ← ดึงเฉพาะที่ client ขอ
      
      // สามารถ optimize query ได้ตาม fields ที่ต้องการ
      return db.users.getAll({ fields: requestedFields });
    },
  },
};
```

### Apollo Tracing

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    // เปิด tracing ใน development
    ...(process.env.NODE_ENV === 'development' ? [{
      async requestDidStart() {
        return {
          async willSendResponse({ response }) {
            // เพิ่ม timing info ใน response
            const extensions = response.body.singleResult?.extensions;
            if (extensions?.tracing) {
              console.log('Query timing:', extensions.tracing);
            }
          },
        };
      },
    }] : []),
  ],
});
```

---

## 📌 Step 111: สรุป Part 005

### Query Cheatsheet

```graphql
# Basic
query { users { name } }

# Named
query GetUsers { users { name } }

# With variable
query GetUser($id: ID!) { user(id: $id) { name } }

# With default
query GetPosts($limit: Int = 10) { posts(limit: $limit) { title } }

# Alias
query { published: posts(status: PUBLISHED) { title } }

# Fragment
fragment F on User { name email }
query { users { ...F } }

# Inline fragment
query { search(term: "foo") { ... on User { name } ... on Post { title } } }

# @include
query ShowPosts($show: Boolean!) { 
  users { posts @include(if: $show) { title } } 
}

# @skip
query HideEmail($hide: Boolean!) {
  users { email @skip(if: $hide) }
}
```

### เนื้อหาที่เรียนรู้

✅ Query syntax ทุกรูปแบบ  
✅ Variables และ default values  
✅ Aliases  
✅ Fragments และ Inline Fragments  
✅ @skip และ @include directives  
✅ Nested queries  
✅ Resolver execution order  
✅ Context ใน resolvers  

### Homework

1. สร้าง query สำหรับ "User Activity Feed" ที่แสดงโพสต์และ comments ล่าสุดของ user
2. ใช้ fragments เพื่อ reuse field sets
3. สร้าง query ที่ใช้ @include สำหรับ role-based data loading

### ในส่วนถัดไป

➡️ **[Part 006](./part-006.md)** — Mutations: สร้าง อ่าน แก้ไข ลบข้อมูล

---

*Part 005 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
