# Part 029 — Schema Versioning & Evolution 📐

> **ระดับ:** Advanced | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1041–1080

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Schema versioning strategies
- @deprecated ใน practice
- Field migration patterns
- Backward compatibility
- Schema changelog
- Consumer-driven contracts
- Schema registry
- Breaking change management

---

## 📌 Step 1041: Schema Versioning Strategies

```
GraphQL มี 3 วิธีหลักสำหรับ versioning:

1. URL Versioning (ไม่แนะนำ)
   /graphql/v1, /graphql/v2
   → ต้องดูแล 2 schemas พร้อมกัน

2. Schema Evolution (แนะนำ)
   → @deprecated + add new fields
   → Backward compatible changes
   
3. Schema Federation + Versioning
   → แต่ละ subgraph version ตัวเอง
```

---

## 📌 Step 1042: Backward Compatible Changes

```graphql
# ✅ ปลอดภัย — adding new fields
type User {
  id: ID!
  name: String!
  email: String!
  
  # ← เพิ่ม field ใหม่ (non-breaking)
  phoneNumber: String
  preferences: UserPreferences
}

# ✅ ปลอดภัย — adding optional arguments
type Query {
  users(
    limit: Int = 10
    offset: Int = 0
    # ← เพิ่ม argument ใหม่ (non-breaking เพราะมี default)
    filter: UserFilterInput
    sort: UserSortInput
  ): UserConnection!
}

# ✅ ปลอดภัย — adding new types
type UserPreferences {
  theme: String
  language: String
  notifications: NotificationSettings
}

# ✅ ปลอดภัย — adding @deprecated (ไม่ลบ)
type User {
  username: String @deprecated(reason: "Use 'name' instead")
  name: String!
}
```

---

## 📌 Step 1043: Breaking Changes (ต้องระวัง)

```graphql
# ❌ Breaking — removing field
type User {
  id: ID!
  name: String!
  # email: String!  ← ลบออก = breaking!
}

# ❌ Breaking — changing field type
type User {
  # id: ID!           ← เดิม
  id: String!          # ← เปลี่ยนเป็น String = breaking!
}

# ❌ Breaking — making optional field required
type Query {
  # users(limit: Int): [User!]!     ← เดิม (optional)
  users(limit: Int!): [User!]!       # ← required = breaking!
}

# ❌ Breaking — removing enum value
enum UserRole {
  ADMIN
  USER
  # MODERATOR  ← ลบออก = breaking! (clients ที่ส่ง MODERATOR จะ error)
}
```

---

## 📌 Step 1044: Field Migration Pattern

```javascript
// Phase 1: Deprecate old field, add new field
const phase1Schema = `
  type User {
    id: ID!
    
    # เดิม: profileImage เป็น String URL
    profileImage: String @deprecated(reason: "Use 'avatar' field instead")
    
    # ใหม่: avatar เป็น object ที่มีข้อมูลมากกว่า
    avatar: Avatar
  }
  
  type Avatar {
    url: String!
    thumbnailUrl: String!
    width: Int
    height: Int
  }
`;

// Phase 2: Both fields return same data
const resolvers = {
  User: {
    profileImage: (user) => user.avatarUrl, // ← backward compat
    avatar: (user) => ({
      url: user.avatarUrl,
      thumbnailUrl: user.avatarThumbnailUrl || user.avatarUrl,
      width: user.avatarWidth,
      height: user.avatarHeight,
    }),
  },
};

// Phase 3 (หลัง migration period): ลบ profileImage ออก
// → ต้องมี tool บอก clients ที่ยังใช้ deprecated fields
```

### Tracking Deprecated Field Usage

```javascript
// Apollo Studio ทำให้เห็นได้ หรือ implement เอง:
const resolvers = {
  User: {
    profileImage: (user, _, context, info) => {
      // Log deprecated field usage
      logger.warn({
        deprecatedField: 'User.profileImage',
        operation: info.operation?.name?.value || 'anonymous',
        userId: context.user?.sub,
      }, 'Deprecated field accessed');
      
      // Track ใน metrics
      deprecatedFieldCounter.inc({
        type: 'User',
        field: 'profileImage',
      });
      
      return user.avatarUrl;
    },
  },
};
```

---

## 📌 Step 1045: Schema Registry

```javascript
// scripts/publish-schema.js
import { ApolloClient, gql, InMemoryCache } from '@apollo/client';
import fs from 'fs/promises';

async function publishSchema() {
  const schema = await fs.readFile('./src/schema.graphql', 'utf-8');
  
  const client = new ApolloClient({
    uri: process.env.SCHEMA_REGISTRY_URL,
    cache: new InMemoryCache(),
    headers: {
      'x-api-key': process.env.SCHEMA_REGISTRY_KEY,
    },
  });
  
  const { data } = await client.mutate({
    mutation: gql`
      mutation PublishSchema($schema: String!, $version: String!) {
        publishSchema(schema: $schema, version: $version) {
          id
          version
          createdAt
        }
      }
    `,
    variables: {
      schema,
      version: process.env.APP_VERSION,
    },
  });
  
  console.log('Schema published:', data.publishSchema);
}

publishSchema().catch(console.error);
```

---

## 📌 Step 1046: Consumer-Driven Contracts

```javascript
// ทำให้ clients บอก server ว่าต้องการ fields อะไร
// ป้องกัน server เปลี่ยน schema โดยไม่แจ้ง

// client-a/schema-contract.graphql
const clientAContract = `
  query GetUser {
    user(id: "1") {
      id
      name
      email
    }
  }
`;

// server: ตรวจสอบว่า contract ยังใช้ได้
import { validate } from 'graphql';
import { schema } from './src/schema.js';
import { parse } from 'graphql';

async function validateContracts() {
  const contracts = await loadContracts('./contracts/**/*.graphql');
  
  for (const contract of contracts) {
    const errors = validate(schema, parse(contract.query));
    
    if (errors.length > 0) {
      console.error(`Contract violation: ${contract.name}`);
      errors.forEach(e => console.error(` - ${e.message}`));
      process.exit(1);
    }
  }
  
  console.log(`✅ All ${contracts.length} contracts validated`);
}
```

---

## 📌 Step 1047: สรุป Part 029

### เนื้อหาที่เรียนรู้

✅ Schema versioning strategies  
✅ Backward compatible vs breaking changes  
✅ Field migration pattern  
✅ Tracking deprecated field usage  
✅ Schema registry  
✅ Consumer-driven contracts  

### Homework

1. สร้าง migration plan สำหรับ field ที่ต้องเปลี่ยน type
2. implement deprecated field usage tracking
3. สร้าง schema registry อย่างง่าย

### ในส่วนถัดไป

➡️ **[Part 030](./part-030.md)** — Real-world GraphQL API Project

---

*Part 029 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
