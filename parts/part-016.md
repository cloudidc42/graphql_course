# Part 016 — Testing GraphQL APIs 🧪

> **ระดับ:** Intermediate | **เวลาเรียน:** 120 นาที | **ขั้นตอนที่:** 511–550

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Unit testing resolvers
- Integration testing ด้วย ApolloServer.executeOperation
- End-to-end testing
- Test fixtures และ factories
- Mocking dependencies
- Snapshot testing
- Coverage reporting
- CI/CD testing pipeline

---

## 📌 Step 511: Testing Strategy

```
Testing Pyramid สำหรับ GraphQL:

          /\
         /E2E\       ← น้อย, ช้า, สมจริง
        /------\
       /  Integ \    ← ปานกลาง
      /----------\
     / Unit Tests \  ← เยอะ, เร็ว, isolated
    /--------------\

Unit Tests:
  - Resolver functions
  - Business logic
  - Utility functions
  - Custom scalars

Integration Tests:
  - Full resolver chain
  - Auth + context
  - Database interactions

E2E Tests:
  - Real HTTP requests
  - Full stack (DB + server + client)
```

---

## 📌 Step 512: Setup Testing Environment

```bash
npm install --save-dev jest @types/jest ts-jest
npm install --save-dev @apollo/server
npm install --save-dev supertest
```

### jest.config.js

```javascript
/** @type {import('jest').Config} */
export default {
  preset: 'ts-jest/presets/default-esm',
  extensionsToTreatAsEsm: ['.ts'],
  moduleNameMapper: {
    '^(\\.{1,2}/.*)\\.js$': '$1',
  },
  testEnvironment: 'node',
  coverageDirectory: 'coverage',
  collectCoverageFrom: [
    'src/**/*.{ts,js}',
    '!src/**/*.d.ts',
    '!src/index.ts',
  ],
  testMatch: [
    '**/__tests__/**/*.{ts,js}',
    '**/*.test.{ts,js}',
    '**/*.spec.{ts,js}',
  ],
  setupFilesAfterFramework: ['./test/setup.ts'],
};
```

---

## 📌 Step 513: Unit Testing Resolvers

```javascript
// src/resolvers/__tests__/user.resolver.test.js
import { userResolvers } from '../user.resolver.js';
import { User } from '../../models/user.model.js';
import { NotFoundError, AuthenticationError } from '../../errors/index.js';

// Mock model
jest.mock('../../models/user.model.js');

describe('User Resolvers', () => {
  afterEach(() => jest.clearAllMocks());
  
  describe('Query.user', () => {
    it('should return user by id', async () => {
      const mockUser = { id: '1', name: 'Alice', email: 'alice@example.com' };
      User.findById.mockResolvedValue(mockUser);
      
      const context = { user: { sub: '99', role: 'ADMIN' } };
      const result = await userResolvers.Query.user(null, { id: '1' }, context, {});
      
      expect(result).toEqual(mockUser);
      expect(User.findById).toHaveBeenCalledWith('1');
    });
    
    it('should throw NotFoundError for missing user', async () => {
      User.findById.mockResolvedValue(null);
      
      const context = { user: { sub: '99', role: 'ADMIN' } };
      
      await expect(
        userResolvers.Query.user(null, { id: '999' }, context, {})
      ).rejects.toThrow(NotFoundError);
    });
    
    it('should throw AuthenticationError when not logged in', async () => {
      const context = {};  // ไม่มี user
      
      await expect(
        userResolvers.Query.user(null, { id: '1' }, context, {})
      ).rejects.toThrow(AuthenticationError);
    });
  });
  
  describe('Mutation.createUser', () => {
    it('should create user successfully', async () => {
      const input = {
        name: 'Bob',
        email: 'bob@example.com',
        password: 'Password123',
      };
      
      const mockUser = { id: '2', ...input };
      User.findOne.mockResolvedValue(null);
      User.create.mockResolvedValue(mockUser);
      
      const result = await userResolvers.Mutation.createUser(null, { input }, {}, {});
      
      expect(result).toMatchObject({ id: '2', name: 'Bob' });
      expect(result.password).toBeUndefined(); // ไม่ return password
    });
    
    it('should throw ConflictError for duplicate email', async () => {
      User.findOne.mockResolvedValue({ id: '1' });
      
      await expect(
        userResolvers.Mutation.createUser(
          null,
          { input: { email: 'existing@example.com', name: 'Test', password: 'Pass123' } },
          {},
          {}
        )
      ).rejects.toThrow('Email already registered');
    });
  });
});
```

---

## 📌 Step 514: Integration Testing ด้วย executeOperation

```javascript
// test/integration/user.test.js
import { ApolloServer } from '@apollo/server';
import { typeDefs } from '../../src/schema.js';
import { resolvers } from '../../src/resolvers/index.js';
import { createContext } from '../../src/context.js';

function buildTestServer() {
  return new ApolloServer({ typeDefs, resolvers });
}

async function executeOperation(server, { query, variables, user }) {
  return server.executeOperation(
    { query, variables },
    {
      contextValue: {
        user: user || null,
        loaders: createTestLoaders(),
      },
    }
  );
}

describe('User API Integration', () => {
  let server;
  
  beforeAll(async () => {
    server = buildTestServer();
    await server.start();
    // Seed test database
    await seedTestDB();
  });
  
  afterAll(async () => {
    await server.stop();
    await clearTestDB();
  });
  
  describe('Query.me', () => {
    it('should return current user', async () => {
      const response = await executeOperation(server, {
        query: `query { me { id name email } }`,
        user: { sub: 'user_1', role: 'USER' },
      });
      
      expect(response.body.kind).toBe('single');
      const { data, errors } = response.body.singleResult;
      
      expect(errors).toBeUndefined();
      expect(data.me).toMatchObject({
        id: 'user_1',
        name: expect.any(String),
        email: expect.any(String),
      });
    });
    
    it('should return UNAUTHENTICATED when not logged in', async () => {
      const response = await executeOperation(server, {
        query: `query { me { id name } }`,
        user: null,
      });
      
      const { errors } = response.body.singleResult;
      expect(errors[0].extensions.code).toBe('UNAUTHENTICATED');
    });
  });
  
  describe('Mutation.createUser', () => {
    it('should create user and return auth payload', async () => {
      const response = await executeOperation(server, {
        query: `
          mutation CreateUser($input: CreateUserInput!) {
            createUser(input: $input) {
              user { id name email }
              accessToken
            }
          }
        `,
        variables: {
          input: {
            name: 'Test User',
            email: `test-${Date.now()}@example.com`,
            password: 'Password123!',
          },
        },
      });
      
      const { data, errors } = response.body.singleResult;
      
      expect(errors).toBeUndefined();
      expect(data.createUser.user).toMatchObject({
        name: 'Test User',
        email: expect.stringContaining('@example.com'),
      });
      expect(data.createUser.accessToken).toBeTruthy();
    });
  });
});
```

---

## 📌 Step 515: Test Factories

```javascript
// test/factories/user.factory.js
import { faker } from '@faker-js/faker/locale/th';

export function createUserInput(overrides = {}) {
  return {
    name: faker.person.fullName(),
    email: faker.internet.email().toLowerCase(),
    password: 'Password123!',
    ...overrides,
  };
}

export async function createTestUser(overrides = {}) {
  const input = createUserInput(overrides);
  const hashedPassword = await bcrypt.hash(input.password, 10);
  
  return User.create({
    ...input,
    password: hashedPassword,
    role: 'USER',
  });
}

export function createAdminUser(overrides = {}) {
  return createTestUser({ ...overrides, role: 'ADMIN' });
}

// test/factories/post.factory.js
export async function createTestPost(authorId, overrides = {}) {
  return Post.create({
    title: faker.lorem.sentence(),
    content: faker.lorem.paragraphs(3),
    authorId,
    status: 'PUBLISHED',
    ...overrides,
  });
}
```

---

## 📌 Step 516: Mock Strategies

```javascript
// Approach 1: Mock entire modules
jest.mock('../../src/models/user.model.js', () => ({
  User: {
    findById: jest.fn(),
    findOne: jest.fn(),
    create: jest.fn(),
    findAll: jest.fn(),
    count: jest.fn(),
  },
}));

// Approach 2: Mock individual functions
const mockUserFind = jest.spyOn(User, 'findById');
mockUserFind.mockResolvedValue({ id: '1', name: 'Alice' });

// Approach 3: In-memory DB (better for integration tests)
// ใช้ SQLite in-memory สำหรับ tests
const sequelize = new Sequelize({
  dialect: 'sqlite',
  storage: ':memory:',
  logging: false,
});

// Approach 4: Test containers (Docker)
import { PostgreSqlContainer } from '@testcontainers/postgresql';

let container;
let pool;

beforeAll(async () => {
  container = await new PostgreSqlContainer().start();
  pool = new Pool({
    connectionString: container.getConnectionUri(),
  });
  await runMigrations(pool);
}, 30000);

afterAll(async () => {
  await pool.end();
  await container.stop();
});
```

---

## 📌 Step 517: Snapshot Testing

```javascript
// test/snapshots/queries.test.js
describe('Query Snapshots', () => {
  it('should match product list snapshot', async () => {
    const response = await executeOperation(server, {
      query: `
        query {
          products(limit: 3) {
            id
            name
            price
            category { name }
          }
        }
      `,
    });
    
    // ครั้งแรก — สร้าง snapshot
    // ครั้งต่อไป — เปรียบเทียบกับ snapshot
    expect(response.body.singleResult.data).toMatchSnapshot();
  });
});
```

---

## 📌 Step 518: Auth Testing

```javascript
// test/auth/auth.test.js
describe('Authentication Flow', () => {
  let testUser;
  let accessToken;
  
  beforeAll(async () => {
    testUser = await createTestUser({ email: 'auth-test@example.com' });
  });
  
  it('should login successfully', async () => {
    const response = await executeOperation(server, {
      query: `
        mutation Login($input: LoginInput!) {
          login(input: $input) {
            accessToken
            refreshToken
            user { id email }
          }
        }
      `,
      variables: {
        input: { email: 'auth-test@example.com', password: 'Password123!' },
      },
    });
    
    const { data, errors } = response.body.singleResult;
    expect(errors).toBeUndefined();
    expect(data.login.accessToken).toBeTruthy();
    
    accessToken = data.login.accessToken;
  });
  
  it('should access protected query with token', async () => {
    const payload = jwt.decode(accessToken);
    
    const response = await executeOperation(server, {
      query: `query { me { id email } }`,
      user: payload,
    });
    
    expect(response.body.singleResult.errors).toBeUndefined();
    expect(response.body.singleResult.data.me.email).toBe('auth-test@example.com');
  });
  
  it('should reject invalid password', async () => {
    const response = await executeOperation(server, {
      query: `mutation { login(input: { email: "auth-test@example.com", password: "wrong" }) { accessToken } }`,
    });
    
    expect(response.body.singleResult.errors[0].extensions.code).toBe('BAD_USER_INPUT');
  });
});
```

---

## 📌 Step 519: สรุป Part 016

### เนื้อหาที่เรียนรู้

✅ Testing strategy (unit/integration/e2e)  
✅ Unit testing resolvers ด้วย Jest  
✅ Integration testing ด้วย executeOperation  
✅ Test factories ด้วย faker  
✅ Mock strategies  
✅ Snapshot testing  
✅ Auth flow testing  

### Homework

1. เขียน unit tests สำหรับ resolvers ทั้งหมด
2. เขียน integration tests สำหรับ auth flow
3. เพิ่ม test coverage ให้ถึง 80%

### ในส่วนถัดไป

➡️ **[Part 017](./part-017.md)** — Apollo Client & React Integration

---

*Part 016 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
