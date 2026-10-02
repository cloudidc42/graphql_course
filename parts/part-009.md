# Part 009 — Types & Fields: ทุกรายละเอียด 🔬

> **ระดับ:** Beginner-Intermediate | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 211–250

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Scalar types ทั้งหมด + Custom scalars
- Object types และ fields
- Non-null และ List modifiers
- Interface types
- Union types
- Enum types
- Input types
- Abstract types resolution
- Type system best practices

---

## 📌 Step 211: Scalar Types ทั้งหมด

GraphQL มี built-in scalars 5 ชนิด:

```graphql
type ScalarExamples {
  # String — UTF-8 text
  name: String
  description: String
  
  # Int — 32-bit signed integer (-2^31 ถึง 2^31 - 1)
  age: Int
  count: Int
  
  # Float — Double-precision floating point
  price: Float
  latitude: Float
  
  # Boolean — true หรือ false
  isActive: Boolean
  isVerified: Boolean
  
  # ID — Unique identifier (String ใน GraphQL แต่อาจเป็น Int ใน backend)
  id: ID
  userId: ID
}
```

### ตัวอย่าง Scalar Values ใน JSON

```json
{
  "name": "สมชาย ใจดี",
  "age": 30,
  "price": 299.99,
  "isActive": true,
  "id": "user_abc123"
}
```

---

## 📌 Step 212: Non-Null Modifier (!)

`!` หมายถึง field นี้ **จะไม่เป็น null เด็ดขาด** — GraphQL จะ error ถ้า resolver return null

```graphql
type User {
  id: ID!         # ต้องมีค่าเสมอ
  name: String!   # ต้องมีค่าเสมอ
  email: String!  # ต้องมีค่าเสมอ
  
  bio: String     # อาจเป็น null ได้ (optional)
  avatar: String  # อาจเป็น null ได้
}
```

### Non-Null บน Arguments

```graphql
type Query {
  # id ต้องส่งมาเสมอ
  user(id: ID!): User
  
  # searchTerm ต้องส่งมาเสมอ, limit เป็น optional
  searchUsers(searchTerm: String!, limit: Int): [User!]!
}
```

### Resolver กับ Non-Null

```javascript
const resolvers = {
  Query: {
    user: async (_, { id }) => {
      const user = await User.findById(id);
      
      // ถ้า return type เป็น User! (non-null)
      // แต่ user ไม่มีอยู่ → GraphQL จะ throw error
      // ดังนั้น throw error เองดีกว่า
      if (!user) {
        throw new GraphQLError(`User ${id} not found`, {
          extensions: { code: 'NOT_FOUND' }
        });
      }
      
      return user;
    },
  },
};
```

---

## 📌 Step 213: List Types

```graphql
type Query {
  # [User] — list อาจเป็น null, items อาจเป็น null
  users1: [User]
  
  # [User!] — list อาจเป็น null, แต่ items ต้องไม่เป็น null
  users2: [User!]
  
  # [User]! — list ต้องไม่เป็น null, แต่ items อาจเป็น null
  users3: [User]!
  
  # [User!]! — list ต้องไม่เป็น null, items ต้องไม่เป็น null (ดีที่สุด)
  users4: [User!]!
}
```

### สรุปความแตกต่าง

```
[User]    → null | [null, null] | [user1, null, user2] | [user1, user2]
[User!]   → null | [] | [user1, user2]
[User]!   → [] | [null, null] | [user1, null] | [user1, user2]
[User!]!  → [] | [user1, user2]   ← recommended
```

### ใน Resolver

```javascript
const resolvers = {
  Query: {
    users4: async () => {
      const users = await User.findAll();
      // users4 เป็น [User!]! ดังนั้น:
      // 1. ต้อง return array เสมอ (ไม่ return null)
      // 2. ทุก item ต้องไม่เป็น null
      return users; // [] ก็โอเค
    },
  },
};
```

---

## 📌 Step 214: Object Types

```graphql
type Product {
  id: ID!
  name: String!
  description: String
  price: Float!
  inStock: Boolean!
  
  # Nested objects
  category: Category!
  seller: User!
  images: [ProductImage!]!
  
  # Computed fields (resolved by resolver)
  discountedPrice: Float
  isOnSale: Boolean!
}

type Category {
  id: ID!
  name: String!
  slug: String!
  products: [Product!]!
}

type ProductImage {
  id: ID!
  url: String!
  alt: String
  isPrimary: Boolean!
}
```

### Resolvers สำหรับ Nested Fields

```javascript
const resolvers = {
  Product: {
    // field resolver สำหรับ computed fields
    discountedPrice: (product) => {
      if (!product.discountPercent) return null;
      return product.price * (1 - product.discountPercent / 100);
    },
    
    isOnSale: (product) => {
      return product.discountPercent > 0 && product.saleEndDate > new Date();
    },
    
    // Lazy load nested objects
    category: async (product) => {
      return Category.findById(product.categoryId);
    },
    
    seller: async (product) => {
      return User.findById(product.sellerId);
    },
  },
};
```

---

## 📌 Step 215: Interface Types

Interface คือ "contract" ที่ types ต้อง implement:

```graphql
interface Node {
  id: ID!
}

interface Timestamped {
  createdAt: String!
  updatedAt: String!
}

interface Searchable {
  title: String!
  description: String
  tags: [String!]!
}

# Types ที่ implement interfaces
type Article implements Node & Timestamped & Searchable {
  id: ID!
  title: String!
  description: String
  tags: [String!]!
  createdAt: String!
  updatedAt: String!
  
  # fields เพิ่มเติม
  content: String!
  author: User!
  publishedAt: String
}

type Video implements Node & Timestamped & Searchable {
  id: ID!
  title: String!
  description: String
  tags: [String!]!
  createdAt: String!
  updatedAt: String!
  
  # fields เพิ่มเติม
  videoUrl: String!
  duration: Int!
  thumbnail: String
}

type Query {
  # Return interface type
  search(term: String!): [Searchable!]!
  node(id: ID!): Node
}
```

### __resolveType สำหรับ Interface

```javascript
const resolvers = {
  Searchable: {
    __resolveType(obj) {
      // GraphQL ต้องรู้ว่า obj เป็น type อะไร
      if (obj.content) return 'Article';
      if (obj.videoUrl) return 'Video';
      return null;
    },
  },
  
  Node: {
    __resolveType(obj) {
      // ใช้ __typename ถ้า backend เก็บไว้
      if (obj.__typename) return obj.__typename;
      
      // หรือ detect จาก fields
      if (obj.content !== undefined) return 'Article';
      if (obj.videoUrl !== undefined) return 'Video';
      return null;
    },
  },
  
  Query: {
    search: async (_, { term }) => {
      const articles = await Article.search(term);
      const videos = await Video.search(term);
      
      // เพิ่ม __typename เพื่อให้ __resolveType ง่ายขึ้น
      return [
        ...articles.map(a => ({ ...a, __typename: 'Article' })),
        ...videos.map(v => ({ ...v, __typename: 'Video' })),
      ];
    },
  },
};
```

### Query ด้วย Interface

```graphql
query {
  search(term: "GraphQL") {
    title          # ← field บน interface
    description    # ← field บน interface
    tags           # ← field บน interface
    
    ... on Article {
      content      # ← field เฉพาะ Article
      author { name }
    }
    
    ... on Video {
      videoUrl     # ← field เฉพาะ Video
      duration
    }
  }
}
```

---

## 📌 Step 216: Union Types

Union คือ "อาจเป็น type ใด type หนึ่งจากหลาย types":

```graphql
# Union ไม่มี shared fields (ต่างจาก Interface)
union SearchResult = User | Product | Article | Category

union MediaContent = Image | Video | Audio

type Query {
  search(term: String!): [SearchResult!]!
  getMedia(id: ID!): MediaContent
}
```

### __resolveType สำหรับ Union

```javascript
const resolvers = {
  SearchResult: {
    __resolveType(obj) {
      // ต้อง return type name เป็น string
      switch (obj.type) {
        case 'user': return 'User';
        case 'product': return 'Product';
        case 'article': return 'Article';
        case 'category': return 'Category';
        default:
          // Detect จาก unique fields
          if (obj.email) return 'User';
          if (obj.price) return 'Product';
          if (obj.content) return 'Article';
          if (obj.slug) return 'Category';
          return null;
      }
    },
  },
};
```

### Query ด้วย Union

```graphql
query {
  search(term: "Node.js") {
    __typename  # ← รู้ว่าเป็น type อะไร
    
    ... on User {
      id
      name
      email
    }
    
    ... on Product {
      id
      name
      price
    }
    
    ... on Article {
      id
      title
      publishedAt
    }
  }
}
```

---

## 📌 Step 217: Enum Types

```graphql
enum UserRole {
  ADMIN
  MODERATOR
  USER
  GUEST
}

enum OrderStatus {
  PENDING
  CONFIRMED
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
  REFUNDED
}

enum SortDirection {
  ASC
  DESC
}

type User {
  id: ID!
  name: String!
  role: UserRole!
}

type Order {
  id: ID!
  status: OrderStatus!
  items: [OrderItem!]!
}

type Query {
  users(role: UserRole, sort: SortDirection): [User!]!
  orders(status: OrderStatus): [Order!]!
}
```

### Enum ใน Resolver

```javascript
const resolvers = {
  Query: {
    users: async (_, { role, sort = 'ASC' }) => {
      const where = role ? { role } : {};
      const order = [['name', sort]];
      
      return User.findAll({ where, order });
    },
    
    orders: async (_, { status }) => {
      const where = status ? { status } : {};
      return Order.findAll({ where });
    },
  },
  
  Mutation: {
    updateOrderStatus: async (_, { orderId, status }) => {
      const order = await Order.findById(orderId);
      
      // Validate transition
      const validTransitions = {
        PENDING: ['CONFIRMED', 'CANCELLED'],
        CONFIRMED: ['PROCESSING', 'CANCELLED'],
        PROCESSING: ['SHIPPED'],
        SHIPPED: ['DELIVERED'],
      };
      
      const allowed = validTransitions[order.status] || [];
      if (!allowed.includes(status)) {
        throw new GraphQLError(
          `Cannot transition from ${order.status} to ${status}`
        );
      }
      
      return order.update({ status });
    },
  },
};
```

---

## 📌 Step 218: Input Types

Input types ใช้สำหรับ arguments ที่ซับซ้อน (mutations):

```graphql
# Input types สำหรับ create
input CreateUserInput {
  name: String!
  email: String!
  password: String!
  role: UserRole
  bio: String
  avatarUrl: String
}

# Input types สำหรับ update (fields อาจเป็น null เพื่อ partial update)
input UpdateUserInput {
  name: String
  email: String
  bio: String
  avatarUrl: String
}

# Input types ซ้อนกัน
input CreateOrderInput {
  customerId: ID!
  items: [OrderItemInput!]!
  shippingAddress: AddressInput!
  paymentMethod: PaymentMethodInput!
}

input OrderItemInput {
  productId: ID!
  quantity: Int!
  variantId: ID
}

input AddressInput {
  street: String!
  city: String!
  state: String
  country: String!
  postalCode: String!
}

input PaymentMethodInput {
  type: PaymentType!
  cardToken: String
  bankAccount: String
}

enum PaymentType {
  CREDIT_CARD
  BANK_TRANSFER
  CASH_ON_DELIVERY
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  createOrder(input: CreateOrderInput!): Order!
}
```

### Input Validation

```javascript
import { z } from 'zod';

const CreateUserSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
  password: z.string().min(8).regex(/^(?=.*[A-Z])(?=.*[0-9])/),
  bio: z.string().max(500).optional(),
});

const resolvers = {
  Mutation: {
    createUser: async (_, { input }) => {
      // Validate input
      const validated = CreateUserSchema.safeParse(input);
      if (!validated.success) {
        throw new GraphQLError('Validation failed', {
          extensions: {
            code: 'VALIDATION_ERROR',
            errors: validated.error.flatten().fieldErrors,
          },
        });
      }
      
      const { password, ...userData } = validated.data;
      const hashedPassword = await bcrypt.hash(password, 12);
      
      return User.create({ ...userData, password: hashedPassword });
    },
  },
};
```

---

## 📌 Step 219: Custom Scalar Types

```javascript
// src/scalars/custom-scalars.js
import { GraphQLScalarType, Kind } from 'graphql';

// DateTime scalar
export const DateTimeScalar = new GraphQLScalarType({
  name: 'DateTime',
  description: 'ISO 8601 date-time string',
  
  // Serialize: แปลง internal value → response JSON
  serialize(value) {
    if (value instanceof Date) {
      return value.toISOString();
    }
    if (typeof value === 'string') {
      return new Date(value).toISOString();
    }
    throw new Error('DateTime must be a Date or ISO string');
  },
  
  // Parse value: แปลง input value → internal representation
  parseValue(value) {
    const date = new Date(value);
    if (isNaN(date.getTime())) {
      throw new Error('Invalid DateTime');
    }
    return date;
  },
  
  // Parse literal: แปลง GraphQL query literal → internal
  parseLiteral(ast) {
    if (ast.kind === Kind.STRING) {
      const date = new Date(ast.value);
      if (isNaN(date.getTime())) {
        throw new Error('Invalid DateTime');
      }
      return date;
    }
    throw new Error('DateTime must be a string literal');
  },
});

// EmailAddress scalar
export const EmailAddressScalar = new GraphQLScalarType({
  name: 'EmailAddress',
  description: 'Valid email address',
  
  serialize(value) {
    return String(value).toLowerCase();
  },
  
  parseValue(value) {
    const email = String(value).toLowerCase();
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      throw new Error('Invalid email address');
    }
    return email;
  },
  
  parseLiteral(ast) {
    if (ast.kind === Kind.STRING) {
      const email = ast.value.toLowerCase();
      if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
        throw new Error('Invalid email address');
      }
      return email;
    }
    throw new Error('EmailAddress must be a string');
  },
});

// JSON scalar (arbitrary JSON)
export const JSONScalar = new GraphQLScalarType({
  name: 'JSON',
  description: 'Arbitrary JSON value',
  
  serialize(value) {
    return value;
  },
  
  parseValue(value) {
    return value;
  },
  
  parseLiteral(ast) {
    switch (ast.kind) {
      case Kind.STRING: return ast.value;
      case Kind.INT: return parseInt(ast.value, 10);
      case Kind.FLOAT: return parseFloat(ast.value);
      case Kind.BOOLEAN: return ast.value;
      case Kind.NULL: return null;
      case Kind.LIST:
        return ast.values.map(v => JSONScalar.parseLiteral(v));
      case Kind.OBJECT:
        return ast.fields.reduce((acc, field) => ({
          ...acc,
          [field.name.value]: JSONScalar.parseLiteral(field.value),
        }), {});
    }
  },
});
```

### ใช้ Custom Scalars ใน Schema

```javascript
// schema.js
const typeDefs = `
  scalar DateTime
  scalar EmailAddress
  scalar JSON
  
  type User {
    id: ID!
    email: EmailAddress!
    name: String!
    createdAt: DateTime!
    updatedAt: DateTime!
    metadata: JSON
  }
`;

const resolvers = {
  DateTime: DateTimeScalar,
  EmailAddress: EmailAddressScalar,
  JSON: JSONScalar,
  
  // ...other resolvers
};
```

---

## 📌 Step 220: ตัวอย่าง — E-commerce Schema

```graphql
scalar DateTime
scalar Decimal
scalar EmailAddress
scalar URL

enum UserRole { ADMIN SELLER BUYER }
enum ProductStatus { DRAFT ACTIVE INACTIVE DELETED }
enum OrderStatus { PENDING CONFIRMED SHIPPED DELIVERED CANCELLED }

interface Node {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type User implements Node {
  id: ID!
  email: EmailAddress!
  name: String!
  role: UserRole!
  avatar: URL
  createdAt: DateTime!
  updatedAt: DateTime!
  
  # Computed
  orders: [Order!]!
  wishlist: [Product!]!
}

type Product implements Node {
  id: ID!
  name: String!
  slug: String!
  description: String
  price: Decimal!
  status: ProductStatus!
  stock: Int!
  images: [ProductImage!]!
  category: Category!
  seller: User!
  createdAt: DateTime!
  updatedAt: DateTime!
  
  # Computed
  inStock: Boolean!
  discountedPrice: Decimal
}

type ProductImage {
  id: ID!
  url: URL!
  alt: String
  order: Int!
}

type Category implements Node {
  id: ID!
  name: String!
  slug: String!
  parent: Category
  children: [Category!]!
  products: [Product!]!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type OrderItem {
  id: ID!
  product: Product!
  quantity: Int!
  price: Decimal!
  total: Decimal!
}

type Order implements Node {
  id: ID!
  customer: User!
  items: [OrderItem!]!
  status: OrderStatus!
  subtotal: Decimal!
  shipping: Decimal!
  tax: Decimal!
  total: Decimal!
  shippingAddress: Address!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type Address {
  street: String!
  city: String!
  state: String
  country: String!
  postalCode: String!
}

union SearchResult = Product | Category | User
```

---

## 📌 Step 221: สรุป Type System

```
Built-in Scalars:
  String, Int, Float, Boolean, ID

Modifiers:
  T    → nullable T
  T!   → non-null T
  [T]  → nullable list of nullable T
  [T!] → nullable list of non-null T
  [T]! → non-null list of nullable T
  [T!]!→ non-null list of non-null T

Type Kinds:
  Object Type  → type User { ... }
  Interface    → interface Node { ... }
  Union        → union Result = A | B | C
  Enum         → enum Role { ADMIN USER }
  Input        → input CreateInput { ... }
  Scalar       → scalar DateTime
```

---

## 📌 Step 222: สรุป Part 009

### เนื้อหาที่เรียนรู้

✅ Scalar types ทั้ง 5 ชนิด  
✅ Non-null modifier (!)  
✅ List types และ combinations  
✅ Object types และ nested resolvers  
✅ Interface types + __resolveType  
✅ Union types + __resolveType  
✅ Enum types  
✅ Input types + validation  
✅ Custom scalar types  
✅ E-commerce schema ตัวอย่าง  

### Homework

1. สร้าง Blog API schema ด้วย types ทุกชนิด
2. implement __resolveType สำหรับ Interface และ Union
3. สร้าง custom scalar สำหรับ Thai phone number

### ในส่วนถัดไป

➡️ **[Part 010](./part-010.md)** — Arguments, Variables & Fragments

---

*Part 009 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
