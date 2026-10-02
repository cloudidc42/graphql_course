# Part 093 — Open Source Contribution to GraphQL Ecosystem 🌐

> **ระดับ:** World-Class | **เวลาเรียน:** 70 นาที | **ขั้นตอนที่:** 2661–2705

---

## 🎯 สิ่งที่จะได้เรียนรู้

- GraphQL ecosystem map
- How to contribute to graphql-js
- สร้าง custom Envelop plugin
- Publish npm packages
- RFC process (graphql-wg)
- Community resources
- Building credibility ใน OSS

---

## 📌 Step 2661: GraphQL Ecosystem Map

```
Core:
  graphql-js (facebook/graphql-js)     ← Reference implementation
  graphql-spec (graphql/graphql-spec)  ← Specification
  
Servers:
  Apollo Server, Yoga, Mercurius, Helix
  
Clients:
  Apollo Client, urql, Relay
  
Schema Tools:
  @graphql-tools/*, graphql-code-generator
  
Federation:
  Apollo Federation, Apollo Router
  
Testing:
  jest-graphql-matchers, graphql-tester
  
Monitoring:
  GraphQL Hive, Apollo Studio
  
Community:
  graphql-wg (Working Group meetings)
  This Week in GraphQL (newsletter)
  The Guild (maintainers of many OSS packages)
```

---

## 📌 Step 2662: สร้าง Custom Envelop Plugin สำหรับ Publish

```typescript
// packages/envelop-plugin-audit-log/src/index.ts
// Plugin ที่ log ทุก mutation พร้อม user info

import { Plugin, isAsyncIterable } from '@envelop/core';
import type { ExecutionArgs, ExecutionResult } from 'graphql';

export interface AuditLogOptions {
  /**
   * กำหนดว่า fields ไหนเป็น sensitive (จะ mask ใน logs)
   */
  sensitiveFields?: string[];
  
  /**
   * Function สำหรับ write log
   */
  logger: (entry: AuditLogEntry) => void | Promise<void>;
  
  /**
   * Log mutations เท่านั้น (default: true)
   */
  mutationsOnly?: boolean;
}

export interface AuditLogEntry {
  timestamp: string;
  operationName: string | null;
  operationType: string;
  userId: string | null;
  ip: string | null;
  variables: Record<string, unknown>;
  duration: number;
  success: boolean;
  errors?: string[];
}

export function useAuditLog(options: AuditLogOptions): Plugin {
  const {
    sensitiveFields = ['password', 'token', 'creditCard'],
    mutationsOnly = true,
    logger,
  } = options;
  
  return {
    onExecute({ args }) {
      const start = Date.now();
      const { operationName, document, contextValue, variableValues } = args;
      
      // Detect operation type
      const operationType = detectOperationType(document);
      
      if (mutationsOnly && operationType !== 'mutation') {
        return;
      }
      
      return {
        onExecuteDone({ result }) {
          // Handle both single result and streaming
          const singleResult = isAsyncIterable(result) ? null : result as ExecutionResult;
          
          void logger({
            timestamp: new Date().toISOString(),
            operationName: operationName ?? null,
            operationType,
            userId: (contextValue as Record<string, unknown>)?.user?.id as string ?? null,
            ip: (contextValue as Record<string, unknown>)?.ip as string ?? null,
            variables: maskSensitiveFields(variableValues ?? {}, sensitiveFields),
            duration: Date.now() - start,
            success: !singleResult?.errors?.length,
            errors: singleResult?.errors?.map(e => e.message),
          });
        },
      };
    },
  };
}

function maskSensitiveFields(
  vars: Record<string, unknown>,
  sensitive: string[]
): Record<string, unknown> {
  return Object.fromEntries(
    Object.entries(vars).map(([k, v]) => [
      k,
      sensitive.includes(k) ? '[REDACTED]' : v,
    ])
  );
}

function detectOperationType(document: unknown): string {
  // Simplified: check first definition
  const def = (document as Record<string, unknown>)?.definitions?.[0] as Record<string, unknown>;
  return (def?.operation as string) ?? 'query';
}
```

---

## 📌 Step 2663: Package.json สำหรับ npm Publish

```json
{
  "name": "@yourname/envelop-plugin-audit-log",
  "version": "1.0.0",
  "description": "Envelop plugin for GraphQL operation audit logging",
  "license": "MIT",
  "main": "dist/index.js",
  "module": "dist/index.mjs",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "require": "./dist/index.js",
      "import": "./dist/index.mjs",
      "types": "./dist/index.d.ts"
    }
  },
  "files": ["dist", "README.md"],
  "keywords": ["graphql", "envelop", "plugin", "audit", "logging"],
  "peerDependencies": {
    "@envelop/core": ">=4.0.0",
    "graphql": ">=16.0.0"
  },
  "devDependencies": {
    "@envelop/core": "^4.0.0",
    "graphql": "^16.0.0",
    "tsup": "^8.0.0",
    "typescript": "^5.0.0",
    "vitest": "^2.0.0"
  },
  "scripts": {
    "build": "tsup src/index.ts --format cjs,esm --dts",
    "test": "vitest",
    "prepublishOnly": "npm run build && npm test"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/yourname/envelop-plugin-audit-log"
  }
}
```

---

## 📌 Step 2664: Tests สำหรับ Plugin

```typescript
// tests/audit-log.test.ts
import { describe, it, expect, vi } from 'vitest';
import { envelop, useSchema } from '@envelop/core';
import { makeExecutableSchema } from '@graphql-tools/schema';
import { useAuditLog } from '../src';

const schema = makeExecutableSchema({
  typeDefs: `
    type Query { hello: String }
    type Mutation { createUser(email: String!, password: String!): Boolean }
  `,
  resolvers: {
    Query: { hello: () => 'world' },
    Mutation: { createUser: () => true },
  },
});

describe('useAuditLog', () => {
  it('logs mutations', async () => {
    const logFn = vi.fn();
    
    const { execute, parse, validate } = envelop({
      plugins: [
        useSchema(schema),
        useAuditLog({ logger: logFn }),
      ],
    })({});
    
    const document = parse(`mutation { createUser(email: "test@test.com", password: "secret") }`);
    await execute({ schema, document, contextValue: { user: { id: 'user-1' } } });
    
    expect(logFn).toHaveBeenCalledOnce();
    const entry = logFn.mock.calls[0][0];
    expect(entry.operationType).toBe('mutation');
    expect(entry.userId).toBe('user-1');
  });
  
  it('masks sensitive fields', async () => {
    const logFn = vi.fn();
    
    const { execute, parse } = envelop({
      plugins: [
        useSchema(schema),
        useAuditLog({ logger: logFn, sensitiveFields: ['password'] }),
      ],
    })({});
    
    const document = parse(`mutation CreateUser($email: String!, $password: String!) {
      createUser(email: $email, password: $password)
    }`);
    
    await execute({
      schema,
      document,
      variableValues: { email: 'a@b.com', password: 'supersecret' },
    });
    
    const entry = logFn.mock.calls[0][0];
    expect(entry.variables.password).toBe('[REDACTED]');
    expect(entry.variables.email).toBe('a@b.com');
  });
  
  it('does not log queries when mutationsOnly=true', async () => {
    const logFn = vi.fn();
    
    const { execute, parse } = envelop({
      plugins: [
        useSchema(schema),
        useAuditLog({ logger: logFn, mutationsOnly: true }),
      ],
    })({});
    
    const document = parse(`query { hello }`);
    await execute({ schema, document });
    
    expect(logFn).not.toHaveBeenCalled();
  });
});
```

---

## 📌 Step 2665: Contributing to graphql-js

```bash
# Fork + clone
git clone https://github.com/yourname/graphql-js
cd graphql-js

# Setup
npm install
npm test  # ตรวจสอบว่า test ผ่านก่อน

# สร้าง feature branch
git checkout -b feature/improve-error-message

# Understand test structure
# src/ - source code
# src/__tests__/ - tests
# website/ - documentation

# Run specific tests
npm test -- --testPathPattern "validate"

# Build
npm run build

# Check types
npm run check
```

```
Contributing Guidelines:
1. ตรวจสอบ existing issues/PRs ก่อน
2. เปิด issue ก่อน implement (สำหรับ features ใหญ่)
3. Follow GraphQL spec (RFC process สำหรับ spec changes)
4. เขียน tests ทุก change
5. Update CHANGELOG.md
6. Sign the CLA

Good First Issues (งานเล็กที่ เหมาะสำหรับ เริ่มต้น):
- Error message improvements
- TypeScript types enhancement
- Documentation fixes
- Test coverage improvement
```

---

## 📌 Step 2666: สรุป Part 093

### เนื้อหาที่เรียนรู้

✅ GraphQL ecosystem overview  
✅ สร้าง Envelop plugin  
✅ Package setup สำหรับ npm  
✅ Tests สำหรับ plugins  
✅ Contributing to graphql-js  

### Building OSS Credibility

```
Path to OSS Recognition:
1. ใช้ libraries → รายงาน bugs (issues)
2. Fix simple bugs → PRs
3. สร้าง utilities/plugins → ช่วย community
4. Write blog posts/talks → build recognition
5. Become maintainer → invited after sustained contributions

Community Involvement:
□ GraphQL Working Group (monthly meetings, open to all)
□ The Guild Discord
□ GraphQL Community Forum
□ This Week in GraphQL newsletter

Ways to Contribute:
□ Documentation
□ Bug fixes
□ New features (discuss first)
□ Performance improvements
□ TypeScript improvements
□ Test coverage
```

### ในส่วนถัดไป

➡️ **[Part 094](./part-094.md)** — GraphQL Interview Preparation

---

*Part 093 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
