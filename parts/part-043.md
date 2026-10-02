# Part 043 — GraphQL Testing Strategies 🧪

> **ระดับ:** Intermediate-Advanced | **เวลาเรียน:** 120 นาที | **ขั้นตอนที่:** 761–800

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Testing pyramid สำหรับ GraphQL
- Unit tests สำหรับ resolvers
- Integration tests ด้วย executeOperation
- End-to-end tests
- Contract testing
- Mock strategies
- Performance testing
- Snapshot testing

---

## 📌 Step 761: Testing Pyramid สำหรับ GraphQL

```
       ┌──────────────┐
       │   E2E Tests  │  ← น้อย, ช้า, ครอบคลุม user journeys
       ├──────────────┤
       │ Integration  │  ← ปานกลาง, executeOperation
       │   Tests      │
       ├──────────────┤
       │  Unit Tests  │  ← มาก, เร็ว, isolate resolvers
       └──────────────┘

Unit Tests:
- Resolver logic
- Helper functions
- Custom scalars
- Validation logic

Integration Tests:
- GraphQL operations
- Auth flows
- Database operations

E2E Tests:
- Complete user journeys
- Browser-based (Playwright)
```

---

## 📌 Step 762: Unit Tests สำหรับ Resolvers

```javascript
// src/resolvers/__tests__/user.resolver.test.js
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { userResolvers } from '../user.resolver.js';
import { GraphQLError } from 'graphql';

// Mock Prisma
const mockPrisma = {
  user: {
    findUnique: vi.fn(),
    findMany: vi.fn(),
    create: vi.fn(),
    update: vi.fn(),
    delete: vi.fn(),
  },
};

// Mock Context
const mockContext = (overrides = {}) => ({
  prisma: mockPrisma,
  user: { sub: 'user-123', role: 'USER', email: 'test@example.com' },
  loaders: {
    userById: { load: vi.fn() },
    postsByUserId: { load: vi.fn() },
  },
  req: { ip: '127.0.0.1' },
  ...overrides,
});

describe('userResolvers', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });
  
  describe('Query.me', () => {
    it('should return current user', async () => {
      const mockUser = { id: 'user-123', name: 'Alice', email: 'alice@example.com' };
      mockPrisma.user.findUnique.mockResolvedValue(mockUser);
      
      const result = await userResolvers.Query.me(null, {}, mockContext());
      
      expect(result).toEqual(mockUser);
      expect(mockPrisma.user.findUnique).toHaveBeenCalledWith({
        where: { id: 'user-123' }
      });
    });
    
    it('should return null when not authenticated', async () => {
      const result = await userResolvers.Query.me(
        null,
        {},
        mockContext({ user: null })
      );
      
      expect(result).toBeNull();
      expect(mockPrisma.user.findUnique).not.toHaveBeenCalled();
    });
  });
  
  describe('Mutation.updateProfile', () => {
    it('should update user profile', async () => {
      const input = { name: 'Alice Updated', bio: 'New bio' };
      const updatedUser = { id: 'user-123', ...input };
      mockPrisma.user.update.mockResolvedValue(updatedUser);
      
      const result = await userResolvers.Mutation.updateProfile(
        null,
        { input },
        mockContext()
      );
      
      expect(result).toEqual(updatedUser);
    });
    
    it('should throw when not authenticated', async () => {
      await expect(
        userResolvers.Mutation.updateProfile(
          null,
          { input: { name: 'Test' } },
          mockContext({ user: null })
        )
      ).rejects.toThrow(GraphQLError);
    });
  });
  
  describe('User.posts', () => {
    it('should load posts via DataLoader', async () => {
      const mockPosts = [{ id: 'post-1', title: 'Hello' }];
      const mockLoader = { load: vi.fn().mockResolvedValue(mockPosts) };
      
      const result = await userResolvers.User.posts(
        { id: 'user-123' },
        {},
        mockContext({ loaders: { postsByUserId: mockLoader } })
      );
      
      expect(mockLoader.load).toHaveBeenCalledWith('user-123');
      expect(result).toEqual(mockPosts);
    });
  });
});
```

---

## 📌 Step 763: Integration Tests

```javascript
// test/integration/user.integration.test.js
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { ApolloServer } from '@apollo/server';
import { typeDefs, resolvers } from '../../src/schema.js';
import { prisma } from '../../src/db.js';

let server;
let testUser;

beforeAll(async () => {
  server = new ApolloServer({ typeDefs, resolvers });
  await server.start();
  
  // Create test user
  testUser = await prisma.user.create({
    data: {
      name: 'Test User',
      email: `test-${Date.now()}@example.com`,
      password: await bcrypt.hash('password123', 10),
    },
  });
});

afterAll(async () => {
  await prisma.user.delete({ where: { id: testUser.id } });
  await server.stop();
});

describe('User Queries', () => {
  it('GET_USER: returns user by ID', async () => {
    const GET_USER = `
      query GetUser($id: ID!) {
        user(id: $id) {
          id
          name
          email
        }
      }
    `;
    
    const response = await server.executeOperation(
      { query: GET_USER, variables: { id: testUser.id } },
      {
        contextValue: {
          prisma,
          user: { sub: testUser.id, role: 'USER' },
          loaders: createLoaders(),
        }
      }
    );
    
    expect(response.body.singleResult.errors).toBeUndefined();
    expect(response.body.singleResult.data.user).toMatchObject({
      id: testUser.id,
      name: 'Test User',
      email: testUser.email,
    });
  });
  
  it('UPDATE_PROFILE: updates user data', async () => {
    const UPDATE_PROFILE = `
      mutation UpdateProfile($input: UpdateProfileInput!) {
        updateProfile(input: $input) {
          id
          name
          bio
        }
      }
    `;
    
    const response = await server.executeOperation(
      {
        query: UPDATE_PROFILE,
        variables: { input: { name: 'Updated Name', bio: 'New bio' } }
      },
      {
        contextValue: {
          prisma,
          user: { sub: testUser.id, role: 'USER' },
        }
      }
    );
    
    expect(response.body.singleResult.errors).toBeUndefined();
    expect(response.body.singleResult.data.updateProfile.name).toBe('Updated Name');
  });
});
```

---

## 📌 Step 764: Mock Strategies

```javascript
// Strategy 1: In-memory database (SQLite)
// ใช้ prisma + SQLite สำหรับ test environment

// prisma/.env.test
// DATABASE_URL="file:./test.db"

// ใน test setup
beforeAll(async () => {
  await prisma.$executeRaw`PRAGMA journal_mode=WAL`; // Better concurrent access
  await prisma.$executeRaw`DELETE FROM users`;
  await prisma.$executeRaw`DELETE FROM posts`;
});

// Strategy 2: Factory pattern
import { faker } from '@faker-js/faker';

export const UserFactory = {
  build: (overrides = {}) => ({
    id: faker.string.uuid(),
    name: faker.person.fullName(),
    email: faker.internet.email(),
    role: 'USER',
    createdAt: new Date(),
    ...overrides,
  }),
  
  create: async (overrides = {}) => {
    return prisma.user.create({
      data: UserFactory.build(overrides),
    });
  },
  
  createMany: async (count, overrides = {}) => {
    return Promise.all(
      Array.from({ length: count }, () => UserFactory.create(overrides))
    );
  },
};
```

---

## 📌 Step 765: Contract Tests

```javascript
// ตรวจสอบว่า schema ยังรองรับ client queries

// test/contracts/product.contract.test.js
import { validate } from 'graphql';
import { schema } from '../../src/schema.js';
import { parse } from 'graphql';

const CLIENT_QUERIES = [
  // จาก mobile app
  `query GetProduct($id: ID!) {
    product(id: $id) {
      id name price
      images { url thumbnailUrl }
      inventory { availableQuantity }
    }
  }`,
  
  // จาก web frontend
  `query GetProducts($filter: ProductFilterInput) {
    products(filter: $filter) {
      edges { node { id name price } }
      pageInfo { hasNextPage endCursor }
    }
  }`,
];

describe('Schema Contracts', () => {
  CLIENT_QUERIES.forEach((query, i) => {
    it(`should support client query #${i + 1}`, () => {
      const errors = validate(schema, parse(query));
      expect(errors).toHaveLength(0);
    });
  });
});
```

---

## 📌 Step 766: Performance Testing

```javascript
// test/performance/load.test.js
import autocannon from 'autocannon';

const instance = autocannon({
  url: 'http://localhost:4000/graphql',
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    query: `query { products(first: 10) { edges { node { id name price } } } }`,
  }),
  connections: 10,    // 10 concurrent connections
  pipelining: 1,
  duration: 10,       // 10 seconds
});

autocannon.track(instance);

instance.on('done', (results) => {
  console.log(`Requests/sec: ${results.requests.average}`);
  console.log(`Latency P99: ${results.latency.p99}ms`);
  
  // Assertions
  expect(results.requests.average).toBeGreaterThan(100); // > 100 req/s
  expect(results.latency.p99).toBeLessThan(500); // P99 < 500ms
  expect(results.errors).toBe(0);
});
```

---

## 📌 Step 767: สรุป Part 043

### เนื้อหาที่เรียนรู้

✅ Testing pyramid  
✅ Unit tests สำหรับ resolvers  
✅ Integration tests ด้วย executeOperation  
✅ Factory pattern  
✅ Contract tests  
✅ Performance testing  

### Testing Best Practices

```
1. Unit test ทุก resolver
2. Integration test ทุก operation
3. Contract test ป้องกัน breaking changes
4. Performance test ก่อน production
5. ใช้ factory แทน hardcoded data
6. Isolate tests (cleanup after each)
7. Test auth flows อย่างละเอียด
8. Test error cases ไม่ใช่แค่ happy path
```

### ในส่วนถัดไป

➡️ **[Part 044](./part-044.md)** — Introspection Attacks & Defense (Security)
➡️ **[Part 046](./part-046.md)** — Query Depth & Complexity Security

---

*Part 043 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
