# Part 004 — Schema Definition Language (SDL)

> **ระดับ:** Beginner | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 61–90

---

## 🎯 สิ่งที่จะได้เรียนรู้

- SDL syntax ครบทุกส่วน
- Scalar types ทุกชนิด
- Object types, Input types, Enum, Interface, Union
- Non-null และ List types
- Descriptions และ Deprecation
- Custom scalars
- Schema directives

---

## 📌 Step 61: SDL คืออะไร

**Schema Definition Language (SDL)** คือภาษาสำหรับกำหนดโครงสร้างของ GraphQL API

```graphql
# นี่คือ SDL
type User {
  id: ID!
  name: String!
  email: String!
}
```

SDL กำหนด:
1. **Types** — ชนิดของข้อมูล
2. **Fields** — ข้อมูลแต่ละชิ้นใน type
3. **Relationships** — ความสัมพันธ์ระหว่าง types
4. **Operations** — Query, Mutation, Subscription ที่ทำได้

---

## 📌 Step 62: Scalar Types พื้นฐาน

GraphQL มี 5 built-in scalars:

```graphql
type ScalarExamples {
  # String — ข้อความ UTF-8
  name: String
  description: String
  
  # Int — จำนวนเต็ม 32-bit
  age: Int
  count: Int
  
  # Float — ทศนิยม 64-bit
  price: Float
  rating: Float
  
  # Boolean — true/false
  isActive: Boolean
  isVerified: Boolean
  
  # ID — unique identifier (ส่งเป็น string เสมอ)
  id: ID
  userId: ID
}
```

### ตัวอย่างค่าจริง

```graphql
# String
name: "สมชาย ใจดี"
emoji: "🚀"
multiline: "บรรทัดแรก\nบรรทัดสอง"

# Int
age: 28
negativeTemp: -5
bigNumber: 2147483647  # max 32-bit int

# Float
price: 99.99
pi: 3.14159265
negative: -273.15

# Boolean
isActive: true
hasProfile: false

# ID (serialized as String)
id: "1"
id: "550e8400-e29b-41d4-a716-446655440000"
id: "user_abc123"
```

---

## 📌 Step 63: Non-null และ List

### Non-null `!`

```graphql
type User {
  id: ID!          # ห้ามเป็น null เด็ดขาด
  name: String!    # ห้ามเป็น null เด็ดขาด
  nickname: String # อาจเป็น null ได้
  age: Int         # อาจเป็น null ได้
}
```

### List `[]`

```graphql
type User {
  tags: [String]      # List ที่เป็น null ได้, elements เป็น null ได้
  emails: [String!]   # List ที่เป็น null ได้, elements ห้ามเป็น null
  roles: [String]!    # List ห้ามเป็น null, elements เป็น null ได้
  perms: [String!]!   # List ห้ามเป็น null, elements ห้ามเป็น null
}
```

### เข้าใจความแตกต่าง

```graphql
# tags: [String]
null           ✓ (list เป็น null ได้)
[]             ✓ (list ว่างได้)
["a", null]    ✓ (element เป็น null ได้)
["a", "b"]     ✓

# emails: [String!]
null           ✓ (list เป็น null ได้)
[]             ✓
["a", null]    ✗ (element ห้ามเป็น null)
["a", "b"]     ✓

# roles: [String]!
null           ✗ (list ห้ามเป็น null)
[]             ✓
["a", null]    ✓ (element เป็น null ได้)
["a", "b"]     ✓

# perms: [String!]!
null           ✗
[]             ✓
["a", null]    ✗
["a", "b"]     ✓ ← ปลอดภัยที่สุด
```

---

## 📌 Step 64: Object Types

```graphql
"""ผู้ใช้งานในระบบ"""
type User {
  """รหัสเฉพาะ — ไม่ซ้ำกัน"""
  id: ID!
  
  """ชื่อ-นามสกุล"""
  name: String!
  
  """อีเมล — unique"""
  email: String!
  
  """อายุ — optional"""
  age: Int
  
  """สถานะการใช้งาน"""
  isActive: Boolean!
  
  """บทบาทในระบบ"""
  role: UserRole!
  
  """โพสต์ทั้งหมด"""
  posts: [Post!]!
  
  """เพื่อน"""
  friends: [User!]!
  
  """โปรไฟล์เพิ่มเติม"""
  profile: UserProfile
  
  """วันที่สร้าง"""
  createdAt: String!
}

"""โปรไฟล์เพิ่มเติมของ User"""
type UserProfile {
  bio: String
  website: String
  location: String
  profilePicture: String
}
```

---

## 📌 Step 65: Enum Types

```graphql
"""สถานะของ User"""
enum UserStatus {
  ACTIVE
  INACTIVE
  SUSPENDED
  PENDING_VERIFICATION
}

"""บทบาทของ User"""
enum UserRole {
  ADMIN
  MODERATOR
  USER
  GUEST
}

"""สถานะของ Post"""
enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
  DELETED
}

"""ลำดับการเรียง"""
enum SortOrder {
  ASC
  DESC
}

"""ฟิลด์สำหรับ sort users"""
enum UserSortField {
  NAME
  EMAIL
  CREATED_AT
  UPDATED_AT
}
```

### ใช้ Enum ใน Query

```graphql
type Query {
  users(
    status: UserStatus
    role: UserRole
    sortBy: UserSortField
    order: SortOrder = DESC
  ): [User!]!
}
```

```graphql
# ตัวอย่าง query
query {
  users(
    status: ACTIVE
    role: USER
    sortBy: CREATED_AT
    order: DESC
  ) {
    name
    email
  }
}
```

---

## 📌 Step 66: Input Types

Input types ใช้สำหรับ arguments ใน mutations:

```graphql
"""Input สำหรับสร้าง user ใหม่"""
input CreateUserInput {
  name: String!
  email: String!
  password: String!
  age: Int
  role: UserRole = USER
}

"""Input สำหรับแก้ไข user (fields ทั้งหมด optional)"""
input UpdateUserInput {
  name: String
  email: String
  age: Int
  isActive: Boolean
  role: UserRole
}

"""Input สำหรับ filter users"""
input UserFilterInput {
  status: UserStatus
  role: UserRole
  nameContains: String
  emailContains: String
  ageMin: Int
  ageMax: Int
  createdAfter: String
  createdBefore: String
}

"""Input สำหรับ pagination"""
input PaginationInput {
  page: Int = 1
  perPage: Int = 10
}

"""Input สำหรับ sorting"""
input SortInput {
  field: UserSortField = CREATED_AT
  order: SortOrder = DESC
}
```

### ความแตกต่างระหว่าง Input Type และ Object Type

```graphql
# Object Type — ใช้สำหรับ response (output)
type User {
  id: ID!
  name: String!
  posts: [Post!]!    # ← field สามารถเป็น Object type ได้
}

# Input Type — ใช้สำหรับ arguments (input)
input CreateUserInput {
  name: String!
  # posts: ???       # ← ใส่ Object type ใน input ไม่ได้!
  # ต้องใช้ postIds: [ID!] แทน
}
```

---

## 📌 Step 67: Interface Types

Interface กำหนด fields ที่ต้องมีเหมือนกัน:

```graphql
"""Node interface — ทุก entity ต้องมี id"""
interface Node {
  id: ID!
}

"""Timestamped interface — มี timestamps"""
interface Timestamped {
  createdAt: String!
  updatedAt: String!
}

"""Searchable interface — ค้นหาได้"""
interface Searchable {
  searchText: String!
}

"""User implements หลาย interfaces"""
type User implements Node & Timestamped & Searchable {
  id: ID!              # จาก Node
  createdAt: String!   # จาก Timestamped
  updatedAt: String!   # จาก Timestamped
  searchText: String!  # จาก Searchable
  
  # fields เฉพาะของ User
  name: String!
  email: String!
  role: UserRole!
}

type Post implements Node & Timestamped & Searchable {
  id: ID!
  createdAt: String!
  updatedAt: String!
  searchText: String!
  
  title: String!
  content: String!
  author: User!
}
```

### ใช้ Interface ใน Query

```graphql
type Query {
  # Query ที่รับ interface
  node(id: ID!): Node
  search(term: String!): [Searchable!]!
}
```

```graphql
# Client query ด้วย inline fragment
query SearchEverything {
  search(term: "graphql") {
    # fields จาก Searchable interface
    searchText
    
    # inline fragment สำหรับ User
    ... on User {
      name
      email
    }
    
    # inline fragment สำหรับ Post
    ... on Post {
      title
      author {
        name
      }
    }
  }
}
```

---

## 📌 Step 68: Union Types

Union คือ "อาจเป็น type ใดก็ได้"

```graphql
"""ผลการค้นหา — อาจเป็น User หรือ Post"""
union SearchResult = User | Post | Comment

"""Response จาก mutation — success หรือ error"""
union UserResult = User | UserError

"""UserError type"""
type UserError {
  code: String!
  message: String!
  field: String
}
```

### Union ใน Query

```graphql
type Query {
  search(term: String!): [SearchResult!]!
}

type Mutation {
  createUser(input: CreateUserInput!): UserResult!
}
```

```graphql
# Client ต้องใช้ inline fragment เสมอ
query Search {
  search(term: "สมชาย") {
    __typename    # ← บอกว่าเป็น type อะไร
    
    ... on User {
      id
      name
      email
    }
    ... on Post {
      id
      title
      author { name }
    }
    ... on Comment {
      id
      text
    }
  }
}

# Mutation ที่รับ Union result
mutation CreateUser {
  createUser(input: { name: "ใหม่", email: "mai@example.com", password: "pass" }) {
    ... on User {
      id
      name
    }
    ... on UserError {
      code
      message
      field
    }
  }
}
```

---

## 📌 Step 69: Fields with Arguments

```graphql
type User {
  id: ID!
  name: String!
  
  # posts ที่ filter ได้
  posts(
    status: PostStatus
    limit: Int = 10
    offset: Int = 0
  ): [Post!]!
  
  # avatar ขนาดต่างๆ
  avatar(size: AvatarSize = MEDIUM): String
  
  # formatted date
  createdAt(format: DateFormat = ISO): String!
}

enum AvatarSize {
  SMALL    # 32x32
  MEDIUM   # 64x64
  LARGE    # 128x128
  ORIGINAL
}

enum DateFormat {
  ISO      # 2024-01-15T10:30:00Z
  SHORT    # 15/01/2024
  LONG     # 15 มกราคม 2024
  RELATIVE # 2 วันที่แล้ว
}
```

```graphql
# ใช้งาน
query UserDetails {
  user(id: "1") {
    name
    avatar(size: LARGE)
    createdAt(format: RELATIVE)
    posts(status: PUBLISHED, limit: 5) {
      title
    }
  }
}
```

---

## 📌 Step 70: Schema Comments และ Descriptions

```graphql
"""
# UserConnection

Paginated list of users.

## Usage

Use this type when you need paginated access to user data.

## Fields

- `users` — The actual user list
- `pageInfo` — Pagination metadata
- `totalCount` — Total number of matching users
"""
type UserConnection {
  """List of users on this page"""
  users: [User!]!
  
  """Pagination information"""
  pageInfo: PageInfo!
  
  """Total number of users matching the filter"""
  totalCount: Int!
}

"""Information about the current page"""
type PageInfo {
  """Whether there are more pages after this one"""
  hasNextPage: Boolean!
  
  """Whether there are pages before this one"""
  hasPreviousPage: Boolean!
  
  """Cursor of the first item on this page"""
  startCursor: String
  
  """Cursor of the last item on this page"""
  endCursor: String
}
```

---

## 📌 Step 71: Deprecation

```graphql
type User {
  id: ID!
  name: String!
  
  # Field ที่ deprecated
  username: String @deprecated(reason: "ใช้ name แทน ตั้งแต่ version 2.0")
  
  # Old field
  fullName: String @deprecated(reason: "ใช้ name แทน")
  
  # New field
  displayName: String
}

type Query {
  # Query ที่ deprecated
  allUsers: [User!]! @deprecated(reason: "ใช้ users(page, perPage) แทน")
  
  # Query ใหม่
  users(page: Int = 1, perPage: Int = 10): UserConnection!
}
```

### ดู Deprecation ใน Introspection

```graphql
query CheckDeprecations {
  __type(name: "User") {
    fields(includeDeprecated: true) {
      name
      isDeprecated
      deprecationReason
    }
  }
}
```

---

## 📌 Step 72: Custom Scalars

```graphql
"""Date in ISO 8601 format (YYYY-MM-DD)"""
scalar Date

"""DateTime in ISO 8601 format"""
scalar DateTime

"""Email address"""
scalar EmailAddress

"""URL string"""
scalar URL

"""JSON object"""
scalar JSON

"""Positive integer"""
scalar PositiveInt

"""File upload"""
scalar Upload
```

### Implement Custom Scalars

```javascript
// src/scalars/date.scalar.js
import { GraphQLScalarType, Kind } from 'graphql';

export const DateScalar = new GraphQLScalarType({
  name: 'Date',
  description: 'Date in ISO 8601 format (YYYY-MM-DD)',

  // รับค่าจาก client (string) → JavaScript object
  parseValue(value) {
    if (typeof value !== 'string') {
      throw new Error('Date must be a string');
    }
    const date = new Date(value);
    if (isNaN(date.getTime())) {
      throw new Error(`Invalid date: ${value}`);
    }
    return date;
  },

  // ส่งค่าไป client (JavaScript object → string)
  serialize(value) {
    if (value instanceof Date) {
      return value.toISOString().split('T')[0];
    }
    throw new Error('Date scalar serializer expected a Date object');
  },

  // Parse literal จาก query string
  parseLiteral(ast) {
    if (ast.kind === Kind.STRING) {
      const date = new Date(ast.value);
      if (isNaN(date.getTime())) {
        throw new Error(`Invalid date literal: ${ast.value}`);
      }
      return date;
    }
    throw new Error('Date must be a string literal');
  },
});

// src/scalars/email.scalar.js
import { GraphQLScalarType, Kind } from 'graphql';

const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

export const EmailAddressScalar = new GraphQLScalarType({
  name: 'EmailAddress',
  description: 'A valid email address',

  parseValue(value) {
    if (typeof value !== 'string' || !EMAIL_REGEX.test(value)) {
      throw new Error(`Invalid email address: ${value}`);
    }
    return value.toLowerCase();
  },

  serialize(value) {
    if (typeof value !== 'string') {
      throw new Error('EmailAddress must be a string');
    }
    return value;
  },

  parseLiteral(ast) {
    if (ast.kind === Kind.STRING) {
      if (!EMAIL_REGEX.test(ast.value)) {
        throw new Error(`Invalid email: ${ast.value}`);
      }
      return ast.value.toLowerCase();
    }
    throw new Error('Email must be a string literal');
  },
});
```

### ลงทะเบียน Custom Scalars

```javascript
import { DateScalar } from './scalars/date.scalar.js';
import { EmailAddressScalar } from './scalars/email.scalar.js';

const resolvers = {
  // Custom scalar resolvers
  Date: DateScalar,
  EmailAddress: EmailAddressScalar,
  
  Query: { ... },
  Mutation: { ... },
};
```

### ใช้ Custom Scalars

```graphql
type User {
  id: ID!
  email: EmailAddress!
  birthDate: Date
  lastLogin: DateTime
}

type Query {
  users(createdAfter: Date): [User!]!
}
```

---

## 📌 Step 73: Built-in Directives

```graphql
# @deprecated — mark เป็น deprecated
type User {
  oldField: String @deprecated(reason: "ใช้ newField แทน")
  newField: String
}

# @skip — skip field ถ้า condition เป็น true
query UserQuery($includeEmail: Boolean!) {
  user(id: "1") {
    name
    email @skip(if: $includeEmail)   # skip ถ้า includeEmail = true
  }
}

# @include — include field ถ้า condition เป็น true
query UserQuery($showPosts: Boolean!) {
  user(id: "1") {
    name
    posts @include(if: $showPosts) {  # include ถ้า showPosts = true
      title
    }
  }
}
```

---

## 📌 Step 74: Custom Schema Directives

```graphql
# ประกาศ directive
directive @auth(requires: UserRole = USER) on FIELD_DEFINITION
directive @deprecated(reason: String = "No longer supported") on FIELD_DEFINITION | ENUM_VALUE
directive @uppercase on FIELD_DEFINITION
directive @rateLimit(max: Int!, window: Int!) on FIELD_DEFINITION
directive @cacheControl(maxAge: Int!, scope: CacheControlScope) on FIELD_DEFINITION | OBJECT

enum CacheControlScope {
  PUBLIC
  PRIVATE
}

type Query {
  # เฉพาะ ADMIN เข้าได้
  adminDashboard: AdminData @auth(requires: ADMIN)
  
  # Cache ได้ 60 วินาที
  publicPosts: [Post!]! @cacheControl(maxAge: 60, scope: PUBLIC)
  
  # Rate limit
  search(term: String!): [SearchResult!]! @rateLimit(max: 10, window: 60)
}

type User {
  # Transform เป็น uppercase
  name: String! @uppercase
  
  # Private data
  email: String! @auth(requires: USER)
  
  # Admin only
  internalNotes: String @auth(requires: ADMIN)
}
```

---

## 📌 Step 75: Schema Organization Pattern

### Pattern 1: SDL First (Recommended)

```javascript
// schema/user.graphql
type User {
  id: ID!
  name: String!
}

// schema/post.graphql
type Post {
  id: ID!
  title: String!
  author: User!
}

// schema/index.js — รวม schemas
import { readFileSync } from 'fs';
import { join } from 'path';

export const typeDefs = [
  readFileSync(join(__dirname, 'user.graphql'), 'utf-8'),
  readFileSync(join(__dirname, 'post.graphql'), 'utf-8'),
];
```

### Pattern 2: Tagged Template Literals

```javascript
import { gql } from 'graphql-tag';

export const userTypeDefs = gql`
  type User {
    id: ID!
    name: String!
  }
`;
```

### Pattern 3: mergeTypeDefs

```javascript
import { mergeTypeDefs } from '@graphql-tools/merge';
import { userTypeDefs } from './user.schema.js';
import { postTypeDefs } from './post.schema.js';
import { rootTypeDefs } from './root.schema.js';

export const typeDefs = mergeTypeDefs([
  rootTypeDefs,
  userTypeDefs,
  postTypeDefs,
]);
```

---

## 📌 Step 76: Schema Validation

```javascript
// ตรวจสอบ schema ก่อน deploy
import { buildSchema, validateSchema } from 'graphql';

const schemaString = `
  type Query {
    user(id: ID!): User
  }
  
  type User {
    id: ID!
    name: String!
  }
`;

try {
  const schema = buildSchema(schemaString);
  const errors = validateSchema(schema);
  
  if (errors.length > 0) {
    console.error('Schema errors:', errors);
    process.exit(1);
  }
  
  console.log('Schema is valid! ✅');
} catch (error) {
  console.error('Schema parse error:', error.message);
  process.exit(1);
}
```

---

## 📌 Step 77: ตัวอย่าง Schema จริง — E-commerce

```graphql
# E-commerce GraphQL Schema

scalar DateTime
scalar Decimal
scalar EmailAddress
scalar URL

# ========================
# Core Types
# ========================

type Product {
  id: ID!
  sku: String!
  name: String!
  description: String!
  price: Decimal!
  compareAtPrice: Decimal
  currency: Currency!
  images: [ProductImage!]!
  category: Category!
  tags: [String!]!
  inventory: InventoryInfo!
  variants: [ProductVariant!]!
  reviews: ProductReviews!
  isActive: Boolean!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type ProductImage {
  id: ID!
  url: URL!
  altText: String
  position: Int!
}

type ProductVariant {
  id: ID!
  title: String!
  sku: String!
  price: Decimal!
  inventory: Int!
  options: [VariantOption!]!
}

type VariantOption {
  name: String!
  value: String!
}

type InventoryInfo {
  quantity: Int!
  isAvailable: Boolean!
  lowStockThreshold: Int!
  isLowStock: Boolean!
}

type Category {
  id: ID!
  name: String!
  slug: String!
  description: String
  parent: Category
  children: [Category!]!
  products(page: Int = 1, perPage: Int = 20): ProductConnection!
  image: URL
}

type Order {
  id: ID!
  orderNumber: String!
  customer: Customer!
  items: [OrderItem!]!
  status: OrderStatus!
  subtotal: Decimal!
  tax: Decimal!
  shipping: Decimal!
  total: Decimal!
  shippingAddress: Address!
  billingAddress: Address!
  paymentMethod: PaymentMethod!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type OrderItem {
  id: ID!
  product: Product!
  variant: ProductVariant
  quantity: Int!
  unitPrice: Decimal!
  totalPrice: Decimal!
}

type Customer {
  id: ID!
  email: EmailAddress!
  firstName: String!
  lastName: String!
  fullName: String!
  phone: String
  addresses: [Address!]!
  orders: [Order!]!
  wishlist: [Product!]!
  createdAt: DateTime!
}

type Address {
  id: ID!
  name: String!
  line1: String!
  line2: String
  city: String!
  state: String!
  zipCode: String!
  country: String!
  isDefault: Boolean!
}

type Cart {
  id: ID!
  customer: Customer
  items: [CartItem!]!
  subtotal: Decimal!
  itemCount: Int!
  couponCode: String
  discount: Decimal!
  total: Decimal!
  updatedAt: DateTime!
}

type CartItem {
  id: ID!
  product: Product!
  variant: ProductVariant
  quantity: Int!
  price: Decimal!
}

type Review {
  id: ID!
  customer: Customer!
  product: Product!
  rating: Int!
  title: String!
  body: String!
  isVerifiedPurchase: Boolean!
  helpfulCount: Int!
  createdAt: DateTime!
}

type ProductReviews {
  items: [Review!]!
  averageRating: Float!
  totalCount: Int!
  ratingDistribution: RatingDistribution!
}

type RatingDistribution {
  one: Int!
  two: Int!
  three: Int!
  four: Int!
  five: Int!
}

# ========================
# Enums
# ========================

enum OrderStatus {
  PENDING
  CONFIRMED
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
  REFUNDED
}

enum PaymentMethod {
  CREDIT_CARD
  DEBIT_CARD
  PROMPTPAY
  BANK_TRANSFER
  COD
}

enum Currency {
  THB
  USD
  EUR
}

enum ProductSortField {
  NAME
  PRICE_LOW_TO_HIGH
  PRICE_HIGH_TO_LOW
  NEWEST
  BEST_SELLING
  RATING
}

# ========================
# Connections
# ========================

type ProductConnection {
  products: [Product!]!
  totalCount: Int!
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
}

type OrderConnection {
  orders: [Order!]!
  totalCount: Int!
}

# ========================
# Inputs
# ========================

input ProductFilterInput {
  categoryId: ID
  tags: [String!]
  priceMin: Decimal
  priceMax: Decimal
  inStock: Boolean
  search: String
}

input AddToCartInput {
  productId: ID!
  variantId: ID
  quantity: Int!
}

input PlaceOrderInput {
  cartId: ID!
  shippingAddressId: ID!
  billingAddressId: ID!
  paymentMethod: PaymentMethod!
  couponCode: String
}

input CreateReviewInput {
  productId: ID!
  rating: Int!
  title: String!
  body: String!
}

# ========================
# Queries & Mutations
# ========================

type Query {
  # Products
  products(
    filter: ProductFilterInput
    sort: ProductSortField = NEWEST
    page: Int = 1
    perPage: Int = 20
  ): ProductConnection!
  
  product(id: ID, slug: String): Product
  
  # Categories
  categories(parentId: ID): [Category!]!
  category(id: ID, slug: String): Category
  
  # Cart
  cart(id: ID!): Cart
  myCart: Cart
  
  # Orders
  myOrders: [Order!]!
  order(id: ID!): Order
  
  # Search
  search(term: String!, type: SearchType): [SearchResult!]!
}

type Mutation {
  # Cart
  addToCart(input: AddToCartInput!): Cart!
  updateCartItem(itemId: ID!, quantity: Int!): Cart!
  removeFromCart(itemId: ID!): Cart!
  applyCoupon(cartId: ID!, code: String!): Cart!
  
  # Orders
  placeOrder(input: PlaceOrderInput!): Order!
  cancelOrder(orderId: ID!): Order!
  
  # Reviews
  createReview(input: CreateReviewInput!): Review!
  markReviewHelpful(reviewId: ID!): Review!
  
  # Wishlist
  addToWishlist(productId: ID!): Customer!
  removeFromWishlist(productId: ID!): Customer!
}

enum SearchType {
  ALL
  PRODUCTS
  CATEGORIES
}

union SearchResult = Product | Category
```

---

## 📌 Step 78: Schema Best Practices

### 1. Naming Conventions

```graphql
# ✅ ดี — PascalCase สำหรับ types
type UserProfile { ... }
type ProductCategory { ... }

# ❌ ไม่ดี
type user_profile { ... }
type productCategory { ... }

# ✅ ดี — camelCase สำหรับ fields
type User {
  firstName: String!
  lastName: String!
  createdAt: DateTime!
}

# ❌ ไม่ดี
type User {
  first_name: String!
  FirstName: String!
}

# ✅ ดี — SCREAMING_SNAKE_CASE สำหรับ enums
enum UserRole {
  SUPER_ADMIN
  REGULAR_USER
}
```

### 2. Null Safety

```graphql
# ✅ ดี — ชัดเจนว่า field ไหน nullable
type User {
  id: ID!             # ต้องมีเสมอ
  name: String!       # ต้องมีเสมอ
  nickname: String    # optional
  bio: String         # optional
  posts: [Post!]!     # ต้องมี array (อาจ empty ได้)
}

# ❌ หลีกเลี่ยง — non-null ทุก field
# (ทำให้ตอน implement ยาก เพราะต้องมีค่าทุก field เสมอ)
type User {
  id: ID!
  nickname: String!  # ← ถ้าไม่มี nickname ต้องส่ง error
}
```

### 3. Descriptions ทุกที่

```graphql
"""
User type — แทน user ในระบบ

ทุก user มี unique email และ id
"""
type User {
  """Unique identifier"""
  id: ID!
  
  """Display name ที่แสดงใน UI"""
  name: String!
  
  """Email address (case-insensitive unique)"""
  email: String!
}
```

### 4. Input Validation

```graphql
# แทนที่จะรับ raw String สำหรับ email
input BadInput {
  email: String!  # ← ไม่มี validation
}

# ใช้ Custom Scalar แทน
input GoodInput {
  email: EmailAddress!  # ← validate format ใน scalar
  website: URL          # ← validate URL format
  birthDate: Date       # ← validate date format
}
```

---

## 📌 Step 79: Schema Documentation Pattern

```javascript
// ใช้ graphql-markdown หรือ spectaql ในการ generate docs
// ติดตั้ง
npm install -D spectaql

// spectaql.config.yml
spectaql:
  display:
    showPlayground: true

introspection:
  url: 'http://localhost:4000/graphql'

info:
  title: 'GraphQL Course API'
  description: 'API สำหรับ GraphQL Course'
  version: '1.0.0'
  contact:
    name: 'Support'
    url: 'https://example.com/support'
    email: 'support@example.com'
```

---

## 📌 Step 80: Schema Linting

```bash
# ติดตั้ง graphql-inspector
npm install -D @graphql-inspector/cli

# Validate schema
npx graphql-inspector validate schema.graphql

# Introspect และบันทึก schema
npx graphql-inspector introspect http://localhost:4000/graphql --write schema.graphql

# Check for breaking changes
npx graphql-inspector diff schema.graphql new-schema.graphql
```

```javascript
// .graphql-inspector.yml
schema: 'src/schema/**/*.graphql'
documents: 'src/**/*.ts'

# ตรวจสอบ operations
validate:
  documents: 'src/**/*.ts'
```

---

## 📌 Step 81: สรุป Part 004

### SDL Cheatsheet

```graphql
# Scalars
String, Int, Float, Boolean, ID

# Non-null
field: String!

# List
field: [String]
field: [String!]
field: [String]!
field: [String!]!

# Object Type
type Foo { bar: String! }

# Input Type
input FooInput { bar: String! }

# Enum
enum Status { ACTIVE INACTIVE }

# Interface
interface Node { id: ID! }
type Foo implements Node { id: ID! }

# Union
union Result = Foo | Bar

# Custom Scalar
scalar Date

# Directive
directive @auth(role: Role) on FIELD_DEFINITION
```

### เนื้อหาที่เรียนรู้

✅ Scalar types ทั้ง 5 ชนิด  
✅ Non-null `!` และ List `[]`  
✅ Object, Input, Enum, Interface, Union types  
✅ Fields with arguments  
✅ Custom scalars  
✅ Built-in directives  
✅ E-commerce schema จริง  
✅ Best practices  

### Homework

1. สร้าง Schema สำหรับ Todo App ที่มี Users, Projects, Tasks
2. เพิ่ม Custom Scalar `PhoneNumber` ที่ validate เบอร์โทรไทย
3. ออกแบบ Schema สำหรับ Social Media ที่มี Posts, Likes, Comments, Followers

### ในส่วนถัดไป

➡️ **[Part 005](./part-005.md)** — GraphQL Queries: ดึงข้อมูลทุกรูปแบบ

---

*Part 004 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
