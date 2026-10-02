# Part 006 — Mutations: สร้าง แก้ไข ลบข้อมูล

> **ระดับ:** Beginner | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 121–150

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Mutation syntax และ patterns
- Input types สำหรับ mutations
- Returning data หลัง mutation
- Error handling ใน mutations
- Optimistic updates
- Bulk mutations
- Transaction patterns

---

## 📌 Step 121: Mutation คืออะไร

**Mutation** ใช้สำหรับ **เปลี่ยนแปลงข้อมูล**:
- **Create** — สร้างข้อมูลใหม่
- **Update** — แก้ไขข้อมูลที่มีอยู่
- **Delete** — ลบข้อมูล

```graphql
# Query (Read) — ไม่เปลี่ยนแปลงข้อมูล
query GetUser { user(id: "1") { name } }

# Mutation (Write) — เปลี่ยนแปลงข้อมูล
mutation CreateUser {
  createUser(input: { name: "ใหม่", email: "new@example.com" }) {
    id
    name
  }
}
```

---

## 📌 Step 122: Mutation Syntax

### Basic Mutation

```graphql
mutation {
  createUser(name: "สมชาย", email: "test@example.com") {
    id
    name
    email
    createdAt
  }
}
```

### Named Mutation + Variables

```graphql
mutation CreateUser($input: CreateUserInput!) {
  createUser(input: $input) {
    id
    name
    email
    createdAt
  }
}
```

```json
{
  "input": {
    "name": "สมชาย ใจดี",
    "email": "somchai@example.com",
    "age": 28
  }
}
```

---

## 📌 Step 123: CRUD Schema

```graphql
# Types
type User {
  id: ID!
  name: String!
  email: String!
  age: Int
  isActive: Boolean!
  role: UserRole!
  createdAt: String!
  updatedAt: String!
}

enum UserRole {
  ADMIN
  USER
}

# Input Types
input CreateUserInput {
  name: String!
  email: String!
  age: Int
  role: UserRole = USER
}

input UpdateUserInput {
  name: String
  email: String
  age: Int
  isActive: Boolean
  role: UserRole
}

# Mutations
type Mutation {
  # Create
  createUser(input: CreateUserInput!): User!
  
  # Update
  updateUser(id: ID!, input: UpdateUserInput!): User!
  
  # Delete
  deleteUser(id: ID!): DeleteResult!
  
  # Batch
  deleteUsers(ids: [ID!]!): BatchDeleteResult!
}

type DeleteResult {
  success: Boolean!
  message: String!
}

type BatchDeleteResult {
  deletedCount: Int!
  failedIds: [ID!]!
}
```

---

## 📌 Step 124: Mutation Resolvers

```javascript
// src/resolvers/mutation.js
import { db } from '../data/index.js';
import { GraphQLError } from 'graphql';

let nextUserId = 4;
let nextPostId = 5;
let nextCommentId = 4;

export const mutationResolvers = {
  Mutation: {
    // =====================
    // USER MUTATIONS
    // =====================

    createUser: async (_, { input }) => {
      // Validation
      if (!input.email.includes('@')) {
        throw new GraphQLError('รูปแบบ email ไม่ถูกต้อง', {
          extensions: { code: 'VALIDATION_ERROR' }
        });
      }

      // Check duplicate
      const existing = db.users.find(u => u.email === input.email);
      if (existing) {
        throw new GraphQLError(`Email ${input.email} ถูกใช้งานแล้ว`, {
          extensions: { code: 'DUPLICATE_EMAIL' }
        });
      }

      const user = {
        id: String(nextUserId++),
        name: input.name,
        email: input.email,
        age: input.age || null,
        isActive: true,
        role: input.role || 'USER',
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString(),
      };

      db.users.push(user);
      return user;
    },

    updateUser: async (_, { id, input }) => {
      const index = db.users.findIndex(u => u.id === id);

      if (index === -1) {
        throw new GraphQLError(`ไม่พบ User id: ${id}`, {
          extensions: { code: 'NOT_FOUND' }
        });
      }

      // Check email duplicate (ถ้าเปลี่ยน email)
      if (input.email) {
        const emailUser = db.users.find(u => u.email === input.email && u.id !== id);
        if (emailUser) {
          throw new GraphQLError(`Email ${input.email} ถูกใช้งานแล้ว`, {
            extensions: { code: 'DUPLICATE_EMAIL' }
          });
        }
      }

      // Update
      db.users[index] = {
        ...db.users[index],
        ...Object.fromEntries(
          Object.entries(input).filter(([, v]) => v !== null && v !== undefined)
        ),
        updatedAt: new Date().toISOString(),
      };

      return db.users[index];
    },

    deleteUser: async (_, { id }) => {
      const index = db.users.findIndex(u => u.id === id);

      if (index === -1) {
        return {
          success: false,
          message: `ไม่พบ User id: ${id}`,
        };
      }

      // ลบ posts และ comments ที่เกี่ยวข้องด้วย
      const postsToDelete = db.posts
        .filter(p => p.authorId === id)
        .map(p => p.id);
      
      db.posts = db.posts.filter(p => p.authorId !== id);
      db.comments = db.comments.filter(
        c => c.authorId !== id && !postsToDelete.includes(c.postId)
      );

      db.users.splice(index, 1);
      return {
        success: true,
        message: 'ลบ User สำเร็จ',
      };
    },

    deleteUsers: async (_, { ids }) => {
      const failedIds = [];
      let deletedCount = 0;

      for (const id of ids) {
        const index = db.users.findIndex(u => u.id === id);
        if (index === -1) {
          failedIds.push(id);
        } else {
          db.users.splice(index, 1);
          deletedCount++;
        }
      }

      return { deletedCount, failedIds };
    },

    // =====================
    // POST MUTATIONS
    // =====================

    createPost: async (_, { input }, context) => {
      // ตรวจสอบ auth
      if (!context.currentUser) {
        throw new GraphQLError('กรุณา Login ก่อน', {
          extensions: { code: 'UNAUTHENTICATED' }
        });
      }

      if (input.title.length < 5) {
        throw new GraphQLError('ชื่อโพสต์ต้องมีอย่างน้อย 5 ตัวอักษร', {
          extensions: { code: 'VALIDATION_ERROR' }
        });
      }

      const post = {
        id: String(nextPostId++),
        title: input.title,
        content: input.content,
        authorId: context.currentUser.id,
        tags: input.tags || [],
        status: input.status || 'DRAFT',
        viewCount: 0,
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString(),
      };

      db.posts.push(post);
      return post;
    },

    updatePost: async (_, { id, input }, context) => {
      const post = db.posts.find(p => p.id === id);

      if (!post) {
        throw new GraphQLError(`ไม่พบ Post id: ${id}`, {
          extensions: { code: 'NOT_FOUND' }
        });
      }

      // ตรวจสอบสิทธิ์
      if (context.currentUser?.id !== post.authorId && 
          context.currentUser?.role !== 'ADMIN') {
        throw new GraphQLError('ไม่มีสิทธิ์แก้ไข Post นี้', {
          extensions: { code: 'FORBIDDEN' }
        });
      }

      const index = db.posts.findIndex(p => p.id === id);
      db.posts[index] = {
        ...post,
        ...Object.fromEntries(
          Object.entries(input).filter(([, v]) => v !== null && v !== undefined)
        ),
        updatedAt: new Date().toISOString(),
      };

      return db.posts[index];
    },

    publishPost: async (_, { id }, context) => {
      const post = db.posts.find(p => p.id === id);
      if (!post) throw new GraphQLError(`ไม่พบ Post id: ${id}`);
      
      if (post.status === 'PUBLISHED') {
        throw new GraphQLError('Post นี้ถูก publish แล้ว');
      }

      const index = db.posts.findIndex(p => p.id === id);
      db.posts[index] = {
        ...post,
        status: 'PUBLISHED',
        publishedAt: new Date().toISOString(),
        updatedAt: new Date().toISOString(),
      };

      return db.posts[index];
    },

    deletePost: async (_, { id }, context) => {
      const post = db.posts.find(p => p.id === id);
      if (!post) return { success: false, message: `ไม่พบ Post id: ${id}` };

      db.posts = db.posts.filter(p => p.id !== id);
      db.comments = db.comments.filter(c => c.postId !== id);

      return { success: true, message: 'ลบ Post สำเร็จ' };
    },

    // =====================
    // COMMENT MUTATIONS
    // =====================

    createComment: async (_, { postId, text }, context) => {
      if (!context.currentUser) {
        throw new GraphQLError('กรุณา Login ก่อน', {
          extensions: { code: 'UNAUTHENTICATED' }
        });
      }

      const post = db.posts.find(p => p.id === postId);
      if (!post) {
        throw new GraphQLError(`ไม่พบ Post id: ${postId}`, {
          extensions: { code: 'NOT_FOUND' }
        });
      }

      if (text.trim().length === 0) {
        throw new GraphQLError('Comment ต้องมีเนื้อหา', {
          extensions: { code: 'VALIDATION_ERROR' }
        });
      }

      const comment = {
        id: String(nextCommentId++),
        text: text.trim(),
        authorId: context.currentUser.id,
        postId,
        createdAt: new Date().toISOString(),
      };

      db.comments.push(comment);
      return comment;
    },

    deleteComment: async (_, { id }, context) => {
      const comment = db.comments.find(c => c.id === id);
      if (!comment) return { success: false, message: `ไม่พบ Comment id: ${id}` };

      if (context.currentUser?.id !== comment.authorId &&
          context.currentUser?.role !== 'ADMIN') {
        throw new GraphQLError('ไม่มีสิทธิ์ลบ Comment นี้', {
          extensions: { code: 'FORBIDDEN' }
        });
      }

      db.comments = db.comments.filter(c => c.id !== id);
      return { success: true, message: 'ลบ Comment สำเร็จ' };
    },
  },
};
```

---

## 📌 Step 125: Mutation Return Types

### Pattern 1: Return ตัว Object ที่เปลี่ยนแปลง

```graphql
# ✅ ดี — return object หลัง mutation
type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
}

# Usage
mutation {
  createUser(input: { name: "ใหม่", email: "new@example.com" }) {
    id
    name
    email
    createdAt  # ← สำคัญ! รู้ค่าที่ server generate
  }
}
```

### Pattern 2: Payload Type (แนะนำมาก)

```graphql
# ดีที่สุดสำหรับ mutations ซับซ้อน
type CreateUserPayload {
  user: User
  errors: [UserError!]!
  success: Boolean!
}

type UserError {
  field: String
  message: String!
  code: String!
}

type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
}
```

```graphql
# Client รับ payload
mutation CreateUser($input: CreateUserInput!) {
  createUser(input: $input) {
    user {
      id
      name
    }
    errors {
      field
      message
    }
    success
  }
}
```

```javascript
// Resolver
createUser: async (_, { input }) => {
  try {
    // Validate
    if (!input.email.includes('@')) {
      return {
        user: null,
        errors: [{ field: 'email', message: 'Email ไม่ถูกต้อง', code: 'INVALID_EMAIL' }],
        success: false,
      };
    }

    const user = await createUserInDB(input);
    return { user, errors: [], success: true };
  } catch (error) {
    return {
      user: null,
      errors: [{ field: null, message: error.message, code: 'INTERNAL_ERROR' }],
      success: false,
    };
  }
},
```

### Pattern 3: Union Result Type

```graphql
union CreateUserResult = User | ValidationError | DuplicateEmailError

type ValidationError {
  message: String!
  field: String!
}

type DuplicateEmailError {
  message: String!
  email: String!
}

type Mutation {
  createUser(input: CreateUserInput!): CreateUserResult!
}
```

```graphql
# Client
mutation CreateUser($input: CreateUserInput!) {
  createUser(input: $input) {
    ... on User {
      id
      name
    }
    ... on ValidationError {
      message
      field
    }
    ... on DuplicateEmailError {
      message
      email
    }
  }
}
```

---

## 📌 Step 126: Multiple Mutations ใน 1 Request

```graphql
# หลาย mutations ทำงาน sequential (ไม่ใช่ parallel)
mutation BatchOperations {
  firstUser: createUser(input: {
    name: "สมชาย"
    email: "somchai@example.com"
  }) {
    id
    name
  }
  
  secondUser: createUser(input: {
    name: "สมหญิง"
    email: "somying@example.com"
  }) {
    id
    name
  }
  
  adminUser: createUser(input: {
    name: "ผู้ดูแล"
    email: "admin@example.com"
    role: ADMIN
  }) {
    id
    name
    role
  }
}
```

> **หมายเหตุ:** Mutations ทำงาน **sequentially** (ทีละอัน) ไม่เหมือน Queries ที่ทำงาน parallel

---

## 📌 Step 127: Upsert Pattern

```graphql
type Mutation {
  # สร้างหรือแก้ไข user ขึ้นอยู่กับ email ที่มีอยู่
  upsertUser(input: UpsertUserInput!): User!
}

input UpsertUserInput {
  email: String!   # ← identifier
  name: String!
  age: Int
  role: UserRole
}
```

```javascript
upsertUser: async (_, { input }) => {
  const existing = db.users.find(u => u.email === input.email);
  
  if (existing) {
    // Update
    const index = db.users.indexOf(existing);
    db.users[index] = { ...existing, ...input, updatedAt: new Date().toISOString() };
    return db.users[index];
  } else {
    // Create
    const user = {
      id: String(nextUserId++),
      isActive: true,
      role: 'USER',
      ...input,
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    };
    db.users.push(user);
    return user;
  }
},
```

---

## 📌 Step 128: Optimistic Updates (Client Side)

```javascript
// Apollo Client: Optimistic Update
import { useMutation, gql } from '@apollo/client';

const LIKE_POST = gql`
  mutation LikePost($postId: ID!) {
    likePost(postId: $postId) {
      id
      likeCount
      isLikedByMe
    }
  }
`;

function LikeButton({ post }) {
  const [likePost] = useMutation(LIKE_POST, {
    // Optimistic response — ทำให้ UI update ทันทีก่อน server ตอบ
    optimisticResponse: {
      likePost: {
        __typename: 'Post',
        id: post.id,
        likeCount: post.likeCount + 1,
        isLikedByMe: true,
      },
    },
    // ถ้า server ตอบมา update cache ด้วยค่าจริง
    update: (cache, { data }) => {
      cache.modify({
        id: cache.identify(post),
        fields: {
          likeCount: () => data.likePost.likeCount,
          isLikedByMe: () => data.likePost.isLikedByMe,
        },
      });
    },
  });

  return (
    <button onClick={() => likePost({ variables: { postId: post.id } })}>
      ❤️ {post.likeCount}
    </button>
  );
}
```

---

## 📌 Step 129: File Upload Mutation

```graphql
# GraphQL Multipart Request spec
scalar Upload

type Mutation {
  uploadAvatar(userId: ID!, file: Upload!): User!
  uploadPostImages(postId: ID!, files: [Upload!]!): Post!
}
```

```javascript
// Apollo Server 4 ด้วย graphql-upload
import { graphqlUploadExpress } from 'graphql-upload-minimal';

// Middleware
app.use(graphqlUploadExpress({ maxFileSize: 10000000, maxFiles: 10 }));

// Resolver
const resolvers = {
  Mutation: {
    uploadAvatar: async (_, { userId, file }) => {
      const { createReadStream, filename, mimetype } = await file;
      
      // Validate file type
      if (!mimetype.startsWith('image/')) {
        throw new GraphQLError('ต้องเป็นไฟล์รูปภาพเท่านั้น');
      }
      
      // Save file
      const stream = createReadStream();
      const chunks = [];
      for await (const chunk of stream) {
        chunks.push(chunk);
      }
      const buffer = Buffer.concat(chunks);
      
      // Upload to storage (S3, CloudFlare R2, etc.)
      const url = await uploadToStorage(buffer, filename);
      
      // Update user
      const user = db.users.find(u => u.id === userId);
      user.avatarUrl = url;
      return user;
    },
  },
};
```

---

## 📌 Step 130: Transaction-like Mutations

```javascript
// จำลอง transaction ด้วย manual rollback
const mutationResolvers = {
  Mutation: {
    transferPoints: async (_, { fromUserId, toUserId, points }, context) => {
      const fromUser = db.users.find(u => u.id === fromUserId);
      const toUser = db.users.find(u => u.id === toUserId);
      
      if (!fromUser || !toUser) {
        throw new GraphQLError('ไม่พบ User');
      }
      
      if (fromUser.points < points) {
        throw new GraphQLError('คะแนนไม่เพียงพอ');
      }
      
      // บันทึกค่าเดิมสำหรับ rollback
      const originalFromPoints = fromUser.points;
      const originalToPoints = toUser.points;
      
      try {
        // Deduct
        fromUser.points -= points;
        
        // Add
        toUser.points += points;
        
        // Log transaction
        db.transactions.push({
          id: String(Date.now()),
          fromUserId,
          toUserId,
          points,
          createdAt: new Date().toISOString(),
        });
        
        return {
          success: true,
          fromUser,
          toUser,
          transaction: db.transactions[db.transactions.length - 1],
        };
      } catch (error) {
        // Rollback
        fromUser.points = originalFromPoints;
        toUser.points = originalToPoints;
        
        throw new GraphQLError(`การโอนคะแนนล้มเหลว: ${error.message}`);
      }
    },
  },
};
```

---

## 📌 Step 131: Input Validation Patterns

```javascript
// src/validators/user.validator.js

// Validate email format
function validateEmail(email) {
  const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!regex.test(email)) {
    return 'รูปแบบ email ไม่ถูกต้อง';
  }
  return null;
}

// Validate Thai phone
function validateThaiPhone(phone) {
  const regex = /^0[0-9]{8,9}$/;
  if (!regex.test(phone)) {
    return 'เบอร์โทรไม่ถูกต้อง (ตัวอย่าง: 0812345678)';
  }
  return null;
}

// Validate password strength
function validatePassword(password) {
  if (password.length < 8) return 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร';
  if (!/[A-Z]/.test(password)) return 'รหัสผ่านต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว';
  if (!/[0-9]/.test(password)) return 'รหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว';
  return null;
}

// Collect all errors
function validateCreateUserInput(input) {
  const errors = {};
  
  const emailError = validateEmail(input.email);
  if (emailError) errors.email = emailError;
  
  const passwordError = validatePassword(input.password);
  if (passwordError) errors.password = passwordError;
  
  if (input.name.length < 2) {
    errors.name = 'ชื่อต้องมีอย่างน้อย 2 ตัวอักษร';
  }
  
  if (input.age && (input.age < 0 || input.age > 150)) {
    errors.age = 'อายุต้องอยู่ระหว่าง 0-150';
  }
  
  return errors;
}

// ใช้งาน
createUser: async (_, { input }) => {
  const errors = validateCreateUserInput(input);
  
  if (Object.keys(errors).length > 0) {
    throw new GraphQLError('Validation failed', {
      extensions: {
        code: 'VALIDATION_ERROR',
        errors, // { email: "...", name: "..." }
      },
    });
  }
  
  // Create user...
},
```

---

## 📌 Step 132: Testing Mutations

```javascript
// test/mutations.test.js
import { ApolloServer } from '@apollo/server';
import { typeDefs } from '../src/schema/index.js';
import { resolvers } from '../src/resolvers/index.js';

describe('User Mutations', () => {
  let server;
  
  beforeAll(async () => {
    server = new ApolloServer({ typeDefs, resolvers });
    await server.start();
  });
  
  afterAll(async () => {
    await server.stop();
  });

  it('should create a new user', async () => {
    const response = await server.executeOperation({
      query: `
        mutation CreateUser($input: CreateUserInput!) {
          createUser(input: $input) {
            id
            name
            email
            role
          }
        }
      `,
      variables: {
        input: {
          name: 'Test User',
          email: 'test@example.com',
          role: 'USER',
        },
      },
    });
    
    expect(response.body.kind).toBe('single');
    expect(response.body.singleResult.errors).toBeUndefined();
    expect(response.body.singleResult.data?.createUser).toMatchObject({
      name: 'Test User',
      email: 'test@example.com',
      role: 'USER',
    });
  });

  it('should fail with duplicate email', async () => {
    // สร้าง user แรก
    await server.executeOperation({
      query: `mutation { createUser(input: { name: "A", email: "dup@test.com" }) { id } }`,
    });
    
    // สร้าง user ที่สอง ด้วย email เดิม
    const response = await server.executeOperation({
      query: `mutation { createUser(input: { name: "B", email: "dup@test.com" }) { id } }`,
    });
    
    expect(response.body.singleResult.errors).toBeDefined();
    expect(response.body.singleResult.errors[0].message).toContain('ถูกใช้งานแล้ว');
  });

  it('should update user', async () => {
    const response = await server.executeOperation({
      query: `
        mutation {
          updateUser(id: "1", input: { name: "Updated Name" }) {
            id
            name
          }
        }
      `,
    });
    
    expect(response.body.singleResult.data?.updateUser.name).toBe('Updated Name');
  });

  it('should delete user', async () => {
    // สร้าง user ก่อน
    const createRes = await server.executeOperation({
      query: `mutation { createUser(input: { name: "ToDelete", email: "del@test.com" }) { id } }`,
    });
    const userId = createRes.body.singleResult.data?.createUser.id;
    
    // ลบ
    const deleteRes = await server.executeOperation({
      query: `mutation { deleteUser(id: "${userId}") { success message } }`,
    });
    
    expect(deleteRes.body.singleResult.data?.deleteUser.success).toBe(true);
  });
});
```

---

## 📌 Step 133: Schema ครบสมบูรณ์สำหรับ Blog

```graphql
type Mutation {
  # Users
  register(input: RegisterInput!): AuthPayload!
  login(email: String!, password: String!): AuthPayload!
  logout: Boolean!
  updateProfile(input: UpdateProfileInput!): User!
  changePassword(oldPassword: String!, newPassword: String!): Boolean!
  deleteAccount: Boolean!
  
  # Posts
  createPost(input: CreatePostInput!): Post!
  updatePost(id: ID!, input: UpdatePostInput!): Post!
  publishPost(id: ID!): Post!
  unpublishPost(id: ID!): Post!
  deletePost(id: ID!): DeleteResult!
  
  # Comments
  createComment(postId: ID!, text: String!): Comment!
  updateComment(id: ID!, text: String!): Comment!
  deleteComment(id: ID!): DeleteResult!
  
  # Reactions
  likePost(postId: ID!): Post!
  unlikePost(postId: ID!): Post!
  
  # Tags
  addTagToPost(postId: ID!, tag: String!): Post!
  removeTagFromPost(postId: ID!, tag: String!): Post!
  
  # Admin
  suspendUser(userId: ID!, reason: String!): User!
  unsuspendUser(userId: ID!): User!
  promoteToAdmin(userId: ID!): User!
}

type AuthPayload {
  token: String!
  user: User!
}

input RegisterInput {
  name: String!
  email: String!
  password: String!
  age: Int
}

input UpdateProfileInput {
  name: String
  bio: String
  website: String
  location: String
}
```

---

## 📌 Step 134: สรุป Part 006

### Mutation Cheatsheet

```graphql
# Basic
mutation { createUser(name: "A", email: "a@b.com") { id } }

# Named + Variables
mutation CreateUser($input: CreateUserInput!) {
  createUser(input: $input) { id name }
}

# Multiple (sequential)
mutation {
  a: createUser(input: {...}) { id }
  b: createUser(input: {...}) { id }
}

# With alias
mutation {
  adminUser: createUser(input: { ..., role: ADMIN }) { id }
}
```

### เนื้อหาที่เรียนรู้

✅ Mutation syntax ทุกรูปแบบ  
✅ Input types  
✅ CRUD resolvers  
✅ 3 patterns สำหรับ return types  
✅ Multiple mutations  
✅ Transaction pattern  
✅ Input validation  
✅ File upload  
✅ Testing mutations  

### Homework

1. เพิ่ม `likePost` และ `unlikePost` mutations
2. สร้าง `changePassword` mutation ที่ตรวจสอบ old password
3. สร้าง `bulkCreateUsers` mutation ที่รับ array of users

### ในส่วนถัดไป

➡️ **[Part 007](./part-007.md)** — Subscriptions: Real-time Data ด้วย WebSocket

---

*Part 006 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
