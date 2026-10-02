# Part 061 — Schema Design Best Practices 📐

> **ระดับ:** Expert | **เวลาเรียน:** 75 นาที | **ขั้นตอนที่:** 1351–1385

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Naming conventions (types, fields, operations)
- Schema evolution & backward compatibility
- Deprecation strategies
- Nullability design
- Input type design
- Pagination standards
- Error handling design
- Schema documentation
- GraphQL Style Guide

---

## 📌 Step 1351: Naming Conventions

```graphql
# ✅ GOOD naming conventions

# Types: PascalCase
type ProductVariant { ... }
type OrderLineItem { ... }
type UserProfile { ... }

# Fields: camelCase
type Product {
  id: ID!
  productName: String!     # ❌ Too redundant
  name: String!            # ✅
  createdAt: DateTime!
  isAvailable: Boolean!    # ✅ Boolean: is/has/can prefix
  hasVariants: Boolean!
  canBeReviewed: Boolean!
  
  # ❌ avoid abbreviations
  mfgDate: DateTime        # Bad: unclear
  manufacturingDate: DateTime! # Good: clear
  
  # ❌ avoid type suffix in field names
  priceFloat: Float        # Bad
  price: Float!            # Good
}

# Mutations: verb + noun
type Mutation {
  # ✅ Verb first, specific action
  createProduct(input: CreateProductInput!): CreateProductResult!
  updateProduct(id: ID!, input: UpdateProductInput!): UpdateProductResult!
  deleteProduct(id: ID!): DeleteProductResult!
  
  # Domain-specific verbs
  publishPost(id: ID!): Post!          # ✅ not updatePostStatus
  archivePost(id: ID!): Post!          # ✅
  likePost(id: ID!): Post!             # ✅
  
  # ❌ Avoid CRUD-only naming in complex domains
  processOrder(id: ID!): Order!        # ambiguous
  fulfillOrder(id: ID!): Order!        # ✅ clear intent
  cancelOrder(id: ID!, reason: String!): Order! # ✅
}

# Enums: SCREAMING_SNAKE_CASE
enum OrderStatus {
  PENDING
  PAYMENT_CONFIRMED
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
  REFUNDED
}

# Input types: suffix with "Input"
input CreateProductInput { ... }
input UpdateProductInput { ... }
input ProductFilterInput { ... }

# Result/payload types
type CreateProductResult {
  product: Product
  errors: [UserError!]!
}

type UserError {
  field: String
  message: String!
  code: String!
}
```

---

## 📌 Step 1352: Nullability Design

```graphql
# Nullability เป็นส่วนหนึ่งของ schema contract

# ✅ ใช้ Non-null (!) เมื่อ:
# - Field มีค่าเสมอ ไม่มีทางเป็น null
# - Operation มี return value เสมอ (หรือ throw error)

type User {
  id: ID!           # ✅ มีเสมอ
  email: String!    # ✅ required field
  name: String!     # ✅ required
  bio: String       # ✅ nullable: optional field
  avatar: String    # ✅ nullable: might not have avatar
}

# ❌ อย่า non-null ทุกอย่างโดยไม่คิด
# ถ้า field อาจเป็น null ในอนาคต → nullable ตั้งแต่ต้น
# เพราะ nullable → non-null = breaking change

# ❌ อย่า nullable input fields โดยไม่ตั้งใจ
input CreateUserInput {
  name: String!     # ✅ required
  email: String!    # ✅ required
  bio: String       # ✅ optional
  role: UserRole    # อาจต้องการ! ถ้าจำเป็น
}

# Lists: ระวัง list nullability
type ProductConnection {
  edges: [ProductEdge!]!   # ✅ list ไม่เป็น null, item ไม่เป็น null
  # vs
  edges: [ProductEdge]     # ❌ list อาจเป็น null, item อาจเป็น null
  # vs
  edges: [ProductEdge!]    # list อาจเป็น null, item ไม่เป็น null
}

# Pattern: queries ควร nullable (not found = null)
# Mutations ควร return result type (ไม่ใช่ null)
type Query {
  product(id: ID!): Product  # ✅ null ถ้าไม่พบ
  me: User                   # ✅ null ถ้าไม่ได้ login
}

type Mutation {
  createProduct(input: CreateProductInput!): CreateProductResult!  # ✅ always result
}
```

---

## 📌 Step 1353: Schema Evolution

```graphql
# Backward-compatible changes ✅:
# - เพิ่ม optional field
# - เพิ่ม type ใหม่
# - เพิ่ม enum value ใหม่
# - เพิ่ม optional input field
# - ทำ required field เป็น nullable
# - เพิ่ม interface implementation

# Breaking changes ❌:
# - ลบ field/type/enum value
# - เปลี่ยนชื่อ field/type
# - เปลี่ยน type (String → Int)
# - ทำ nullable field เป็น non-null
# - เพิ่ม required input field

# Deprecation workflow:
type Product {
  # เก่า (deprecated)
  price: Float @deprecated(reason: "Use `pricing.basePrice` instead")
  
  # ใหม่
  pricing: ProductPricing!
}

type ProductPricing {
  basePrice: Float!
  salePrice: Float
  currency: String!
}

# Sunset policy:
# 1. Mark deprecated (พร้อม migration guide)
# 2. Announce ใน changelog
# 3. Monitor usage (schema analytics)
# 4. Remove after grace period (3-6 เดือน)

# Version annotation (optional)
type User {
  """
  @deprecated(since: "2024-01", removal: "2025-01")
  Use `profile.displayName` instead
  """
  username: String @deprecated(reason: "Use profile.displayName")
  
  profile: UserProfile!
}
```

---

## 📌 Step 1354: Input Design Patterns

```graphql
# Pattern 1: Flat vs Nested input
# ใช้ flat สำหรับ simple updates
input UpdateUserInput {
  name: String
  bio: String
  avatar: String
}

# ใช้ nested สำหรับ complex objects
input CreateOrderInput {
  items: [OrderItemInput!]!
  shipping: ShippingInput!
  payment: PaymentInput!
  promoCode: String
}

# Pattern 2: Patch vs Replace
# Patch: update เฉพาะ fields ที่ส่งมา (null = no change, undefined = skip)
input PatchUserInput {
  name: String       # null = set to null, undefined = skip
  bio: String
}

# Replace: set ทุก field (null = set to null)
input ReplaceUserInput {
  name: String!      # required
  bio: String        # null allowed
}

# Pattern 3: Discriminated input union (workaround)
# GraphQL ไม่มี input union โดยตรง
# ใช้ optional + validation

input NotificationInput {
  # ต้องระบุเพียงหนึ่งในนี้
  email: EmailNotificationInput
  sms: SMSNotificationInput
  push: PushNotificationInput
}

# Validate ใน resolver:
# if ([email, sms, push].filter(Boolean).length !== 1) throw error

# Pattern 4: oneOf directive (GraphQL spec draft)
input NotificationInput @oneOf {
  email: EmailNotificationInput
  sms: SMSNotificationInput
  push: PushNotificationInput
}
```

---

## 📌 Step 1355: Schema Documentation

```graphql
"""
Product ใน catalog — สิ่งที่ขายในร้าน
ประกอบด้วยข้อมูลพื้นฐาน, pricing, และ inventory
"""
type Product {
  """
  Unique identifier ของ product
  Format: UUID v4
  """
  id: ID!
  
  """
  ชื่อ product ที่แสดงต่อลูกค้า
  Max length: 255 characters
  """
  name: String!
  
  """
  ราคาขายหน้าร้าน (ก่อน discount)
  Unit: บาท (THB)
  """
  price: Float!
  
  """
  จำนวนสินค้าที่มีในคลัง
  Negative values หมายถึง backorder
  """
  stock: Int!
  
  """
  รูปภาพ product ทั้งหมด เรียงจาก primary → secondary
  รูปแรกใช้เป็น thumbnail
  """
  images: [ProductImage!]!
  
  """
  @deprecated(since: "2024-06")
  ใช้ `stock > 0` แทน
  """
  inStock: Boolean @deprecated(reason: "Use `stock > 0`")
}

"""
Input สำหรับสร้าง Product ใหม่
ต้องการ permission: ADMIN หรือ SELLER
"""
input CreateProductInput {
  """ชื่อ product (required, max 255 chars)"""
  name: String!
  
  """ราคาขาย เป็น THB, ต้อง > 0"""
  price: Float!
  
  """จำนวน stock เริ่มต้น, ต้อง >= 0"""
  initialStock: Int!
  
  """Category ID ที่ product อยู่ใน"""
  categoryId: ID!
}
```

---

## 📌 Step 1356: สรุป Part 061

### เนื้อหาที่เรียนรู้

✅ Naming conventions  
✅ Nullability design  
✅ Schema evolution & deprecation  
✅ Input type patterns  
✅ Schema documentation  

### Quick Reference

```
Types:         PascalCase (UserProfile, OrderItem)
Fields:        camelCase (createdAt, isActive)
Enums:         SCREAMING_SNAKE_CASE (ORDER_PENDING)
Inputs:        XxxInput (CreateUserInput)
Results:       XxxResult, XxxPayload
Mutations:     verb+noun (createUser, cancelOrder)

Non-null (!):  สำหรับ fields ที่มีเสมอ
Nullable:      optional fields, "not found" cases
List:          [Type!]! สำหรับ collections ปกติ

Deprecation:   @deprecated(reason: "...") + sunset plan
Evolution:     เพิ่มได้เสมอ, ลบ = breaking change
```

### ในส่วนถัดไป

➡️ **[Part 062](./part-062.md)** — Performance Optimization: N+1 และ Caching Strategies

---

*Part 061 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
