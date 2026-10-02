# Part 003 — ติดตั้งและตั้งค่า GraphQL Server แบบสมบูรณ์

> **ระดับ:** Beginner | **เวลาเรียน:** 75 นาที | **ขั้นตอนที่:** 36–60

---

## 🎯 สิ่งที่จะได้เรียนรู้

- ติดตั้ง GraphQL Server ด้วย Apollo Server 4
- ติดตั้งด้วย graphql-yoga (alternative)
- Configuration ขั้นสูง
- CORS, Logging, Error Handling
- Development vs Production setup
- Environment variables
- Project structure แบบ production-ready

---

## 📌 Step 36: ตัวเลือก GraphQL Servers

### 1. Apollo Server 4 — Most Popular

```
ข้อดี:
✅ Documentation ดีมาก
✅ Apollo Studio integration
✅ Active community
✅ TypeScript support
✅ Plugin system

ข้อเสีย:
❌ Opinionated
❌ Vendor lock-in (Apollo ecosystem)
❌ Overhead สำหรับ simple use cases
```

### 2. GraphQL Yoga — Modern & Flexible

```
ข้อดี:
✅ Framework-agnostic
✅ Built on Fetch API (edge-compatible)
✅ Multipart uploads built-in
✅ File upload built-in
✅ Server-Sent Events สำหรับ subscriptions

ข้อเสีย:
❌ Less mature than Apollo
❌ Smaller ecosystem
```

### 3. Mercurius — Fastify

```
ข้อดี:
✅ เร็วที่สุดสำหรับ Fastify
✅ TypeScript support
✅ Federation support

ข้อเสีย:
❌ Tied to Fastify
```

---

## 📌 Step 37: Setup Apollo Server 4 — Full Project

### โครงสร้าง Project

```
graphql-server/
├── src/
│   ├── index.ts              ← entry point
│   ├── server.ts             ← server configuration
│   ├── schema/
│   │   ├── index.ts          ← combine schemas
│   │   ├── user.schema.ts    ← User types
│   │   └── post.schema.ts    ← Post types
│   ├── resolvers/
│   │   ├── index.ts          ← combine resolvers
│   │   ├── user.resolver.ts  ← User resolvers
│   │   └── post.resolver.ts  ← Post resolvers
│   ├── datasources/
│   │   ├── user.datasource.ts
│   │   └── post.datasource.ts
│   ├── models/
│   │   ├── user.model.ts
│   │   └── post.model.ts
│   ├── context.ts            ← GraphQL context
│   └── config.ts             ← configuration
├── package.json
├── tsconfig.json
└── .env
```

### ติดตั้ง Dependencies

```bash
mkdir graphql-server
cd graphql-server
npm init -y

# Core
npm install @apollo/server graphql

# TypeScript
npm install -D typescript ts-node @types/node

# Utils
npm install dotenv cors
npm install -D @types/cors

# Database (ตอนนี้ใช้ mock, ทีหลังเพิ่ม real DB)
```

### `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### `.env`

```env
# Server
NODE_ENV=development
PORT=4000

# CORS
CORS_ORIGIN=http://localhost:3000

# Database (ใช้ต่อไปใน Parts หน้า)
DATABASE_URL=postgresql://localhost:5432/graphql_course

# Auth (ใช้ต่อไปใน Parts หน้า)
JWT_SECRET=your-super-secret-key-change-in-production
JWT_EXPIRY=7d
```

---

## 📌 Step 38: สร้าง Config

### `src/config.ts`

```typescript
import dotenv from 'dotenv';

// โหลด .env ก่อน
dotenv.config();

interface Config {
  nodeEnv: string;
  port: number;
  corsOrigin: string | string[];
  isDevelopment: boolean;
  isProduction: boolean;
  jwtSecret: string;
  jwtExpiry: string;
}

export const config: Config = {
  nodeEnv: process.env.NODE_ENV || 'development',
  port: parseInt(process.env.PORT || '4000', 10),
  corsOrigin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  isDevelopment: process.env.NODE_ENV !== 'production',
  isProduction: process.env.NODE_ENV === 'production',
  jwtSecret: process.env.JWT_SECRET || 'dev-secret-change-in-production',
  jwtExpiry: process.env.JWT_EXPIRY || '7d',
};

// Validate required env vars in production
if (config.isProduction) {
  const required = ['JWT_SECRET', 'DATABASE_URL'];
  const missing = required.filter(key => !process.env[key]);
  
  if (missing.length > 0) {
    throw new Error(`Missing required environment variables: ${missing.join(', ')}`);
  }
}
```

---

## 📌 Step 39: สร้าง Schema

### `src/schema/user.schema.ts`

```typescript
export const userTypeDefs = `#graphql
  """ข้อมูลผู้ใช้งานในระบบ"""
  type User {
    """รหัสเฉพาะของผู้ใช้"""
    id: ID!
    
    """ชื่อผู้ใช้งาน"""
    name: String!
    
    """อีเมล (unique)"""
    email: String!
    
    """อายุ"""
    age: Int
    
    """สถานะการใช้งาน"""
    isActive: Boolean!
    
    """บทบาท"""
    role: UserRole!
    
    """โพสต์ทั้งหมดของผู้ใช้"""
    posts: [Post!]!
    
    """วันที่สร้าง"""
    createdAt: String!
    
    """วันที่แก้ไขล่าสุด"""
    updatedAt: String!
  }

  """บทบาทของผู้ใช้"""
  enum UserRole {
    ADMIN
    USER
    MODERATOR
  }

  """Input สำหรับสร้าง user ใหม่"""
  input CreateUserInput {
    name: String!
    email: String!
    age: Int
    role: UserRole = USER
  }

  """Input สำหรับแก้ไข user"""
  input UpdateUserInput {
    name: String
    email: String
    age: Int
    isActive: Boolean
    role: UserRole
  }

  """ผลลัพธ์ paginated"""
  type UsersConnection {
    """รายการ users"""
    users: [User!]!
    
    """จำนวนทั้งหมด"""
    total: Int!
    
    """page ปัจจุบัน"""
    page: Int!
    
    """จำนวนต่อ page"""
    perPage: Int!
    
    """มี page ถัดไปไหม"""
    hasNextPage: Boolean!
  }
`;
```

### `src/schema/post.schema.ts`

```typescript
export const postTypeDefs = `#graphql
  """โพสต์บทความ"""
  type Post {
    id: ID!
    title: String!
    content: String!
    excerpt: String!
    
    """ผู้เขียน"""
    author: User!
    
    """แท็ก"""
    tags: [String!]!
    
    """สถานะการเผยแพร่"""
    status: PostStatus!
    
    """จำนวนการดู"""
    viewCount: Int!
    
    """ความคิดเห็น"""
    comments: [Comment!]!
    
    createdAt: String!
    updatedAt: String!
  }

  """ความคิดเห็น"""
  type Comment {
    id: ID!
    text: String!
    author: User!
    post: Post!
    createdAt: String!
  }

  """สถานะโพสต์"""
  enum PostStatus {
    DRAFT
    PUBLISHED
    ARCHIVED
  }

  input CreatePostInput {
    title: String!
    content: String!
    tags: [String!]
    status: PostStatus = DRAFT
  }

  input UpdatePostInput {
    title: String
    content: String
    tags: [String!]
    status: PostStatus
  }
`;
```

### `src/schema/index.ts`

```typescript
import { userTypeDefs } from './user.schema.js';
import { postTypeDefs } from './post.schema.js';

// Root query and mutation types
const rootTypeDefs = `#graphql
  type Query {
    # Users
    users(page: Int = 1, perPage: Int = 10): UsersConnection!
    user(id: ID!): User
    me: User
    
    # Posts
    posts(
      tag: String
      status: PostStatus
      authorId: ID
      page: Int = 1
      perPage: Int = 10
    ): [Post!]!
    post(id: ID!): Post
  }

  type Mutation {
    # Users
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User!
    deleteUser(id: ID!): Boolean!
    
    # Posts
    createPost(input: CreatePostInput!): Post!
    updatePost(id: ID!, input: UpdatePostInput!): Post!
    deletePost(id: ID!): Boolean!
    
    # Comments
    createComment(postId: ID!, text: String!): Comment!
    deleteComment(id: ID!): Boolean!
  }
`;

export const typeDefs = [rootTypeDefs, userTypeDefs, postTypeDefs];
```

---

## 📌 Step 40: สร้าง Data Sources

### `src/datasources/user.datasource.ts`

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  age?: number;
  isActive: boolean;
  role: 'ADMIN' | 'USER' | 'MODERATOR';
  createdAt: string;
  updatedAt: string;
}

// Mock database
const mockUsers: User[] = [
  {
    id: '1',
    name: 'สมชาย ใจดี',
    email: 'somchai@example.com',
    age: 28,
    isActive: true,
    role: 'ADMIN',
    createdAt: '2024-01-01T00:00:00Z',
    updatedAt: '2024-01-01T00:00:00Z',
  },
  {
    id: '2',
    name: 'สมหญิง สวยงาม',
    email: 'somying@example.com',
    age: 25,
    isActive: true,
    role: 'USER',
    createdAt: '2024-02-01T00:00:00Z',
    updatedAt: '2024-02-01T00:00:00Z',
  },
  {
    id: '3',
    name: 'มานะ พยายาม',
    email: 'mana@example.com',
    age: 32,
    isActive: false,
    role: 'USER',
    createdAt: '2024-03-01T00:00:00Z',
    updatedAt: '2024-04-01T00:00:00Z',
  },
];

let nextId = 4;

export class UserDataSource {
  async getUsers(page: number = 1, perPage: number = 10) {
    const start = (page - 1) * perPage;
    const end = start + perPage;
    const users = mockUsers.slice(start, end);
    
    return {
      users,
      total: mockUsers.length,
      page,
      perPage,
      hasNextPage: end < mockUsers.length,
    };
  }

  async getUserById(id: string) {
    return mockUsers.find(u => u.id === id) || null;
  }

  async getUserByEmail(email: string) {
    return mockUsers.find(u => u.email === email) || null;
  }

  async createUser(input: {
    name: string;
    email: string;
    age?: number;
    role?: 'ADMIN' | 'USER' | 'MODERATOR';
  }): Promise<User> {
    // ตรวจสอบ email ซ้ำ
    const existing = await this.getUserByEmail(input.email);
    if (existing) {
      throw new Error(`Email ${input.email} already in use`);
    }

    const user: User = {
      id: String(nextId++),
      name: input.name,
      email: input.email,
      age: input.age,
      isActive: true,
      role: input.role || 'USER',
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    };

    mockUsers.push(user);
    return user;
  }

  async updateUser(id: string, input: Partial<Omit<User, 'id' | 'createdAt'>>) {
    const index = mockUsers.findIndex(u => u.id === id);
    if (index === -1) return null;

    mockUsers[index] = {
      ...mockUsers[index],
      ...input,
      updatedAt: new Date().toISOString(),
    };

    return mockUsers[index];
  }

  async deleteUser(id: string): Promise<boolean> {
    const index = mockUsers.findIndex(u => u.id === id);
    if (index === -1) return false;
    mockUsers.splice(index, 1);
    return true;
  }
}
```

---

## 📌 Step 41: สร้าง Context

### `src/context.ts`

```typescript
import { UserDataSource } from './datasources/user.datasource.js';
import { PostDataSource } from './datasources/post.datasource.js';

export interface GraphQLContext {
  // Data sources
  dataSources: {
    users: UserDataSource;
    posts: PostDataSource;
  };
  
  // User จาก auth (ถ้า logged in)
  currentUser?: {
    id: string;
    email: string;
    role: string;
  };
  
  // Request info
  requestId: string;
}

export async function createContext(): Promise<GraphQLContext> {
  return {
    dataSources: {
      users: new UserDataSource(),
      posts: new PostDataSource(),
    },
    requestId: `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
  };
}
```

---

## 📌 Step 42: สร้าง Resolvers

### `src/resolvers/user.resolver.ts`

```typescript
import { GraphQLContext } from '../context.js';

export const userResolvers = {
  Query: {
    users: async (
      _: unknown,
      args: { page?: number; perPage?: number },
      context: GraphQLContext
    ) => {
      const { page = 1, perPage = 10 } = args;
      return context.dataSources.users.getUsers(page, perPage);
    },

    user: async (
      _: unknown,
      args: { id: string },
      context: GraphQLContext
    ) => {
      return context.dataSources.users.getUserById(args.id);
    },

    me: async (
      _: unknown,
      __: unknown,
      context: GraphQLContext
    ) => {
      if (!context.currentUser) return null;
      return context.dataSources.users.getUserById(context.currentUser.id);
    },
  },

  Mutation: {
    createUser: async (
      _: unknown,
      args: { input: { name: string; email: string; age?: number } },
      context: GraphQLContext
    ) => {
      return context.dataSources.users.createUser(args.input);
    },

    updateUser: async (
      _: unknown,
      args: { id: string; input: Record<string, unknown> },
      context: GraphQLContext
    ) => {
      const user = await context.dataSources.users.updateUser(args.id, args.input);
      if (!user) throw new Error(`User ${args.id} not found`);
      return user;
    },

    deleteUser: async (
      _: unknown,
      args: { id: string },
      context: GraphQLContext
    ) => {
      return context.dataSources.users.deleteUser(args.id);
    },
  },

  // Field resolvers
  User: {
    posts: async (
      parent: { id: string },
      _: unknown,
      context: GraphQLContext
    ) => {
      return context.dataSources.posts.getPostsByAuthor(parent.id);
    },
  },
};
```

### `src/resolvers/index.ts`

```typescript
import { userResolvers } from './user.resolver.js';
import { postResolvers } from './post.resolver.js';
import { mergeResolvers } from '@graphql-tools/merge';

export const resolvers = mergeResolvers([
  userResolvers,
  postResolvers,
]);
```

---

## 📌 Step 43: สร้าง Apollo Server

### `src/server.ts`

```typescript
import { ApolloServer } from '@apollo/server';
import { ApolloServerPluginDrainHttpServer } from '@apollo/server/plugin/drainHttpServer';
import { ApolloServerPluginLandingPageLocalDefault } from '@apollo/server/plugin/landingPage/default';
import { ApolloServerPluginLandingPageDisabled } from '@apollo/server/plugin/disabled';
import http from 'http';
import { typeDefs } from './schema/index.js';
import { resolvers } from './resolvers/index.js';
import { config } from './config.js';
import { GraphQLContext } from './context.js';

export function createApolloServer(httpServer: http.Server) {
  const plugins = [
    // Graceful shutdown
    ApolloServerPluginDrainHttpServer({ httpServer }),
    
    // Playground: เปิดใน development, ปิดใน production
    config.isDevelopment
      ? ApolloServerPluginLandingPageLocalDefault({ 
          embed: true,
          includeCookies: true,
        })
      : ApolloServerPluginLandingPageDisabled(),
  ];

  const server = new ApolloServer<GraphQLContext>({
    typeDefs,
    resolvers,
    plugins,
    
    // ปิด introspection ใน production (security)
    introspection: config.isDevelopment,
    
    // Format errors ก่อน send ไป client
    formatError: (formattedError, error) => {
      // Log error details ใน server
      console.error('GraphQL Error:', {
        message: formattedError.message,
        path: formattedError.path,
        locations: formattedError.locations,
      });
      
      // ใน production: อย่า expose internal error details
      if (config.isProduction) {
        // ซ่อน stack trace
        return {
          message: formattedError.message,
          locations: formattedError.locations,
          path: formattedError.path,
          extensions: {
            code: formattedError.extensions?.code,
          },
        };
      }
      
      return formattedError;
    },
  });

  return server;
}
```

---

## 📌 Step 44: สร้าง Entry Point

### `src/index.ts`

```typescript
import express from 'express';
import http from 'http';
import cors from 'cors';
import { expressMiddleware } from '@apollo/server/express4';
import { json } from 'body-parser';
import { createApolloServer } from './server.js';
import { createContext } from './context.js';
import { config } from './config.js';

async function main() {
  // สร้าง Express app
  const app = express();
  const httpServer = http.createServer(app);

  // สร้าง Apollo Server
  const apolloServer = createApolloServer(httpServer);

  // เริ่ม Apollo Server ก่อน
  await apolloServer.start();

  // Middleware
  app.use(
    '/graphql',
    cors<cors.CorsRequest>({
      origin: config.corsOrigin,
      credentials: true,
    }),
    json(),
    expressMiddleware(apolloServer, {
      context: createContext,
    })
  );

  // Health check endpoint
  app.get('/health', (_, res) => {
    res.json({
      status: 'ok',
      timestamp: new Date().toISOString(),
      environment: config.nodeEnv,
    });
  });

  // เริ่ม server
  await new Promise<void>((resolve) => {
    httpServer.listen({ port: config.port }, resolve);
  });

  console.log(`🚀 GraphQL Server ready at http://localhost:${config.port}/graphql`);
  console.log(`❤️  Health check at http://localhost:${config.port}/health`);
  
  if (config.isDevelopment) {
    console.log(`📊 Apollo Sandbox at http://localhost:${config.port}/graphql`);
  }
}

main().catch((error) => {
  console.error('Failed to start server:', error);
  process.exit(1);
});
```

---

## 📌 Step 45: ติดตั้ง Missing Dependencies

```bash
npm install express @graphql-tools/merge body-parser
npm install -D @types/express

# สำหรับ ts-node
npm install -D ts-node
```

### อัพเดท `package.json`

```json
{
  "name": "graphql-server",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "ts-node --esm src/index.ts",
    "dev:watch": "nodemon --exec 'ts-node --esm src/index.ts' src/**/*.ts"
  },
  "dependencies": {
    "@apollo/server": "^4.10.0",
    "@graphql-tools/merge": "^9.0.0",
    "body-parser": "^1.20.0",
    "cors": "^2.8.5",
    "dotenv": "^16.0.0",
    "express": "^4.18.0",
    "graphql": "^16.8.0"
  },
  "devDependencies": {
    "@types/cors": "^2.8.0",
    "@types/express": "^4.17.0",
    "@types/node": "^20.0.0",
    "nodemon": "^3.0.0",
    "ts-node": "^10.9.0",
    "typescript": "^5.0.0"
  }
}
```

---

## 📌 Step 46: Alternative — GraphQL Yoga Setup

GraphQL Yoga เป็น modern alternative ที่ flexibility สูงกว่า:

```bash
mkdir graphql-yoga-server
cd graphql-yoga-server
npm init -y
npm install graphql-yoga graphql
npm install -D typescript ts-node @types/node
```

### `src/index.ts` ด้วย Yoga

```typescript
import { createSchema, createYoga } from 'graphql-yoga';
import { createServer } from 'http';

const typeDefs = `#graphql
  type Query {
    hello: String!
    user(id: ID!): User
    users: [User!]!
  }

  type Mutation {
    createUser(name: String!, email: String!): User!
  }

  type User {
    id: ID!
    name: String!
    email: String!
  }
`;

const users = [
  { id: '1', name: 'สมชาย', email: 'somchai@example.com' },
  { id: '2', name: 'สมหญิง', email: 'somying@example.com' },
];

const resolvers = {
  Query: {
    hello: () => 'สวัสดี GraphQL Yoga!',
    user: (_: unknown, { id }: { id: string }) => 
      users.find(u => u.id === id),
    users: () => users,
  },
  Mutation: {
    createUser: (_: unknown, { name, email }: { name: string; email: string }) => {
      const user = {
        id: String(users.length + 1),
        name,
        email,
      };
      users.push(user);
      return user;
    },
  },
};

// สร้าง schema
const schema = createSchema({ typeDefs, resolvers });

// สร้าง Yoga instance
const yoga = createYoga({
  schema,
  // เปิด GraphiQL playground
  graphiql: {
    title: 'GraphQL Yoga Playground',
  },
  cors: {
    origin: 'http://localhost:3000',
    credentials: true,
  },
});

// สร้าง HTTP server
const server = createServer(yoga);

server.listen(4000, () => {
  console.log('🚀 GraphQL Yoga ready at http://localhost:4000/graphql');
});
```

---

## 📌 Step 47: Logging Setup

### `src/plugins/logging.plugin.ts`

```typescript
import { ApolloServerPlugin, GraphQLRequestContext } from '@apollo/server';
import { GraphQLContext } from '../context.js';

export const loggingPlugin: ApolloServerPlugin<GraphQLContext> = {
  // เมื่อรับ request
  async requestDidStart(requestContext) {
    const { request, contextValue } = requestContext;
    const startTime = Date.now();
    
    console.log(`[${contextValue.requestId}] GraphQL Request started`, {
      operationName: request.operationName,
      query: request.query?.substring(0, 200), // truncate long queries
    });

    return {
      // เมื่อ execute เสร็จ
      async executionDidStart() {
        return {
          async executionDidEnd(error?: Error) {
            if (error) {
              console.error(`[${contextValue.requestId}] Execution error:`, error);
            }
          },
        };
      },

      // เมื่อมี errors
      async didEncounterErrors({ errors }) {
        errors.forEach(error => {
          console.error(`[${contextValue.requestId}] GraphQL error:`, {
            message: error.message,
            locations: error.locations,
            path: error.path,
          });
        });
      },

      // เมื่อส่ง response
      async willSendResponse({ response }) {
        const duration = Date.now() - startTime;
        console.log(`[${contextValue.requestId}] Request completed in ${duration}ms`);
      },
    };
  },
};
```

---

## 📌 Step 48: CORS Configuration

```typescript
// src/middleware/cors.ts
import cors from 'cors';
import { config } from '../config.js';

const allowedOrigins = Array.isArray(config.corsOrigin)
  ? config.corsOrigin
  : [config.corsOrigin];

export const corsOptions: cors.CorsOptions = {
  origin: (origin, callback) => {
    // อนุญาต requests ที่ไม่มี origin (เช่น curl, Postman)
    if (!origin) return callback(null, true);
    
    if (allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      console.warn(`CORS blocked request from origin: ${origin}`);
      callback(new Error(`Origin ${origin} not allowed by CORS`));
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400, // 24 hours
};
```

---

## 📌 Step 49: Error Handling Patterns

### Custom Error Classes

```typescript
// src/errors/index.ts
import { GraphQLError } from 'graphql';

export class AuthenticationError extends GraphQLError {
  constructor(message: string) {
    super(message, {
      extensions: {
        code: 'UNAUTHENTICATED',
        http: { status: 401 },
      },
    });
    Object.defineProperty(this, 'name', { value: 'AuthenticationError' });
  }
}

export class AuthorizationError extends GraphQLError {
  constructor(message: string) {
    super(message, {
      extensions: {
        code: 'FORBIDDEN',
        http: { status: 403 },
      },
    });
    Object.defineProperty(this, 'name', { value: 'AuthorizationError' });
  }
}

export class NotFoundError extends GraphQLError {
  constructor(resource: string, id: string) {
    super(`${resource} with id ${id} not found`, {
      extensions: {
        code: 'NOT_FOUND',
        http: { status: 404 },
      },
    });
    Object.defineProperty(this, 'name', { value: 'NotFoundError' });
  }
}

export class ValidationError extends GraphQLError {
  constructor(message: string, fields?: Record<string, string>) {
    super(message, {
      extensions: {
        code: 'VALIDATION_ERROR',
        fields,
        http: { status: 400 },
      },
    });
    Object.defineProperty(this, 'name', { value: 'ValidationError' });
  }
}
```

### ใช้งาน Custom Errors

```typescript
import { NotFoundError, ValidationError } from '../errors/index.js';

const resolvers = {
  Query: {
    user: async (_, { id }, context) => {
      const user = await context.dataSources.users.getUserById(id);
      
      if (!user) {
        throw new NotFoundError('User', id);
      }
      
      return user;
    },
  },
  
  Mutation: {
    createUser: async (_, { input }, context) => {
      // Validate
      if (!input.email.includes('@')) {
        throw new ValidationError('Invalid email format', {
          email: 'Must be a valid email address',
        });
      }
      
      const existing = await context.dataSources.users.getUserByEmail(input.email);
      if (existing) {
        throw new ValidationError('Email already in use', {
          email: 'This email is already registered',
        });
      }
      
      return context.dataSources.users.createUser(input);
    },
  },
};
```

---

## 📌 Step 50: Testing การติดตั้ง

### ทดสอบ Health Check

```bash
curl http://localhost:4000/health

# Response:
# {
#   "status": "ok",
#   "timestamp": "2024-10-02T00:00:00.000Z",
#   "environment": "development"
# }
```

### ทดสอบ GraphQL Query

```bash
curl -X POST http://localhost:4000/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ users(page: 1, perPage: 10) { users { id name email } total } }"
  }'
```

### ทดสอบด้วย Query Variables

```bash
curl -X POST http://localhost:4000/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query GetUser($id: ID!) { user(id: $id) { name email } }",
    "variables": { "id": "1" }
  }'
```

---

## 📌 Step 51: Graceful Shutdown

```typescript
// src/index.ts — เพิ่ม graceful shutdown
async function main() {
  // ... (setup code)

  // Graceful shutdown
  const shutdown = async (signal: string) => {
    console.log(`\nReceived ${signal}, shutting down gracefully...`);
    
    // หยุด รับ connections ใหม่
    httpServer.close(async () => {
      console.log('HTTP server closed');
      
      // ปิด Apollo Server
      await apolloServer.stop();
      console.log('Apollo Server stopped');
      
      // ปิด database connections (ทำใน Parts หน้า)
      // await db.disconnect();
      
      console.log('Shutdown complete');
      process.exit(0);
    });
    
    // Force shutdown หลัง 10 วินาที
    setTimeout(() => {
      console.error('Could not close connections in time, forcing shutdown');
      process.exit(1);
    }, 10000);
  };

  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT', () => shutdown('SIGINT'));
}
```

---

## 📌 Step 52: Development Workflow

### Nodemon Configuration

```json
// nodemon.json
{
  "watch": ["src"],
  "ext": "ts,json",
  "ignore": ["src/**/*.spec.ts", "src/**/*.test.ts"],
  "exec": "ts-node --esm src/index.ts",
  "env": {
    "NODE_ENV": "development"
  }
}
```

### Scripts ใน package.json

```json
{
  "scripts": {
    "build": "tsc --noEmit && tsc",
    "start": "node dist/index.js",
    "dev": "nodemon",
    "typecheck": "tsc --noEmit",
    "lint": "eslint src/**/*.ts",
    "test": "jest"
  }
}
```

---

## 📌 Step 53: Production Configuration

```typescript
// src/config.ts — production additions
export const config = {
  // ...existing config
  
  // Rate limiting
  rateLimit: {
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // limit each IP to 100 requests per window
  },
  
  // Query complexity
  maxQueryDepth: config.isProduction ? 7 : 15,
  maxQueryComplexity: config.isProduction ? 1000 : 5000,
  
  // Timeouts
  requestTimeout: 30000, // 30 seconds
  
  // Logging
  logLevel: config.isProduction ? 'error' : 'debug',
};
```

---

## 📌 Step 54: Docker Setup

### `Dockerfile`

```dockerfile
# Build stage
FROM node:20-alpine AS builder

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy source
COPY . .

# Build TypeScript
RUN npm run build

# Production stage
FROM node:20-alpine AS production

WORKDIR /app

# Copy built files
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

# Run as non-root user
USER node

EXPOSE 4000

CMD ["node", "dist/index.js"]
```

### `docker-compose.yml`

```yaml
version: '3.8'

services:
  graphql-server:
    build: .
    ports:
      - "4000:4000"
    environment:
      - NODE_ENV=production
      - PORT=4000
      - JWT_SECRET=${JWT_SECRET}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
```

---

## 📌 Step 55: Environment Variables Best Practices

```bash
# .env.example (commit ไปใน git ได้)
NODE_ENV=development
PORT=4000
CORS_ORIGIN=http://localhost:3000
DATABASE_URL=postgresql://localhost:5432/graphql_dev
JWT_SECRET=CHANGE_THIS_IN_PRODUCTION
JWT_EXPIRY=7d

# .env (อย่า commit!)
# เพิ่มใน .gitignore
NODE_ENV=development
PORT=4000
CORS_ORIGIN=http://localhost:3000
DATABASE_URL=postgresql://localhost:5432/graphql_dev
JWT_SECRET=my-super-secret-key-123
JWT_EXPIRY=7d
```

### `.gitignore`

```
node_modules/
dist/
.env
.env.local
.env.production
*.log
coverage/
```

---

## 📌 Step 56: Schema ครบสมบูรณ์ + PostDataSource

### `src/datasources/post.datasource.ts`

```typescript
interface Post {
  id: string;
  title: string;
  content: string;
  authorId: string;
  tags: string[];
  status: 'DRAFT' | 'PUBLISHED' | 'ARCHIVED';
  viewCount: number;
  createdAt: string;
  updatedAt: string;
}

const mockPosts: Post[] = [
  {
    id: '1',
    title: 'เริ่มต้นกับ GraphQL',
    content: 'GraphQL เป็นภาษา query ที่ยืดหยุ่น...',
    authorId: '1',
    tags: ['graphql', 'api'],
    status: 'PUBLISHED',
    viewCount: 150,
    createdAt: '2024-03-01T00:00:00Z',
    updatedAt: '2024-03-01T00:00:00Z',
  },
  {
    id: '2',
    title: 'Node.js Best Practices ปี 2024',
    content: 'สิ่งที่ควรรู้เกี่ยวกับ Node.js...',
    authorId: '1',
    tags: ['nodejs', 'javascript'],
    status: 'PUBLISHED',
    viewCount: 89,
    createdAt: '2024-04-01T00:00:00Z',
    updatedAt: '2024-04-01T00:00:00Z',
  },
  {
    id: '3',
    title: 'React Hooks ทำความเข้าใจ',
    content: 'Hooks เปลี่ยนวิธีเขียน React...',
    authorId: '2',
    tags: ['react', 'javascript'],
    status: 'DRAFT',
    viewCount: 0,
    createdAt: '2024-05-01T00:00:00Z',
    updatedAt: '2024-05-01T00:00:00Z',
  },
];

export class PostDataSource {
  async getPosts(filters?: {
    tag?: string;
    status?: string;
    authorId?: string;
    page?: number;
    perPage?: number;
  }) {
    let result = [...mockPosts];
    
    if (filters?.tag) {
      result = result.filter(p => p.tags.includes(filters.tag!));
    }
    if (filters?.status) {
      result = result.filter(p => p.status === filters.status);
    }
    if (filters?.authorId) {
      result = result.filter(p => p.authorId === filters.authorId);
    }
    
    return result;
  }

  async getPostById(id: string) {
    return mockPosts.find(p => p.id === id) || null;
  }

  async getPostsByAuthor(authorId: string) {
    return mockPosts.filter(p => p.authorId === authorId);
  }

  async getExcerpt(post: Post): Promise<string> {
    return post.content.substring(0, 150) + (post.content.length > 150 ? '...' : '');
  }
}
```

---

## 📌 Step 57: Post Resolver

```typescript
// src/resolvers/post.resolver.ts
import { GraphQLContext } from '../context.js';

export const postResolvers = {
  Query: {
    posts: async (_: unknown, args: {
      tag?: string;
      status?: string;
      authorId?: string;
      page?: number;
      perPage?: number;
    }, context: GraphQLContext) => {
      return context.dataSources.posts.getPosts(args);
    },

    post: async (_: unknown, { id }: { id: string }, context: GraphQLContext) => {
      return context.dataSources.posts.getPostById(id);
    },
  },

  Mutation: {
    createPost: async (
      _: unknown,
      { input }: { input: { title: string; content: string; tags?: string[] } },
      context: GraphQLContext
    ) => {
      if (!context.currentUser) {
        throw new Error('Authentication required');
      }
      // Implementation ใน Part 015 (Auth)
      return null;
    },
  },

  Post: {
    author: async (parent: { authorId: string }, _: unknown, context: GraphQLContext) => {
      return context.dataSources.users.getUserById(parent.authorId);
    },
    
    excerpt: async (parent: { content: string }) => {
      return parent.content.substring(0, 150) + 
        (parent.content.length > 150 ? '...' : '');
    },
    
    comments: async (parent: { id: string }, _: unknown, context: GraphQLContext) => {
      // Implementation with comments datasource
      return [];
    },
  },
};
```

---

## 📌 Step 58: ทดสอบ Full Setup

```graphql
# Test 1: Query users
query GetUsers {
  users(page: 1, perPage: 10) {
    users {
      id
      name
      email
      role
      isActive
    }
    total
    page
    hasNextPage
  }
}

# Test 2: Query user with posts
query UserWithPosts {
  user(id: "1") {
    name
    email
    role
    posts {
      id
      title
      status
      tags
      viewCount
    }
  }
}

# Test 3: Create user
mutation CreateUser {
  createUser(input: {
    name: "วิชัย เก่งมาก"
    email: "wichai@example.com"
    age: 30
  }) {
    id
    name
    email
    createdAt
  }
}

# Test 4: Filter posts
query FilteredPosts {
  posts(tag: "graphql", status: PUBLISHED) {
    id
    title
    tags
    viewCount
  }
}
```

---

## 📌 Step 59: Monitoring — Basic Metrics

```typescript
// src/plugins/metrics.plugin.ts
import { ApolloServerPlugin } from '@apollo/server';
import { GraphQLContext } from '../context.js';

interface Metrics {
  totalRequests: number;
  errorCount: number;
  avgResponseTime: number;
  operationCounts: Record<string, number>;
}

const metrics: Metrics = {
  totalRequests: 0,
  errorCount: 0,
  avgResponseTime: 0,
  operationCounts: {},
};

export const metricsPlugin: ApolloServerPlugin<GraphQLContext> = {
  async requestDidStart({ request }) {
    metrics.totalRequests++;
    const startTime = Date.now();
    const operationName = request.operationName || 'anonymous';
    
    metrics.operationCounts[operationName] = 
      (metrics.operationCounts[operationName] || 0) + 1;

    return {
      async didEncounterErrors() {
        metrics.errorCount++;
      },
      async willSendResponse() {
        const duration = Date.now() - startTime;
        // Running average
        metrics.avgResponseTime = 
          (metrics.avgResponseTime * (metrics.totalRequests - 1) + duration) / 
          metrics.totalRequests;
      },
    };
  },
};

// Expose metrics endpoint
export function getMetrics() {
  return {
    ...metrics,
    errorRate: metrics.totalRequests > 0 
      ? metrics.errorCount / metrics.totalRequests 
      : 0,
  };
}
```

---

## 📌 Step 60: สรุป Part 003

### เนื้อหาที่เรียนรู้

✅ โครงสร้าง Project แบบ production-ready  
✅ Apollo Server 4 setup พร้อม Express  
✅ GraphQL Yoga เป็น alternative  
✅ TypeScript configuration  
✅ Environment variables  
✅ Custom error classes  
✅ CORS, Logging, Metrics  
✅ Graceful shutdown  
✅ Docker configuration  

### Homework

1. รัน server ให้ทำงานได้ครบทุก endpoint
2. เพิ่ม `PostDataSource` ให้ครบสมบูรณ์
3. เพิ่ม `CommentsDataSource`
4. ลองเปลี่ยน `console.log` เป็น winston logger

### ในส่วนถัดไป

➡️ **[Part 004](./part-004.md)** — Schema Definition Language (SDL): ทุกอย่างที่ต้องรู้เกี่ยวกับ Types

---

*Part 003 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
