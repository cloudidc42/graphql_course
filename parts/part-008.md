# Part 008 — Resolvers: ทำความเข้าใจแบบลึก

> **ระดับ:** Beginner→Intermediate | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 181–210

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Resolver chain และ execution model
- Parent, args, context, info
- Default resolvers
- Async resolvers
- Error handling ใน resolvers
- Abstract type resolvers (`__resolveType`)
- Custom scalar resolvers
- Resolver composition

---

## 📌 Step 181: Resolver คืออะไร

**Resolver** คือ function ที่ถูกเรียกเมื่อ GraphQL ต้องการค่าของ field:

```
Schema:
type User {
  id: ID!
  name: String!
  posts: [Post!]!
}

Resolvers:
User.id    → (parent) => parent.id
User.name  → (parent) => parent.name
User.posts → (parent) => db.posts.filter(p => p.userId === parent.id)
```

---

## 📌 Step 182: 4 Arguments ของ Resolver

```javascript
const resolver = (parent, args, context, info) => { ... }

// parent   — ผลลัพธ์จาก parent resolver
// args     — arguments ที่ client ส่งมา
// context  — shared object (auth, dataSources, etc.)
// info     — execution info (fieldName, schema, etc.)
```

### Parent (Root Value)

```javascript
const resolvers = {
  Query: {
    // Root Query — parent = rootValue (usually null/undefined)
    user: (parent, { id }) => {
      console.log('parent:', parent); // undefined or {}
      return db.users.find(u => u.id === id);
    },
  },
  
  User: {
    // parent = User object จาก Query.user
    posts: (parent, args) => {
      console.log('parent.id:', parent.id); // user id
      return db.posts.filter(p => p.authorId === parent.id);
    },
    
    Post: {
      // parent = Post object จาก User.posts
      author: (parent) => {
        console.log('parent.authorId:', parent.authorId);
        return db.users.find(u => u.id === parent.authorId);
      },
    },
  },
};
```

### Args

```javascript
const resolvers = {
  Query: {
    posts: (_, args) => {
      console.log('args:', args);
      // { tag: "graphql", page: 2, perPage: 10, status: "PUBLISHED" }
      
      let results = db.posts;
      if (args.tag) results = results.filter(p => p.tags.includes(args.tag));
      if (args.status) results = results.filter(p => p.status === args.status);
      
      const start = (args.page - 1) * args.perPage;
      return results.slice(start, start + args.perPage);
    },
  },
};
```

### Context

```javascript
// สร้าง context เมื่อ request เข้ามา
const contextValue = async ({ req }) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  const user = token ? await verifyToken(token) : null;
  
  return {
    currentUser: user,
    dataSources: {
      users: new UserDataSource(),
      posts: new PostDataSource(),
    },
    loaders: {
      user: new DataLoader(async (ids) => {
        const users = await db.getUsersByIds(ids);
        return ids.map(id => users.find(u => u.id === id));
      }),
    },
    // Request info
    req,
    requestId: generateRequestId(),
  };
};

// ใช้ใน resolver
const resolvers = {
  Query: {
    me: (_, __, context) => {
      if (!context.currentUser) {
        throw new GraphQLError('กรุณา Login', {
          extensions: { code: 'UNAUTHENTICATED' }
        });
      }
      return context.dataSources.users.getById(context.currentUser.id);
    },
  },
  
  Post: {
    // ใช้ DataLoader ป้องกัน N+1
    author: (post, _, context) => context.loaders.user.load(post.authorId),
  },
};
```

### Info

```javascript
const resolvers = {
  Query: {
    user: (_, { id }, __, info) => {
      // ชื่อ field ปัจจุบัน
      console.log(info.fieldName); // "user"
      
      // ดู fields ที่ client ขอ
      const requestedFields = getRequestedFields(info);
      console.log(requestedFields); // ['id', 'name', 'email']
      
      // Schema object
      const schema = info.schema;
      
      // Path ถึง field นี้
      console.log(info.path); // { prev: null, key: 'user', typename: 'Query' }
      
      return db.users.find(u => u.id === id);
    },
  },
};

// Helper: ดึง fields ที่ client request
function getRequestedFields(info) {
  return info.fieldNodes[0].selectionSet.selections
    .filter(s => s.kind === 'Field')
    .map(s => s.name.value);
}
```

---

## 📌 Step 183: Default Resolvers

ถ้าไม่มี resolver สำหรับ field — GraphQL ใช้ **default resolver**:

```javascript
// Default resolver ทำแค่นี้:
const defaultFieldResolver = (source, args, contextValue, info) => {
  const field = source[info.fieldName];
  return typeof field === 'function' ? field(args) : field;
};

// ดังนั้น — ถ้า object มี property ตรงกับ field name
// ไม่ต้องเขียน resolver!

const user = {
  id: '1',
  name: 'สมชาย',    // ← User.name resolver ไม่จำเป็น
  email: 'test@x.com', // ← User.email resolver ไม่จำเป็น
};

// แต่ถ้า field name ต่างกับ property name ต้องเขียน:
const userFromDB = {
  user_id: '1',   // snake_case จาก DB
  full_name: 'สมชาย',
};

const resolvers = {
  User: {
    id: (u) => u.user_id,          // ← ต้องเขียน
    name: (u) => u.full_name,      // ← ต้องเขียน
  },
};
```

---

## 📌 Step 184: Async Resolvers

```javascript
const resolvers = {
  Query: {
    // Async resolver
    user: async (_, { id }) => {
      // Database call
      const user = await User.findById(id);
      return user;
    },
    
    // Promise-based
    posts: (_, args) => {
      return Post.findAll({ where: args });  // returns Promise
    },
    
    // Error handling
    safeUser: async (_, { id }) => {
      try {
        const user = await User.findById(id);
        if (!user) return null; // ส่ง null ถ้าไม่มี
        return user;
      } catch (error) {
        console.error('DB Error:', error);
        throw new GraphQLError('เกิดข้อผิดพลาดจาก database');
      }
    },
  },
};
```

---

## 📌 Step 185: Resolver Execution Model

```
Query: {
  user(id: "1") {              ← 1. Query.user resolver
    name                        ← 2. User.name (default)
    posts {                     ← 3. User.posts resolver
      title                     ← 4. Post.title (default) ← parallel
      author {                  ← 5. Post.author resolver ← parallel
        name                    ← 6. User.name (default)
      }
    }
  }
}

Execution order:
1. Query.user(null, {id: "1"}, ctx, info) → User
2. Parallel:
   - User.name(user, {}, ctx, info) → "สมชาย" (default)
   - User.posts(user, {}, ctx, info) → [Post, Post, ...]
3. For each Post (parallel):
   - Post.title(post, {}, ctx, info) (default)
   - Post.author(post, {}, ctx, info) → User
4. User.name(user, {}, ctx, info) (default)
```

> **Key insight:** GraphQL executes **sibling fields in parallel** แต่ children ต้องรอ parent

---

## 📌 Step 186: Abstract Type Resolvers

สำหรับ Interface และ Union — ต้องบอก GraphQL ว่า object เป็น type อะไร:

```javascript
const resolvers = {
  // Union resolver
  SearchResult: {
    __resolveType(obj) {
      // ตรวจสอบ object แล้ว return ชื่อ type
      if (obj.email) return 'User';      // User มี email
      if (obj.title) return 'Post';      // Post มี title
      if (obj.text) return 'Comment';   // Comment มี text
      return null;
    },
  },
  
  // Interface resolver
  Node: {
    __resolveType(obj) {
      if (obj.__type) return obj.__type; // ถ้า object มี __type field
      if (obj.email) return 'User';
      if (obj.title) return 'Post';
      return null;
    },
  },
  
  // Cleaner: เก็บ __typename ใน object
  Query: {
    search: (_, { term }) => {
      const results = [];
      
      db.users.filter(u => u.name.includes(term)).forEach(u => {
        results.push({ ...u, __typename: 'User' });  // ← เพิ่ม __typename
      });
      
      db.posts.filter(p => p.title.includes(term)).forEach(p => {
        results.push({ ...p, __typename: 'Post' });  // ← เพิ่ม __typename
      });
      
      return results;
    },
  },
};
```

---

## 📌 Step 187: Custom Scalar Resolvers

```javascript
import { GraphQLScalarType, Kind } from 'graphql';

// DateTime scalar
const DateTimeScalar = new GraphQLScalarType({
  name: 'DateTime',
  description: 'ISO 8601 DateTime string',

  // JavaScript → GraphQL (serialize ก่อนส่ง client)
  serialize(value) {
    if (value instanceof Date) return value.toISOString();
    if (typeof value === 'string') return new Date(value).toISOString();
    throw new Error('DateTime must be a Date object or ISO string');
  },

  // GraphQL → JavaScript (parse variable)
  parseValue(value) {
    if (typeof value !== 'string') throw new Error('DateTime must be a string');
    const date = new Date(value);
    if (isNaN(date.getTime())) throw new Error(`Invalid DateTime: ${value}`);
    return date;
  },

  // GraphQL → JavaScript (parse literal in query)
  parseLiteral(ast) {
    if (ast.kind !== Kind.STRING) throw new Error('DateTime must be a string');
    const date = new Date(ast.value);
    if (isNaN(date.getTime())) throw new Error(`Invalid DateTime: ${ast.value}`);
    return date;
  },
});

// JSON scalar
const JSONScalar = new GraphQLScalarType({
  name: 'JSON',
  description: 'Arbitrary JSON value',
  serialize: (value) => value,
  parseValue: (value) => value,
  parseLiteral: (ast) => {
    switch (ast.kind) {
      case Kind.STRING: return ast.value;
      case Kind.INT: return parseInt(ast.value, 10);
      case Kind.FLOAT: return parseFloat(ast.value);
      case Kind.BOOLEAN: return ast.value;
      case Kind.NULL: return null;
      case Kind.LIST: return ast.values.map(parseLiteral);
      case Kind.OBJECT: {
        const obj = {};
        ast.fields.forEach(f => { obj[f.name.value] = parseLiteral(f.value); });
        return obj;
      }
    }
  },
});

// Register
const resolvers = {
  DateTime: DateTimeScalar,
  JSON: JSONScalar,
  // ...
};
```

---

## 📌 Step 188: Resolver Composition

```javascript
// Middleware pattern สำหรับ resolvers

// Auth guard
function withAuth(resolver) {
  return (parent, args, context, info) => {
    if (!context.currentUser) {
      throw new GraphQLError('กรุณา Login', {
        extensions: { code: 'UNAUTHENTICATED' }
      });
    }
    return resolver(parent, args, context, info);
  };
}

// Role guard
function withRole(role) {
  return (resolver) => (parent, args, context, info) => {
    if (context.currentUser?.role !== role) {
      throw new GraphQLError('ไม่มีสิทธิ์', {
        extensions: { code: 'FORBIDDEN' }
      });
    }
    return resolver(parent, args, context, info);
  };
}

// Rate limit
function withRateLimit(key, max, windowMs) {
  const counts = new Map();
  
  return (resolver) => async (parent, args, context, info) => {
    const clientKey = context.currentUser?.id || context.req?.ip;
    const now = Date.now();
    
    const entry = counts.get(clientKey) || { count: 0, resetAt: now + windowMs };
    if (now > entry.resetAt) {
      entry.count = 0;
      entry.resetAt = now + windowMs;
    }
    
    entry.count++;
    counts.set(clientKey, entry);
    
    if (entry.count > max) {
      throw new GraphQLError('Too many requests', {
        extensions: { code: 'RATE_LIMIT_EXCEEDED' }
      });
    }
    
    return resolver(parent, args, context, info);
  };
}

// Logging
function withLogging(resolver) {
  return async (parent, args, context, info) => {
    const start = Date.now();
    try {
      const result = await resolver(parent, args, context, info);
      console.log(`✓ ${info.fieldName} (${Date.now() - start}ms)`);
      return result;
    } catch (error) {
      console.error(`✗ ${info.fieldName}: ${error.message}`);
      throw error;
    }
  };
}

// ใช้งาน
const resolvers = {
  Query: {
    // Compose multiple middlewares
    users: withLogging(
      withAuth(
        async (_, args, context) => {
          return context.dataSources.users.getAll(args);
        }
      )
    ),
    
    adminData: withLogging(
      withAuth(
        withRole('ADMIN')(
          async (_, __, context) => {
            return context.dataSources.admin.getData();
          }
        )
      )
    ),
  },
};
```

---

## 📌 Step 189: Schema Directives เป็น Resolver Middleware

```javascript
// ใช้ mapSchema สำหรับ directive-based middleware
import { mapSchema, getDirective, MapperKind } from '@graphql-tools/utils';

function authDirectiveTransformer(schema) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const authDirective = getDirective(schema, fieldConfig, 'auth')?.[0];
      
      if (authDirective) {
        const { requires } = authDirective;
        const { resolve = defaultFieldResolver } = fieldConfig;
        
        fieldConfig.resolve = async (source, args, context, info) => {
          if (!context.currentUser) {
            throw new GraphQLError('กรุณา Login');
          }
          
          if (requires && context.currentUser.role !== requires) {
            throw new GraphQLError(`ต้องการ role: ${requires}`);
          }
          
          return resolve(source, args, context, info);
        };
      }
      
      return fieldConfig;
    },
  });
}

// ใช้งาน
let schema = makeExecutableSchema({ typeDefs, resolvers });
schema = authDirectiveTransformer(schema);
```

---

## 📌 Step 190: Resolver Performance Patterns

### 1. Lazy Loading

```javascript
const resolvers = {
  User: {
    // ถ้า client ไม่ขอ posts ก็ไม่โหลด
    posts: (user, _, context) => {
      return context.dataSources.posts.getByAuthor(user.id);
    },
  },
};
```

### 2. Field-level Caching

```javascript
const cache = new Map();

const resolvers = {
  Query: {
    expensiveOperation: async (_, { id }) => {
      const cacheKey = `expensive:${id}`;
      
      if (cache.has(cacheKey)) {
        return cache.get(cacheKey);
      }
      
      const result = await doExpensiveWork(id);
      cache.set(cacheKey, result);
      
      // Expire หลัง 5 นาที
      setTimeout(() => cache.delete(cacheKey), 5 * 60 * 1000);
      
      return result;
    },
  },
};
```

### 3. Batching with DataLoader

```javascript
import DataLoader from 'dataloader';

// สร้าง loader ใน context (per request!)
const createLoaders = () => ({
  user: new DataLoader(async (ids) => {
    // 1 query สำหรับ N users
    const users = await db.getUsersByIds([...ids]);
    return ids.map(id => users.find(u => u.id === id) || null);
  }),
  
  postsByAuthor: new DataLoader(async (authorIds) => {
    const posts = await db.getPostsByAuthorIds([...authorIds]);
    return authorIds.map(id => posts.filter(p => p.authorId === id));
  }),
});

// Context
const context = async ({ req }) => ({
  currentUser: await getUser(req),
  loaders: createLoaders(),
});

// Resolver ใช้ loader
const resolvers = {
  Post: {
    author: (post, _, context) => context.loaders.user.load(post.authorId),
  },
  User: {
    posts: (user, _, context) => context.loaders.postsByAuthor.load(user.id),
  },
};
```

---

## 📌 Step 191: Error Handling Patterns

```javascript
import { GraphQLError } from 'graphql';

// Custom error classes
class NotFoundError extends GraphQLError {
  constructor(type, id) {
    super(`${type} (id: ${id}) ไม่พบ`, {
      extensions: { code: 'NOT_FOUND', type, id }
    });
  }
}

class ValidationError extends GraphQLError {
  constructor(errors) {
    super('Validation failed', {
      extensions: { code: 'VALIDATION_ERROR', errors }
    });
  }
}

// ใน resolver
const resolvers = {
  Query: {
    user: async (_, { id }) => {
      const user = await User.findById(id);
      if (!user) throw new NotFoundError('User', id);
      return user;
    },
  },
  
  Mutation: {
    createUser: async (_, { input }) => {
      const errors = {};
      if (!input.name) errors.name = 'ต้องกรอกชื่อ';
      if (!input.email.includes('@')) errors.email = 'Email ไม่ถูกต้อง';
      
      if (Object.keys(errors).length > 0) {
        throw new ValidationError(errors);
      }
      
      return User.create(input);
    },
  },
};
```

---

## 📌 Step 192: สรุป Part 008

### Resolver Signature

```javascript
(parent, args, context, info) => value | Promise<value> | null
```

### เนื้อหาที่เรียนรู้

✅ 4 arguments ของ resolver  
✅ Default resolvers  
✅ Async resolvers  
✅ Execution model (parallel siblings)  
✅ Abstract type resolvers (`__resolveType`)  
✅ Custom scalar resolvers  
✅ Resolver composition (middleware)  
✅ Performance patterns  

### ในส่วนถัดไป

➡️ **[Part 009](./part-009.md)** — Types & Fields: ทุกรายละเอียด

---

*Part 008 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
