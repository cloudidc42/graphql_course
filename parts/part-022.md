# Part 022 — Schema Stitching & Integration Patterns 🔗

> **ระดับ:** Advanced | **เวลาเรียน:** 120 นาที | **ขั้นตอนที่:** 741–780

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Schema Stitching คืออะไร
- เปรียบเทียบ Federation vs Stitching
- mergeSchemas
- Remote schema wrapping
- Type merging
- Transforms
- REST API wrapping
- Third-party API integration

---

## 📌 Step 741: Schema Stitching คืออะไร

```
Schema Stitching = รวม GraphQL schemas หลายอันเป็นอันเดียว

ใช้เมื่อ:
- ต้องการ combine schemas ที่มีอยู่แล้ว
- Wrap REST APIs เป็น GraphQL
- Integrate third-party GraphQL APIs
- ไม่ต้องการ deploy Federation infrastructure
```

---

## 📌 Step 742: Federation vs Stitching

```
Feature            Federation     Schema Stitching
─────────────────────────────────────────────────
Architecture        Distributed   Gateway-controlled
Query planning      Distributed   Centralized
Schema ownership    Subgraphs     Gateway
Type sharing        @key entities Type merging
Performance         Better        Good
Complexity          Higher        Lower
Use case            Microservices Simpler setups
```

---

## 📌 Step 743: mergeSchemas

```javascript
// npm install @graphql-tools/schema @graphql-tools/stitch
import { makeExecutableSchema } from '@graphql-tools/schema';
import { stitchSchemas } from '@graphql-tools/stitch';

// Schema 1: Users
const usersSchema = makeExecutableSchema({
  typeDefs: `
    type User {
      id: ID!
      name: String!
      email: String!
    }
    type Query {
      user(id: ID!): User
      users: [User!]!
    }
  `,
  resolvers: userResolvers,
});

// Schema 2: Products
const productsSchema = makeExecutableSchema({
  typeDefs: `
    type Product {
      id: ID!
      name: String!
      price: Float!
      sellerId: ID!
    }
    type Query {
      product(id: ID!): Product
      products: [Product!]!
    }
  `,
  resolvers: productResolvers,
});

// Stitch schemas together
const stitchedSchema = stitchSchemas({
  subschemas: [
    { schema: usersSchema },
    { schema: productsSchema },
  ],
  
  // Type extensions — เชื่อม types ระหว่าง schemas
  typeDefs: `
    extend type Product {
      seller: User!
    }
    
    extend type User {
      products: [Product!]!
    }
  `,
  
  resolvers: {
    Product: {
      seller: {
        selectionSet: '{ sellerId }',
        resolve(product, _, context, info) {
          return delegateToSchema({
            schema: usersSchema,
            operation: 'query',
            fieldName: 'user',
            args: { id: product.sellerId },
            context,
            info,
          });
        },
      },
    },
    
    User: {
      products: {
        selectionSet: '{ id }',
        resolve(user, _, context, info) {
          return delegateToSchema({
            schema: productsSchema,
            operation: 'query',
            fieldName: 'productsBySeller',
            args: { sellerId: user.id },
            context,
            info,
          });
        },
      },
    },
  },
});
```

---

## 📌 Step 744: Remote Schema Wrapping

```javascript
// npm install @graphql-tools/executor-http
import { buildHTTPExecutor } from '@graphql-tools/executor-http';
import { schemaFromExecutor, wrapSchema } from '@graphql-tools/wrap';

// Remote schema — ดึง schema จาก URL
const usersExecutor = buildHTTPExecutor({
  endpoint: 'http://users-service:4001/graphql',
  headers: { 'x-api-key': process.env.USERS_SERVICE_KEY },
});

const usersSchema = await schemaFromExecutor(usersExecutor);
const remoteUsersSchema = wrapSchema({ schema: usersSchema, executor: usersExecutor });

// Stitch remote + local schemas
const stitchedSchema = stitchSchemas({
  subschemas: [
    { schema: remoteUsersSchema },
    { schema: localProductsSchema },
  ],
});
```

---

## 📌 Step 745: REST API Wrapping

```javascript
// npm install @graphql-tools/wrap
// Wrap REST API เป็น GraphQL schema

const restTypeDefs = `
  type WeatherData {
    city: String!
    temperature: Float!
    humidity: Float!
    description: String!
    icon: String!
  }
  
  type Query {
    weather(city: String!): WeatherData
    weatherByCoords(lat: Float!, lon: Float!): WeatherData
  }
`;

const restResolvers = {
  Query: {
    weather: async (_, { city }) => {
      const response = await fetch(
        `https://api.openweathermap.org/data/2.5/weather?q=${city}&appid=${process.env.WEATHER_API_KEY}&units=metric`
      );
      const data = await response.json();
      
      return {
        city: data.name,
        temperature: data.main.temp,
        humidity: data.main.humidity,
        description: data.weather[0].description,
        icon: `https://openweathermap.org/img/wn/${data.weather[0].icon}@2x.png`,
      };
    },
    
    weatherByCoords: async (_, { lat, lon }) => {
      const response = await fetch(
        `https://api.openweathermap.org/data/2.5/weather?lat=${lat}&lon=${lon}&appid=${process.env.WEATHER_API_KEY}&units=metric`
      );
      const data = await response.json();
      
      return {
        city: data.name,
        temperature: data.main.temp,
        humidity: data.main.humidity,
        description: data.weather[0].description,
        icon: `https://openweathermap.org/img/wn/${data.weather[0].icon}@2x.png`,
      };
    },
  },
};

const weatherSchema = makeExecutableSchema({
  typeDefs: restTypeDefs,
  resolvers: restResolvers,
});
```

---

## 📌 Step 746: Schema Transforms

```javascript
import { 
  FilterObjectFields, 
  RenameTypes, 
  RenameRootFields,
  TransformObjectFields,
} from '@graphql-tools/wrap';

// Transform เพื่อ hide internal fields
const filteredUsersSchema = wrapSchema({
  schema: usersSchema,
  transforms: [
    // ซ่อน field ที่ sensitive
    new FilterObjectFields((typeName, fieldName) => {
      if (typeName === 'User') {
        return !['password', 'secret', 'internalNotes'].includes(fieldName);
      }
      return true;
    }),
    
    // Rename types (ป้องกัน conflicts)
    new RenameTypes((name) => {
      if (name === 'User') return 'ExternalUser';
      return name;
    }),
    
    // Transform field values
    new TransformObjectFields((typeName, fieldName, fieldConfig) => {
      if (typeName === 'User' && fieldName === 'email') {
        return {
          ...fieldConfig,
          resolve: (user) => maskEmail(user.email),
        };
      }
      return fieldConfig;
    }),
  ],
});
```

---

## 📌 Step 747: Type Merging

```javascript
// stitchSchemas ด้วย Type Merging
const stitchedSchema = stitchSchemas({
  subschemas: [
    {
      schema: usersSchema,
      merge: {
        // Tell stitching how to fetch User by reference
        User: {
          fieldName: 'user',
          selectionSet: '{ id }',
          args: (partialUser) => ({ id: partialUser.id }),
        },
      },
    },
    {
      schema: productsSchema,
      merge: {
        User: {
          fieldName: 'userById',
          selectionSet: '{ id }',
          args: (partialUser) => ({ id: partialUser.id }),
        },
      },
    },
  ],
});
```

---

## 📌 Step 748: สรุป Part 022

### เนื้อหาที่เรียนรู้

✅ Schema Stitching vs Federation  
✅ mergeSchemas  
✅ Remote schema wrapping  
✅ REST API → GraphQL wrapping  
✅ Schema transforms  
✅ Type merging  

### Homework

1. Wrap ข้อมูลจาก public REST API (เช่น GitHub API) เป็น GraphQL
2. Stitch 2 local schemas เข้าด้วยกัน
3. เพิ่ม FilterObjectFields transform

### ในส่วนถัดไป

➡️ **[Part 023](./part-023.md)** — PostgreSQL + Prisma Integration

---

*Part 022 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
