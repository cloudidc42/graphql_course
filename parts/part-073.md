# Part 073 — GraphQL Documentation & Developer Experience 📚

> **ระดับ:** Expert | **เวลาเรียน:** 70 นาที | **ขั้นตอนที่:** 1821–1860

---

## 🎯 สิ่งที่จะได้เรียนรู้

- GraphQL Playground & GraphiQL setup
- Auto-generated documentation
- Code generation กับ GraphQL Codegen
- Apollo Studio
- Docusaurus integration
- GraphQL Hive
- SDK generation
- Developer Portal

---

## 📌 Step 1821: GraphQL IDE Integrations

```typescript
// Apollo Sandbox (production-safe GraphQL IDE)
// เปิดที่ http://localhost:4000/graphql → redirect ไป Apollo Sandbox

import { ApolloServerPluginLandingPageLocalDefault } from '@apollo/server/plugin/landingPage/default';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    process.env.NODE_ENV === 'production'
      // ใน production: redirect ไป Apollo Studio
      ? ApolloServerPluginLandingPageProductionDefault({
          graphRef: 'my-graph@prod',
          footer: false,
        })
      // ใน development: local sandbox
      : ApolloServerPluginLandingPageLocalDefault({
          footer: false,
          embed: true,
          includeCookies: true,   // สำหรับ cookie-based auth
        }),
  ],
});

// GraphiQL standalone (ถ้าต้องการ customize)
app.get('/graphiql', (req, res) => {
  res.send(`
<!DOCTYPE html>
<html>
  <head>
    <title>GraphQL Explorer</title>
    <style>body { height: 100vh; margin: 0; overflow: hidden; }</style>
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <link rel="stylesheet" href="https://unpkg.com/graphiql/graphiql.min.css" />
  </head>
  <body>
    <div id="graphiql" style="height: 100vh;"></div>
    <script src="https://unpkg.com/graphiql/graphiql.min.js"></script>
    <script>
      const root = ReactDOM.createRoot(document.getElementById('graphiql'));
      root.render(
        React.createElement(GraphiQL, {
          fetcher: GraphiQL.createFetcher({
            url: '/graphql',
            headers: { 'Authorization': 'Bearer ' + localStorage.getItem('token') || '' },
          }),
          defaultQuery: '{ me { id name } }',
        })
      );
    </script>
  </body>
</html>
  `);
});
```

---

## 📌 Step 1822: GraphQL Code Generator

```bash
npm install -D @graphql-codegen/cli \
  @graphql-codegen/typescript \
  @graphql-codegen/typescript-resolvers \
  @graphql-codegen/typescript-operations \
  @graphql-codegen/typescript-react-apollo
```

```yaml
# codegen.yml
schema: http://localhost:4000/graphql

generates:
  # Server-side: resolver types
  src/generated/types.ts:
    plugins:
      - typescript
      - typescript-resolvers
    config:
      contextType: ../types#AppContext
      mappers:
        User: ../models#UserModel
        Product: ../models#ProductModel
        Order: ../models#OrderModel
      scalars:
        DateTime: Date
        JSON: Record<string, unknown>
        Money: number
  
  # Client-side: operation types
  src/generated/graphql.ts:
    documents:
      - "src/**/*.graphql"
      - "src/**/*.tsx"
    plugins:
      - typescript
      - typescript-operations
      - typescript-react-apollo
    config:
      withHooks: true
      withComponent: false
      withHOC: false
      scalars:
        DateTime: string
        JSON: Record<string, unknown>
```

```typescript
// ใช้ generated types
// BEFORE: ไม่มี type safety
const { data } = useQuery(GET_PRODUCT, { variables: { id } });
console.log(data.product.name);  // ❌ no type check

// AFTER: fully typed
import { useGetProductQuery } from './generated/graphql';

const { data } = useGetProductQuery({ variables: { id } });
console.log(data?.product?.name);  // ✅ typed, auto-complete
```

---

## 📌 Step 1823: Schema Documentation กับ SDL Comments

```graphql
"""
# GraphQL API Documentation

E-commerce platform API ที่ครอบคลุมทุกฟีเจอร์

## Authentication
ส่ง JWT token ใน Authorization header:
\`\`\`
Authorization: Bearer <token>
\`\`\`

## Pagination
ทุก list ที่ return จาก API ใช้ Relay cursor pagination:
\`\`\`graphql
query {
  products(first: 10, after: "cursor") {
    edges { node { id } cursor }
    pageInfo { hasNextPage endCursor }
  }
}
\`\`\`

## Error Handling
\`\`\`json
{
  "errors": [{
    "message": "User-friendly message",
    "extensions": {
      "code": "NOT_FOUND",
      "field": "productId"
    }
  }]
}
\`\`\`
"""
schema {
  query: Query
  mutation: Mutation
  subscription: Subscription
}

"""
Root query type.

Available operations:
- Product catalog browsing
- User management
- Order management
- Search
"""
type Query {
  """
  ดึงข้อมูล product โดย ID

  **ต้องการ auth:** ไม่

  **ตัวอย่างใช้งาน:**
  \`\`\`graphql
  query GetProduct($id: ID!) {
    product(id: $id) {
      id
      name
      price
    }
  }
  \`\`\`

  **Errors:**
  - `NOT_FOUND`: product ไม่มีอยู่
  """
  product(
    "ID ของ product ที่ต้องการ"
    id: ID!
  ): Product

  """
  ดึง product list พร้อม filtering และ pagination

  **ต้องการ auth:** ไม่

  **Rate limit:** 100 requests/minute
  """
  products(
    "จำนวน records ต่อหน้า (max: 100)"
    first: Int
    "Cursor สำหรับ pagination"
    after: String
    "Filter options"
    filter: ProductFilterInput
    "Sort options"
    orderBy: ProductOrderByInput
  ): ProductConnection!
}
```

---

## 📌 Step 1824: GraphQL Hive (Schema Registry)

```bash
# GraphQL Hive: open-source schema registry
# หรือ cloud version: app.graphql-hive.com

npm install @graphql-hive/client
```

```typescript
// src/server.ts
import { useHive } from '@graphql-hive/client';
import { createYoga } from 'graphql-yoga';

const yoga = createYoga({
  schema,
  plugins: [
    useHive({
      enabled: true,
      debug: false,
      token: process.env.HIVE_TOKEN!,
      
      // Usage reporting
      usage: {
        enabled: true,
        sampleRate: 0.1,  // 10% sample rate
        
        // ไม่ report ถ้า operation ล้มเหลว
        exclude: [
          (args) => args.operationName === 'IntrospectionQuery',
        ],
      },
      
      // Schema reporting
      reporting: {
        enabled: true,
        author: `CI/CD`,
        commit: process.env.GIT_COMMIT ?? 'unknown',
      },
    }),
  ],
});
```

---

## 📌 Step 1825: Docusaurus Integration

```bash
npx create-docusaurus@latest graphql-docs classic --typescript
cd graphql-docs
npm install docusaurus-graphql-plugin
```

```javascript
// docusaurus.config.js
const config = {
  plugins: [
    [
      'docusaurus-graphql-plugin',
      {
        schema: 'http://localhost:4000/graphql',
        outputDir: 'docs/api',
        
        // Auto-generate doc pages จาก schema
        generateDocumentation: {
          queries: true,
          mutations: true,
          subscriptions: true,
          types: true,
        },
      },
    ],
  ],
};
```

---

## 📌 Step 1826: สรุป Part 073

### เนื้อหาที่เรียนรู้

✅ Apollo Sandbox / GraphiQL  
✅ GraphQL Codegen setup  
✅ SDL documentation  
✅ GraphQL Hive  
✅ Docusaurus integration  

### DX Checklist

```
For API Consumers:
□ Interactive playground (Apollo Sandbox)
□ Schema documentation (descriptions on every type/field)
□ Code examples สำหรับ common operations
□ Changelog สำหรับ breaking changes
□ SDK / client library ที่ typed

For API Developers:
□ Codegen: types ตรงกันระหว่าง schema และ resolvers
□ Schema registry (Hive / Apollo Studio)
□ Breaking change detection
□ Usage analytics (ใช้ field ไหนบ้าง)
```

### ในส่วนถัดไป

➡️ **[Part 074](./part-074.md)** — GraphQL with gRPC & Microservices

---

*Part 073 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
