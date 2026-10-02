# Part 010 — Arguments, Variables & Fragments 🔧

> **ระดับ:** Beginner-Intermediate | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 251–290

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Arguments ใน queries และ mutations
- Variables — dynamic values
- Default values
- Named queries/mutations
- Fragments — reusable field sets
- Inline fragments
- Named fragments
- @skip และ @include directives
- Fragment composition

---

## 📌 Step 251: Arguments พื้นฐาน

Arguments อนุญาตให้ส่งค่าไปยัง fields:

```graphql
# Argument ใน Query
query {
  user(id: "user_123") {
    name
    email
  }
  
  products(
    category: "electronics"
    minPrice: 1000
    maxPrice: 5000
    limit: 10
    offset: 0
  ) {
    name
    price
  }
}
```

### Arguments ใน Schema

```graphql
type Query {
  # Arguments หลายชนิด
  user(id: ID!): User
  
  products(
    category: String
    minPrice: Float
    maxPrice: Float
    limit: Int = 10       # ← default value
    offset: Int = 0       # ← default value
    sort: SortInput
  ): [Product!]!
  
  searchUsers(
    term: String!
    role: UserRole
    isActive: Boolean = true
  ): [User!]!
}

input SortInput {
  field: String!
  direction: SortDirection = ASC
}
```

### Resolver รับ Arguments

```javascript
const resolvers = {
  Query: {
    products: async (_, { 
      category,
      minPrice,
      maxPrice,
      limit = 10,
      offset = 0,
      sort,
    }) => {
      const where = {};
      
      if (category) where.category = category;
      if (minPrice !== undefined) where.price = { ...where.price, $gte: minPrice };
      if (maxPrice !== undefined) where.price = { ...where.price, $lte: maxPrice };
      
      const orderBy = sort 
        ? [[sort.field, sort.direction]]
        : [['createdAt', 'DESC']];
      
      return Product.findAll({
        where,
        order: orderBy,
        limit,
        offset,
      });
    },
  },
};
```

---

## 📌 Step 252: Variables — Dynamic Queries

แทนที่จะ hardcode values ใน query — ใช้ Variables:

```graphql
# ❌ Hardcoded (ไม่ดี)
query {
  user(id: "user_123") {
    name
  }
}

# ✅ ใช้ Variables (ดีกว่า)
query GetUser($userId: ID!) {
  user(id: $userId) {
    name
    email
  }
}
```

### ส่ง Variables ใน Request

```json
{
  "query": "query GetUser($userId: ID!) { user(id: $userId) { name email } }",
  "variables": {
    "userId": "user_123"
  }
}
```

### Variables ใน Apollo Client

```javascript
// React + Apollo Client
import { useQuery, gql } from '@apollo/client';

const GET_USER = gql`
  query GetUser($userId: ID!) {
    user(id: $userId) {
      id
      name
      email
      avatar
    }
  }
`;

function UserProfile({ userId }) {
  const { data, loading, error } = useQuery(GET_USER, {
    variables: { userId },
  });
  
  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;
  
  return (
    <div>
      <img src={data.user.avatar} alt={data.user.name} />
      <h1>{data.user.name}</h1>
      <p>{data.user.email}</p>
    </div>
  );
}
```

---

## 📌 Step 253: Variable Types & Default Values

```graphql
# ตัวอย่าง variables หลายชนิด
query SearchProducts(
  $term: String!        # required
  $category: String     # optional (nullable)
  $limit: Int = 10      # optional with default
  $offset: Int = 0      # optional with default
  $sort: SortInput      # optional object
  $includeInactive: Boolean = false
) {
  searchProducts(
    term: $term
    category: $category
    limit: $limit
    offset: $offset
    sort: $sort
    includeInactive: $includeInactive
  ) {
    id
    name
    price
    category { name }
  }
}
```

### Variables Object ที่ส่งมา

```json
{
  "term": "หูฟัง",
  "category": "electronics",
  "limit": 20,
  "sort": {
    "field": "price",
    "direction": "ASC"
  }
}
```

---

## 📌 Step 254: Named Operations

ควรตั้งชื่อ operations เสมอ (ดีสำหรับ debugging):

```graphql
# Named Query
query GetUserProfile($userId: ID!) {
  user(id: $userId) {
    name
    email
  }
}

# Named Mutation
mutation UpdateUserProfile($userId: ID!, $input: UpdateUserInput!) {
  updateUser(id: $userId, input: $input) {
    id
    name
    email
  }
}

# Named Subscription
subscription OnNewMessage($channelId: ID!) {
  newMessage(channelId: $channelId) {
    id
    content
    sender { name }
  }
}
```

### ทำไมต้องตั้งชื่อ

```
Benefits ของ Named Operations:
1. Server logs จะแสดงชื่อ operation แทน query string
2. APM tools (Datadog, New Relic) track ได้ดีกว่า
3. Persisted queries ใช้ชื่อเป็น key
4. Debug ง่ายกว่า
5. Code กอ่าน richer
```

---

## 📌 Step 255: Fragments — Reusable Field Sets

Fragment คือ reusable block ของ fields:

```graphql
# นิยาม fragment
fragment UserBasicInfo on User {
  id
  name
  email
  avatar
}

fragment UserFullInfo on User {
  ...UserBasicInfo
  bio
  role
  createdAt
  
  orders(limit: 5) {
    id
    total
    status
  }
}

# ใช้ fragment ใน query
query GetUsersPage {
  me {
    ...UserFullInfo
  }
  
  suggestedUsers: users(limit: 5) {
    ...UserBasicInfo
  }
  
  recentlyJoined: users(sort: { field: "createdAt", direction: DESC }, limit: 3) {
    ...UserBasicInfo
    createdAt
  }
}
```

### ประโยชน์ของ Fragments

```graphql
# ใช้ fragment เดิมในหลาย queries
fragment ProductCard on Product {
  id
  name
  price
  images(limit: 1) { url alt }
  inStock
  rating
}

query HomePage {
  featuredProducts {
    ...ProductCard
  }
  newArrivals {
    ...ProductCard
  }
  onSale {
    ...ProductCard
    discountedPrice
    discountPercent
  }
}

query SearchPage($term: String!) {
  searchProducts(term: $term) {
    ...ProductCard
    category { name }
    seller { name }
  }
}
```

---

## 📌 Step 256: Inline Fragments (สำหรับ Union/Interface)

```graphql
query GetSearchResults($term: String!) {
  search(term: $term) {
    # __typename บอกว่าเป็น type อะไร
    __typename
    
    # Inline fragment สำหรับ Union members
    ... on User {
      id
      name
      email
      avatar
    }
    
    ... on Product {
      id
      name
      price
      category { name }
    }
    
    ... on Article {
      id
      title
      excerpt
      publishedAt
      author { name }
    }
  }
}
```

### ใช้กับ Interface

```graphql
fragment NodeFields on Node {
  id
  createdAt
  updatedAt
}

query GetAllContent {
  content {
    ...NodeFields
    
    ... on Article {
      title
      content
    }
    
    ... on Video {
      title
      videoUrl
      duration
    }
    
    ... on Image {
      url
      width
      height
    }
  }
}
```

---

## 📌 Step 257: @skip และ @include Directives

```graphql
query GetUser(
  $userId: ID!
  $includeOrders: Boolean!
  $skipAvatar: Boolean!
) {
  user(id: $userId) {
    id
    name
    email
    
    # แสดง avatar เฉพาะเมื่อ $skipAvatar = false
    avatar @skip(if: $skipAvatar)
    
    # แสดง orders เฉพาะเมื่อ $includeOrders = true
    orders @include(if: $includeOrders) {
      id
      total
      status
      createdAt
    }
  }
}
```

### Variables

```json
{
  "userId": "user_123",
  "includeOrders": true,
  "skipAvatar": false
}
```

### ใน Apollo Client

```javascript
const GET_USER = gql`
  query GetUser($userId: ID!, $includeOrders: Boolean!, $skipAvatar: Boolean!) {
    user(id: $userId) {
      id
      name
      email
      avatar @skip(if: $skipAvatar)
      orders @include(if: $includeOrders) {
        id
        total
        status
      }
    }
  }
`;

function UserPage({ userId, showOrders }) {
  const { data } = useQuery(GET_USER, {
    variables: {
      userId,
      includeOrders: showOrders,
      skipAvatar: window.innerWidth < 480,  // ไม่โหลด avatar บน mobile
    },
  });
}
```

---

## 📌 Step 258: Fragment Composition Pattern

```graphql
# Atomic fragments
fragment AvatarFields on User {
  avatar
  avatarAlt
}

fragment ContactFields on User {
  email
  phone
}

fragment LocationFields on User {
  address {
    city
    country
  }
}

# Composite fragments
fragment UserCardFragment on User {
  id
  name
  ...AvatarFields
}

fragment UserProfileFragment on User {
  id
  name
  bio
  ...AvatarFields
  ...ContactFields
  ...LocationFields
}

fragment UserAdminFragment on User {
  ...UserProfileFragment
  role
  createdAt
  lastLoginAt
  isVerified
}

# Query ใช้ composite fragments
query AdminUserList {
  users {
    ...UserAdminFragment
  }
}

query PublicUserProfiles {
  users {
    ...UserProfileFragment
  }
}
```

---

## 📌 Step 259: Persisted Queries

Persisted queries ช่วยลด network payload โดยส่ง ID แทน full query:

```javascript
// Apollo Client — Automatic Persisted Queries (APQ)
import { ApolloClient, InMemoryCache, HttpLink } from '@apollo/client';
import { createPersistedQueryLink } from '@apollo/client/link/persisted-queries';
import { sha256 } from 'crypto-hash';

const persistedQueriesLink = createPersistedQueryLink({ sha256 });

const client = new ApolloClient({
  link: persistedQueriesLink.concat(
    new HttpLink({ uri: '/graphql' })
  ),
  cache: new InMemoryCache(),
});
```

### Flow ของ APQ

```
First request:
Client → { "extensions": { "persistedQuery": { "sha256Hash": "abc123..." } } }
Server → { "errors": [{ "message": "PersistedQueryNotFound" }] }
Client → { "query": "...", "extensions": { "persistedQuery": { "sha256Hash": "abc123..." } } }
Server → { "data": { ... } }  + เก็บ query ไว้ด้วย hash

Subsequent requests:
Client → { "extensions": { "persistedQuery": { "sha256Hash": "abc123..." } } }
Server → { "data": { ... } }  ← ไม่ต้องส่ง query string อีก!
```

---

## 📌 Step 260: สรุป Part 010

### เนื้อหาที่เรียนรู้

✅ Arguments พื้นฐาน  
✅ Variables — ทำ queries dynamic  
✅ Variable types และ defaults  
✅ Named operations  
✅ Fragments — reusable field sets  
✅ Inline fragments สำหรับ Union/Interface  
✅ @skip และ @include directives  
✅ Fragment composition pattern  
✅ Persisted queries  

### Homework

1. สร้าง query ด้วย variables สำหรับ paginated product list
2. สร้าง fragments สำหรับ User profile ที่ใช้ร่วมกันหลาย queries
3. ใช้ @skip/@include เพื่อ conditionally load heavy data

### ในส่วนถัดไป

➡️ **[Part 011](./part-011.md)** — Directives: @deprecated, @auth, Custom Directives

---

*Part 010 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
