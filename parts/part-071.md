# Part 071 — GraphQL Testing Strategies 🧪

> **ระดับ:** Expert | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1741–1780

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Unit testing resolvers
- Integration testing กับ supertest
- Schema testing
- Snapshot testing
- Mocking dependencies
- Contract testing
- E2E testing กับ Playwright
- Test coverage

---

## 📌 Step 1741: Unit Testing Resolvers

```bash
npm install -D vitest @vitest/coverage-v8 @faker-js/faker
```

```typescript
// src/resolvers/__tests__/product.resolver.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { productResolvers } from '../product.resolver.js';
import type { AppContext } from '../../types.js';

// Mock Prisma
const mockPrisma = {
  product: {
    findUnique: vi.fn(),
    findMany: vi.fn(),
    create: vi.fn(),
    update: vi.fn(),
    delete: vi.fn(),
  },
};

// Mock context
const createMockContext = (overrides?: Partial<AppContext>): AppContext => ({
  user: { id: 'user-1', role: 'ADMIN', email: 'admin@test.com' },
  prisma: mockPrisma as unknown as typeof mockPrisma,
  loaders: {
    product: { load: vi.fn(), loadMany: vi.fn() },
    category: { load: vi.fn(), loadMany: vi.fn() },
  } as unknown as AppContext['loaders'],
  ...overrides,
});

describe('Product Resolvers', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });
  
  describe('Query.product', () => {
    it('returns product by id', async () => {
      const product = { id: 'prod-1', name: 'Test Product', price: 100 };
      mockPrisma.product.findUnique.mockResolvedValue(product);
      
      const result = await productResolvers.Query.product(
        {},
        { id: 'prod-1' },
        createMockContext(),
        {} as never
      );
      
      expect(result).toEqual(product);
      expect(mockPrisma.product.findUnique).toHaveBeenCalledWith({
        where: { id: 'prod-1' },
      });
    });
    
    it('returns null for non-existent product', async () => {
      mockPrisma.product.findUnique.mockResolvedValue(null);
      
      const result = await productResolvers.Query.product(
        {},
        { id: 'not-found' },
        createMockContext(),
        {} as never
      );
      
      expect(result).toBeNull();
    });
  });
  
  describe('Mutation.createProduct', () => {
    it('creates product successfully', async () => {
      const input = { name: 'New Product', price: 200, categoryId: 'cat-1', initialStock: 10 };
      const created = { id: 'new-prod', ...input };
      
      mockPrisma.product.create.mockResolvedValue(created);
      
      const result = await productResolvers.Mutation.createProduct(
        {},
        { input },
        createMockContext(),
        {} as never
      );
      
      expect(result).toEqual(created);
    });
    
    it('throws when not authenticated', async () => {
      await expect(
        productResolvers.Mutation.createProduct(
          {},
          { input: {} as never },
          createMockContext({ user: null as never }),
          {} as never
        )
      ).rejects.toThrow('Not authenticated');
    });
    
    it('throws when not admin', async () => {
      await expect(
        productResolvers.Mutation.createProduct(
          {},
          { input: {} as never },
          createMockContext({ user: { id: 'u1', role: 'CUSTOMER', email: 'u@test.com' } }),
          {} as never
        )
      ).rejects.toThrow();
    });
  });
});
```

---

## 📌 Step 1742: Integration Testing

```typescript
// src/__tests__/integration/products.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import request from 'supertest';
import { createApp } from '../../app.js';
import { resetTestDb, seedTestData } from '../helpers/db.js';

let app: Express.Application;

beforeAll(async () => {
  await resetTestDb();
  await seedTestData();
  app = createApp();
});

afterAll(async () => {
  await prisma.$disconnect();
});

describe('GraphQL API Integration', () => {
  it('fetches products list', async () => {
    const response = await request(app)
      .post('/graphql')
      .set('Content-Type', 'application/json')
      .send({
        query: `
          query {
            products(first: 5) {
              edges {
                node {
                  id
                  name
                  price
                }
              }
              totalCount
            }
          }
        `,
      });
    
    expect(response.status).toBe(200);
    expect(response.body.errors).toBeUndefined();
    expect(response.body.data.products.edges).toHaveLength(5);
    expect(response.body.data.products.totalCount).toBeGreaterThan(0);
  });
  
  it('creates product with auth', async () => {
    const token = generateTestToken({ id: 'admin-1', role: 'ADMIN' });
    
    const response = await request(app)
      .post('/graphql')
      .set('Authorization', `Bearer ${token}`)
      .send({
        query: `
          mutation CreateProduct($input: CreateProductInput!) {
            createProduct(input: $input) {
              id
              name
              price
            }
          }
        `,
        variables: {
          input: {
            name: 'Integration Test Product',
            price: 999,
            categoryId: 'cat-1',
            initialStock: 50,
            images: ['https://example.com/img.jpg'],
          },
        },
      });
    
    expect(response.status).toBe(200);
    expect(response.body.errors).toBeUndefined();
    expect(response.body.data.createProduct.name).toBe('Integration Test Product');
    expect(response.body.data.createProduct.id).toBeDefined();
  });
  
  it('returns error for unauthenticated mutation', async () => {
    const response = await request(app)
      .post('/graphql')
      .send({
        query: `mutation { createProduct(input: { name: "Test", price: 100 }) { id } }`,
      });
    
    expect(response.body.errors).toBeDefined();
    expect(response.body.errors[0].extensions.code).toBe('UNAUTHENTICATED');
  });
});
```

---

## 📌 Step 1743: Schema Testing

```typescript
// src/__tests__/schema.test.ts
import { describe, it, expect } from 'vitest';
import { buildSchema, parse, validate } from 'graphql';
import { printSchema } from 'graphql';
import { schema } from '../schema.js';

describe('GraphQL Schema', () => {
  it('is valid GraphQL schema', () => {
    // buildSchema will throw if invalid
    expect(() => printSchema(schema)).not.toThrow();
  });
  
  it('has required query fields', () => {
    const queryType = schema.getQueryType();
    expect(queryType).toBeDefined();
    
    const fields = queryType?.getFields();
    expect(fields).toHaveProperty('product');
    expect(fields).toHaveProperty('products');
    expect(fields).toHaveProperty('me');
  });
  
  it('validates correct query', () => {
    const query = parse(`
      query GetProduct($id: ID!) {
        product(id: $id) {
          id
          name
          price
        }
      }
    `);
    
    const errors = validate(schema, query);
    expect(errors).toHaveLength(0);
  });
  
  it('rejects invalid field', () => {
    const query = parse(`
      query {
        product(id: "1") {
          id
          nonExistentField
        }
      }
    `);
    
    const errors = validate(schema, query);
    expect(errors).toHaveLength(1);
    expect(errors[0].message).toContain('nonExistentField');
  });
  
  // Schema snapshot test
  it('matches schema snapshot', () => {
    expect(printSchema(schema)).toMatchSnapshot();
  });
});
```

---

## 📌 Step 1744: Mock Server สำหรับ Frontend Testing

```typescript
// mocks/graphql-server.ts
// สำหรับ frontend developers ทดสอบโดยไม่ต้องรัน backend จริง

import { createServer } from 'http';
import { createSchema, createYoga } from 'graphql-yoga';
import { addMocksToSchema } from '@graphql-tools/mock';
import { makeExecutableSchema } from '@graphql-tools/schema';
import { faker } from '@faker-js/faker';

const typeDefs = `
  type Product { id: ID! name: String! price: Float! }
  type Query { product(id: ID!): Product products: [Product!]! }
`;

const schema = makeExecutableSchema({ typeDefs });

const mockedSchema = addMocksToSchema({
  schema,
  mocks: {
    ID: () => faker.string.uuid(),
    String: () => faker.lorem.words(3),
    Float: () => faker.number.float({ min: 10, max: 10000, fractionDigits: 2 }),
    
    Product: () => ({
      id: faker.string.uuid(),
      name: faker.commerce.productName(),
      price: faker.number.float({ min: 10, max: 10000, fractionDigits: 2 }),
    }),
  },
  
  resolvers: {
    Query: {
      products: () => Array.from({ length: 10 }, () => ({
        id: faker.string.uuid(),
        name: faker.commerce.productName(),
        price: faker.number.float({ min: 10, max: 10000 }),
      })),
    },
  },
});

const yoga = createYoga({ schema: mockedSchema });
const server = createServer(yoga);
server.listen(4001, () => console.log('Mock server running on :4001'));
```

---

## 📌 Step 1745: สรุป Part 071

### เนื้อหาที่เรียนรู้

✅ Unit testing resolvers  
✅ Integration testing ด้วย supertest  
✅ Schema validation tests  
✅ Snapshot testing  
✅ Mock server สำหรับ frontend  

### Test Pyramid สำหรับ GraphQL

```
          /\
         /E2E\        ← ทดสอบ flow สมบูรณ์ (Playwright)
        /------\
       /Integr. \     ← Test API endpoints, DB interactions
      /----------\
     / Unit Tests \   ← Resolvers, business logic
    /--------------\

ไม่ต้อง test:
- Third-party libraries
- Simple getters/setters
- Generated code (Prisma types)
```

### ในส่วนถัดไป

➡️ **[Part 072](./part-072.md)** — GraphQL at Scale: Architecture Patterns

---

*Part 071 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
