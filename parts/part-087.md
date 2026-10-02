# Part 087 — GraphQL Contract Testing 📋

> **ระดับ:** Expert | **เวลาเรียน:** 75 นาที | **ขั้นตอนที่:** 2391–2435

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Contract testing คืออะไร
- Consumer-Driven Contract Testing
- Pact สำหรับ GraphQL
- Schema compatibility tests
- Operation compatibility
- CI/CD integration
- Provider verification
- Breaking change alerts

---

## 📌 Step 2391: Contract Testing คืออะไร

```
Contract Testing:
- ทดสอบว่า consumer (client) กับ provider (server) ตกลงกันได้
- ทดแทน E2E tests บางส่วน
- เร็วกว่า integration tests

ปัญหาที่แก้:
Consumer (React app) เรียก query:
  query GetProduct($id: ID!) {
    product(id: $id) {
      id
      name
      price     ← consumer ต้องการ field นี้
      stock     ← consumer ต้องการ field นี้
    }
  }

ถ้า provider ลบ/เปลี่ยน 'stock' โดยไม่รู้ว่า consumer ใช้อยู่
→ Contract test จะ fail ก่อน deploy

Types:
1. Consumer-Driven: consumer เขียน contract, provider verify
2. Schema-Driven: ตรวจสอบ schema compatibility
3. Operation-Based: ตรวจสอบว่า operations ยังทำงานได้
```

---

## 📌 Step 2392: Schema Compatibility Test

```typescript
// tests/contract/schema-compatibility.test.ts
import { buildSchema, findBreakingChanges, findDangerousChanges } from 'graphql';
import { readFileSync } from 'fs';

describe('Schema Compatibility', () => {
  let productionSchema: GraphQLSchema;
  let newSchema: GraphQLSchema;
  
  beforeAll(async () => {
    // Production schema: ดึงจาก registry หรือ file
    const productionSDL = readFileSync('./contracts/production-schema.graphql', 'utf-8');
    productionSchema = buildSchema(productionSDL);
    
    // New schema: ที่กำลังจะ deploy
    const newSDL = readFileSync('./schema.graphql', 'utf-8');
    newSchema = buildSchema(newSDL);
  });
  
  it('should not have breaking changes', () => {
    const breakingChanges = findBreakingChanges(productionSchema, newSchema);
    
    // ถ้า breaking changes ที่ approved ไม่ fail
    const approvedBreaking = loadApprovedBreakingChanges();
    const unapprovedBreaking = breakingChanges.filter(
      c => !approvedBreaking.includes(c.description)
    );
    
    expect(unapprovedBreaking).toHaveLength(0);
  });
  
  it('should document dangerous changes', () => {
    const dangerous = findDangerousChanges(productionSchema, newSchema);
    
    if (dangerous.length > 0) {
      console.warn('Dangerous changes detected:');
      dangerous.forEach(c => console.warn(`  ${c.type}: ${c.description}`));
    }
    
    // Dangerous changes ไม่ fail แต่ต้อง document
    // (เพิ่ม enum values อาจทำให้ exhaustive switch ใน clients พัง)
  });
  
  it('should maintain all registered client operations', async () => {
    // โหลด operations ที่ clients ใช้จาก registry
    const clientOperations = loadClientOperationRegistry();
    
    for (const { clientName, operationName, query } of clientOperations) {
      const errors = validate(newSchema, parse(query));
      expect(errors, `${clientName}:${operationName} is invalid`).toHaveLength(0);
    }
  });
});
```

---

## 📌 Step 2393: Client Operation Registry

```typescript
// contracts/client-registry.ts
// บันทึกว่า client แต่ละตัวใช้ operation ไหน

interface RegisteredOperation {
  clientName: string;
  clientVersion: string;
  operationName: string;
  query: string;
  registeredAt: string;
  hash: string;
}

// clients/web-app/operations.ts
// Client เก็บ operations ที่ตัวเองใช้
export const webAppOperations: RegisteredOperation[] = [
  {
    clientName: 'web-app',
    clientVersion: '3.x',
    operationName: 'GetProductList',
    query: /* GraphQL */`
      query GetProductList($filter: ProductFilter) {
        products(filter: $filter) {
          id
          name
          price
          stock
          thumbnailUrl
          category { id name }
        }
      }
    `,
    registeredAt: '2025-01-01',
    hash: 'abc123',
  },
  
  {
    clientName: 'web-app',
    clientVersion: '3.x',
    operationName: 'CreateOrder',
    query: /* GraphQL */`
      mutation CreateOrder($input: CreateOrderInput!) {
        createOrder(input: $input) {
          id
          status
          totalAmount
          estimatedDelivery
        }
      }
    `,
    registeredAt: '2025-01-01',
    hash: 'def456',
  },
];

// Generate contract file สำหรับ server
// npm run generate-contracts
async function generateContracts() {
  const allOperations = [
    ...webAppOperations,
    ...mobileAppOperations,
    ...partnerApiOperations,
  ];
  
  writeFileSync(
    './contracts/client-operations.json',
    JSON.stringify(allOperations, null, 2)
  );
}
```

---

## 📌 Step 2394: Provider Verification Tests

```typescript
// tests/contract/provider.test.ts
// Server ตรวจสอบว่า operations ทุกอย่างยังทำงานได้

import { createTestServer } from '../helpers/test-server';
import { loadClientOperationRegistry } from './client-registry';

describe('Provider Contract Verification', () => {
  let testServer: TestServer;
  
  beforeAll(async () => {
    testServer = await createTestServer();
  });
  
  afterAll(() => testServer.stop());
  
  const clientOperations = loadClientOperationRegistry();
  
  for (const op of clientOperations) {
    it(`${op.clientName}: ${op.operationName} returns expected shape`, async () => {
      const result = await testServer.executeOperation({
        query: op.query,
        variables: op.sampleVariables ?? {},
      });
      
      // ไม่ควรมี errors
      expect(result.errors).toBeUndefined();
      
      // Shape ต้องตรงกับ expected
      if (op.expectedShape) {
        expect(result.data).toMatchObject(op.expectedShape);
      }
      
      // ต้องมี required fields ทั้งหมด
      if (op.requiredFields) {
        for (const field of op.requiredFields) {
          expect(result.data).toHaveProperty(field);
        }
      }
    });
  }
});
```

---

## 📌 Step 2395: CI/CD Contract Verification

```yaml
# .github/workflows/contract-tests.yml
name: Contract Tests

on:
  push:
    paths:
      - 'schema.graphql'
      - 'src/**/*.ts'
  pull_request:

jobs:
  schema-compatibility:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Download production schema
        run: |
          # ดึง schema จาก Hive/Studio registry
          npx hive schema:fetch \
            --registry ${{ vars.HIVE_ENDPOINT }} \
            --token ${{ secrets.HIVE_TOKEN }} \
            > contracts/production-schema.graphql
      
      - name: Check breaking changes
        run: npx ts-node scripts/check-breaking-changes.ts
        env:
          ALLOW_BREAKING_CHANGES: ${{ vars.ALLOW_BREAKING_CHANGES }}
      
      - name: Run contract tests
        run: npm run test:contracts
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
      
      - name: Publish schema (on main only)
        if: github.ref == 'refs/heads/main'
        run: |
          npx hive schema:publish \
            --registry ${{ vars.HIVE_ENDPOINT }} \
            --token ${{ secrets.HIVE_TOKEN }} \
            --author "${{ github.actor }}" \
            --commit "${{ github.sha }}" \
            schema.graphql

  client-compatibility:
    runs-on: ubuntu-latest
    needs: schema-compatibility
    strategy:
      matrix:
        client: [web-app, mobile-app, partner-api]
    steps:
      - name: Verify ${{ matrix.client }} operations
        run: npm run test:contracts -- --client=${{ matrix.client }}
```

---

## 📌 Step 2396: สรุป Part 087

### เนื้อหาที่เรียนรู้

✅ Contract testing concepts  
✅ Schema compatibility tests  
✅ Client operation registry  
✅ Provider verification  
✅ CI/CD integration  

### Contract Testing Strategy

```
Level 1: Schema Compatibility
→ findBreakingChanges() ใน CI
→ ทุก PR ที่แก้ schema

Level 2: Operation Registry
→ Clients ลงทะเบียน operations
→ Server ตรวจว่า operations ยังใช้ได้

Level 3: Provider Verification
→ Execute operations ด้วย test data
→ ตรวจสอบ response shape

Level 4: Consumer Contract Tests
→ Pact หรือ similar
→ Real contract files ที่ consumer สร้าง

ลงทุนระยะยาว:
□ Client registry → รู้ว่า field ไหน safe to remove
□ Automated checks → CI fail ก่อน merge
□ Schema changelog → communicate changes
```

### ในส่วนถัดไป

➡️ **[Part 088](./part-088.md)** — GraphQL Gateway Patterns

---

*Part 087 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
