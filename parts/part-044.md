# Part 044 — Introspection Attacks 🔴

> **ระดับ:** Expert | **เวลาเรียน:** 120 นาที | **ขั้นตอนที่:** 651–690

---

## ⚠️ คำเตือน (Disclaimer)

เนื้อหาในส่วนนี้เป็น **การศึกษาเพื่อป้องกัน** (Defensive Security) เท่านั้น ความรู้เหล่านี้ใช้สำหรับ:
- ทดสอบ API ของตัวเองหรือที่ได้รับอนุญาต
- Penetration testing แบบ authorized
- พัฒนา defensive measures

**ห้ามใช้กับระบบที่ไม่ได้รับอนุญาต**

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Introspection คืออะไร และทำงานอย่างไร
- วิธีที่ attacker ใช้ Introspection
- เครื่องมือ reconnaissance สำหรับ GraphQL
- วิธีป้องกัน Introspection attacks
- Field suggestion attacks
- Schema fingerprinting

---

## 📌 Step 651: GraphQL Introspection คืออะไร

**Introspection** คือความสามารถของ GraphQL ในการ **สำรวจ schema ของตัวเอง**:

```graphql
# Query เพื่อดู schema ทั้งหมด
query {
  __schema {
    types {
      name
      kind
      description
      fields {
        name
        type {
          name
          kind
        }
      }
    }
  }
}
```

### ข้อมูลที่ Introspection เปิดเผย

```graphql
# 1. Types ทั้งหมด
__schema { types { name kind } }

# 2. Queries ที่มี
__schema { queryType { fields { name args { name type } } } }

# 3. Mutations ที่มี
__schema { mutationType { fields { name args { name type } } } }

# 4. Subscriptions
__schema { subscriptionType { fields { name } } }

# 5. รายละเอียดของ type เฉพาะ
__type(name: "User") {
  fields {
    name
    type { name kind ofType { name kind } }
    isDeprecated
    deprecationReason
    description
  }
}
```

---

## 📌 Step 652: ทำไม Introspection ถึงอันตราย

### Attack Scenario

```
1. Attacker พบ GraphQL endpoint: https://target.com/graphql
2. ส่ง introspection query
3. ได้รับ schema ทั้งหมด
4. ค้นพบ:
   - Field ที่ sensitive: adminDashboard, userPasswords, secretKeys
   - Mutations ที่ dangerous: deleteAllUsers, resetAdminPassword
   - Types ที่เปิดเผย business logic: PaymentInfo, InternalUser
5. สร้าง attack vector ที่เหมาะสม
```

### ตัวอย่าง Introspection Query เต็ม

```graphql
query IntrospectionQuery {
  __schema {
    queryType { name }
    mutationType { name }
    subscriptionType { name }
    types {
      ...FullType
    }
    directives {
      name
      description
      locations
      args {
        ...InputValue
      }
    }
  }
}

fragment FullType on __Type {
  kind
  name
  description
  fields(includeDeprecated: true) {
    name
    description
    args {
      ...InputValue
    }
    type {
      ...TypeRef
    }
    isDeprecated
    deprecationReason
  }
  inputFields {
    ...InputValue
  }
  interfaces {
    ...TypeRef
  }
  enumValues(includeDeprecated: true) {
    name
    description
    isDeprecated
    deprecationReason
  }
  possibleTypes {
    ...TypeRef
  }
}

fragment InputValue on __InputValue {
  name
  description
  type { ...TypeRef }
  defaultValue
}

fragment TypeRef on __Type {
  kind
  name
  ofType {
    kind
    name
    ofType {
      kind
      name
      ofType {
        kind
        name
        ofType {
          kind
          name
          ofType {
            kind
            name
            ofType {
              kind
              name
              ofType {
                kind
                name
              }
            }
          }
        }
      }
    }
  }
}
```

---

## 📌 Step 653: เครื่องมือ Reconnaissance

### 1. GraphQL Voyager (Visualization)

```bash
# ติดตั้ง
npm install graphql-voyager

# หรือใช้ online tool
# https://graphql-voyager.com/
```

```html
<!-- embed ใน HTML -->
<script src="https://cdn.jsdelivr.net/npm/graphql-voyager/dist/voyager.worker.js"></script>
<script src="https://cdn.jsdelivr.net/npm/graphql-voyager/dist/voyager.min.js"></script>

<div id="voyager"></div>
<script>
  GraphQLVoyager.init(document.getElementById('voyager'), {
    introspection: fetchIntrospection('https://target.com/graphql'),
  });
</script>
```

### 2. InQL (Burp Suite plugin)

```bash
# ใช้ Burp Suite + InQL extension
# https://github.com/doyensec/inql

# สิ่งที่ InQL ทำ:
# - Auto-detect GraphQL endpoints
# - Extract schema via introspection
# - Generate queries/mutations
# - Fuzzing support
```

### 3. GraphQL Map

```bash
# https://github.com/swisskyrepo/GraphQLmap
git clone https://github.com/swisskyrepo/GraphQLmap
cd GraphQLmap
pip3 install -r requirements.txt

# ใช้งาน
python3 graphqlmap.py -u https://target.com/graphql --dump_new

# Features:
# - Schema dumping
# - Field fuzzing
# - Query generation
```

### 4. clairvoyance (Schema Reconstruction)

```bash
# เมื่อ Introspection ถูกปิด — ใช้ field suggestion
# https://github.com/nikitastupin/clairvoyance
pip install clairvoyance

clairvoyance -u https://target.com/graphql -o schema.json
```

### 5. Feroxbuster / ffuf สำหรับ Endpoint Discovery

```bash
# ค้นหา GraphQL endpoints
ffuf -u https://target.com/FUZZ \
  -w /path/to/wordlist.txt \
  -mc 200 \
  -H "Content-Type: application/json" \
  -d '{"query":"{__typename}"}'

# Common GraphQL paths:
# /graphql
# /api/graphql
# /v1/graphql
# /query
# /gql
# /graphiql
# /playground
```

---

## 📌 Step 654: Field Suggestion Attack

แม้ปิด Introspection แล้ว — GraphQL ยังมี **field suggestion** ที่อาจ leak ข้อมูล:

```graphql
# Query ผิด
{
  usr {   ← typo
    name
  }
}

# Error message บอก suggestions!
{
  "errors": [{
    "message": "Cannot query field \"usr\" on type \"Query\". Did you mean \"user\" or \"users\"?"
  }]
}
```

### Exploit Field Suggestions

```python
# Python script สำหรับ extract field names ผ่าน suggestions
import requests
import string

TARGET = "https://target.com/graphql"

def query_field(field_name):
    response = requests.post(TARGET, json={
        "query": f"{{ {field_name} }}"
    })
    return response.json()

def get_suggestions(typo):
    result = query_field(typo)
    errors = result.get("errors", [])
    for error in errors:
        msg = error.get("message", "")
        if "Did you mean" in msg:
            # Extract suggestions
            import re
            suggestions = re.findall(r'"([^"]+)"', msg.split("Did you mean")[1])
            return suggestions
    return []

# ลอง characters ต่างๆ
found_fields = set()
for char in string.ascii_lowercase:
    suggestions = get_suggestions(char * 3)  # ใช้ typo ที่แน่ว่าไม่มี
    found_fields.update(suggestions)
    
print("Found fields:", found_fields)
```

---

## 📌 Step 655: Schema Fingerprinting

แม้ปิด Introspection — เราสามารถระบุได้ว่าใช้ library อะไร:

```graphql
# Apollo Server มี built-in types
query {
  __schema {
    types {
      name
    }
  }
}
# ผลลัพธ์: CacheControlScope, Upload → บ่งบอก Apollo

# Hasura มี specific patterns
# { users_aggregate { aggregate { count } } }
# → ถ้าทำงานได้ = Hasura

# Dgraph มี @cascade, @filter directives
# WPGraphQL มี WordPress-specific types
```

---

## 📌 Step 656: การโจมตี Introspection ในทางปฏิบัติ

### Scenario: Target มี GraphQL API ที่ยังไม่ได้ป้องกัน

```bash
# ขั้นตอน 1: ยืนยัน GraphQL endpoint
curl -s -X POST https://target.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{__typename}"}' | jq
# Response: {"data": {"__typename": "Query"}} ✓

# ขั้นตอน 2: Full introspection dump
curl -s -X POST https://target.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { types { name fields { name } } } }"}' | jq

# ขั้นตอน 3: ดู sensitive types
# สมมติพบ: AdminUser, PaymentInfo, SecretKey

# ขั้นตอน 4: Query sensitive fields
curl -X POST https://target.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ adminUsers { email passwordHash role } }"}'
```

---

## 📌 Step 657: ป้องกัน — ปิด Introspection

### Apollo Server 4

```javascript
// ✅ ปิด introspection ใน Production
const server = new ApolloServer({
  typeDefs,
  resolvers,
  // ปิด introspection ใน production
  introspection: process.env.NODE_ENV !== 'production',
});
```

### GraphQL Yoga

```javascript
import { createYoga } from 'graphql-yoga';

const yoga = createYoga({
  schema,
  // ปิด introspection
  maskedErrors: true,
  graphqlEndpoint: '/graphql',
  
  // Plugin ปิด introspection
  plugins: [
    {
      onExecute({ args }) {
        const isIntrospection = args.document.definitions.some(
          def => def.selectionSet?.selections?.some(
            sel => sel.name?.value?.startsWith('__')
          )
        );
        
        if (isIntrospection && process.env.NODE_ENV === 'production') {
          throw new Error('Introspection is disabled in production');
        }
      },
    },
  ],
});
```

### Custom Validation Rule

```javascript
import { NoSchemaIntrospectionCustomRule } from 'graphql';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  // Custom validation rules
  validationRules: process.env.NODE_ENV === 'production'
    ? [NoSchemaIntrospectionCustomRule]
    : [],
});
```

---

## 📌 Step 658: ป้องกัน — Whitelist Introspection

แทนที่จะปิดทั้งหมด — อนุญาตเฉพาะ trusted clients:

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  introspection: true, // เปิดไว้ แต่ตรวจสอบใน plugin
  plugins: [
    {
      async requestDidStart({ request, contextValue }) {
        const isIntrospection = request.query?.includes('__schema') || 
                                request.query?.includes('__type');
        
        if (isIntrospection) {
          // ตรวจสอบ API key
          const apiKey = request.http?.headers.get('x-api-key');
          if (apiKey !== process.env.INTROSPECTION_API_KEY) {
            throw new GraphQLError('Introspection requires valid API key', {
              extensions: { code: 'FORBIDDEN' }
            });
          }
          
          // ตรวจสอบ IP whitelist
          const clientIp = contextValue.req?.ip;
          const allowedIps = process.env.ALLOWED_INTROSPECTION_IPS?.split(',') || [];
          if (allowedIps.length > 0 && !allowedIps.includes(clientIp)) {
            throw new GraphQLError('Introspection not allowed from this IP');
          }
        }
      },
    },
  ],
});
```

---

## 📌 Step 659: ป้องกัน — Field Suggestion Disable

```javascript
// Apollo Server 4 — ปิด suggestions ด้วย custom error formatter
const server = new ApolloServer({
  typeDefs,
  resolvers,
  formatError: (formattedError) => {
    // ลบ "Did you mean" suggestions
    if (formattedError.message.includes('Did you mean')) {
      return {
        ...formattedError,
        message: formattedError.message.split('Did you mean')[0].trim(),
      };
    }
    return formattedError;
  },
});

// หรือ replace message ทั้งหมด
formatError: (formattedError) => {
  if (process.env.NODE_ENV === 'production') {
    // ซ่อน schema-related errors
    if (formattedError.extensions?.code === 'GRAPHQL_VALIDATION_FAILED') {
      return {
        message: 'Invalid query',
        extensions: { code: 'GRAPHQL_VALIDATION_FAILED' },
      };
    }
  }
  return formattedError;
},
```

---

## 📌 Step 660: ป้องกัน — Schema Obfuscation (ใช้ด้วยความระวัง)

```javascript
// ตัวอย่าง: ใช้ generic field names
// ❌ อย่าทำใน production จริง — ทำให้ DX แย่มาก

// แทนที่จะเป็น:
type User {
  id: ID!
  email: String!
  passwordHash: String!
  adminLevel: Int!
}

// ใช้:
type User {
  id: ID!
  email: String!
  # ไม่ expose sensitive fields เลย
}
```

---

## 📌 Step 661: ป้องกัน — Apollo Studio Private Schema

```javascript
// ใช้ Apollo Studio สำหรับ introspection (secure)
import { ApolloServerPluginSchemaReporting } from '@apollo/server/plugin/schemaReporting';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  introspection: false, // ปิด public introspection
  plugins: [
    // Report schema ไปยัง Apollo Studio เท่านั้น
    ApolloServerPluginSchemaReporting({
      initialSchemaReportingDelayRandomness: 0,
    }),
  ],
});
```

---

## 📌 Step 662: Detection — Monitor Introspection Attempts

```javascript
// Plugin สำหรับ detect และ log introspection attempts
const introspectionMonitorPlugin = {
  async requestDidStart({ request, contextValue }) {
    const isIntrospection = 
      request.query?.includes('__schema') || 
      request.query?.includes('__type') ||
      request.query?.includes('__typename') === false; // ← ไม่นับ __typename
    
    if (isIntrospection) {
      const clientInfo = {
        ip: contextValue.req?.ip,
        userAgent: request.http?.headers.get('user-agent'),
        timestamp: new Date().toISOString(),
        query: request.query?.substring(0, 500),
      };
      
      console.warn('🔍 Introspection attempt detected:', clientInfo);
      
      // ส่งไปยัง monitoring system
      await securityMonitor.alert({
        type: 'INTROSPECTION_ATTEMPT',
        severity: 'MEDIUM',
        ...clientInfo,
      });
    }
  },
};
```

---

## 📌 Step 663: ตรวจสอบ Schema สำหรับ Sensitive Data

```javascript
// Script ตรวจหา sensitive field names
const sensitivePatterns = [
  /password/i,
  /secret/i,
  /token/i,
  /apikey/i,
  /api_key/i,
  /private/i,
  /internal/i,
  /admin/i,
  /credential/i,
  /hash/i,
  /salt/i,
  /ssn/i,
  /creditcard/i,
  /cvv/i,
];

function checkSchemaForSensitiveFields(schema) {
  const issues = [];
  const typeMap = schema.getTypeMap();
  
  for (const [typeName, type] of Object.entries(typeMap)) {
    if (typeName.startsWith('__')) continue; // Skip introspection types
    
    if (type.getFields) {
      const fields = type.getFields();
      for (const [fieldName] of Object.entries(fields)) {
        for (const pattern of sensitivePatterns) {
          if (pattern.test(fieldName)) {
            issues.push({
              type: typeName,
              field: fieldName,
              pattern: pattern.toString(),
              severity: 'HIGH',
            });
          }
        }
      }
    }
  }
  
  return issues;
}

// ใช้งาน
import { buildSchema } from 'graphql';
const schema = buildSchema(typeDefs);
const issues = checkSchemaForSensitiveFields(schema);

if (issues.length > 0) {
  console.warn('⚠️ Sensitive fields in schema:', issues);
  // ถ้า production — throw error
  if (process.env.NODE_ENV === 'production') {
    throw new Error('Schema contains potentially sensitive field names');
  }
}
```

---

## 📌 Step 664: Defense in Depth Strategy

```
Layer 1: Disable Introspection in Production
→ apollo: introspection: false
→ yoga: NoSchemaIntrospectionCustomRule

Layer 2: Disable Field Suggestions
→ formatError: ลบ "Did you mean..." messages

Layer 3: Authentication for Development Introspection
→ Require API key for /graphql in staging
→ VPN-only access to development endpoints

Layer 4: Schema Audit
→ ตรวจสอบ sensitive field names
→ ไม่ expose internal fields ใน public schema
→ ใช้ depth limiting

Layer 5: Monitoring
→ Log introspection attempts
→ Alert on suspicious patterns
→ Rate limit /graphql endpoint

Layer 6: Network Security
→ WAF rules สำหรับ GraphQL introspection
→ IP whitelisting สำหรับ admin endpoints
```

---

## 📌 Step 665: Security Checklist

```
GraphQL Introspection Security Checklist:

✅ ปิด introspection ใน production environment
✅ ปิด field suggestions ใน production
✅ ปิด GraphiQL/Playground ใน production
✅ ไม่ expose sensitive fields ใน public schema
✅ Monitor และ log introspection attempts
✅ ทดสอบ API ด้วย introspection queries ก่อน deploy
✅ Audit schema เป็นประจำสำหรับ sensitive data
✅ ใช้ depth limiting
✅ ใช้ rate limiting บน /graphql endpoint
✅ แยก schema สำหรับ public/internal APIs
✅ ใช้ authentication ก่อน access introspection ใน dev/staging
✅ Regular security testing (penetration testing)
```

---

## 📌 Step 666: ตัวอย่างจริง — Vulnerable vs Secure

### Vulnerable Configuration

```javascript
// ❌ อันตราย — เปิด introspection ทุกที่
const server = new ApolloServer({
  typeDefs: `
    type Query {
      users: [User!]!
      adminUsers: [AdminUser!]!  # ← expose admin users
    }
    type AdminUser {
      id: ID!
      email: String!
      passwordHash: String!   # ← expose password hash!!
      secretKey: String!      # ← expose secret key!!
    }
  `,
  resolvers,
  introspection: true,  # ← เปิดใน production!
});
```

### Secure Configuration

```javascript
// ✅ ปลอดภัย
import { NoSchemaIntrospectionCustomRule } from 'graphql';

const isProduction = process.env.NODE_ENV === 'production';

const server = new ApolloServer({
  typeDefs: `
    type Query {
      users: [User!]!
      # ไม่มี adminUsers ใน public schema
    }
    type User {
      id: ID!
      name: String!
      email: String!
      # ไม่มี passwordHash, secretKey
    }
  `,
  resolvers,
  introspection: !isProduction,
  validationRules: isProduction ? [NoSchemaIntrospectionCustomRule] : [],
  formatError: (error) => {
    if (isProduction && error.message.includes('Did you mean')) {
      return { ...error, message: 'Invalid query' };
    }
    return error;
  },
  plugins: [
    {
      async requestDidStart({ request, contextValue }) {
        if (request.query?.includes('__schema')) {
          console.warn('Introspection attempt from', contextValue.req?.ip);
        }
      },
    },
  ],
});
```

---

## 📌 Step 667: สรุป Part 044

### Key Takeaways

```
1. Introspection เปิดเผย schema ทั้งหมด → ปิดใน production

2. Field suggestions ก็ leak ข้อมูลได้ → ปิดด้วย formatError

3. เครื่องมือ reconnaissance:
   - GraphQL Voyager (visualization)
   - InQL (Burp Suite)
   - clairvoyance (schema reconstruction)
   - GraphQLmap

4. Defense in Depth:
   - Disable introspection
   - Disable suggestions
   - Audit schema
   - Monitor attempts
   - Network security
```

### เนื้อหาที่เรียนรู้

✅ Introspection คืออะไร  
✅ วิธีที่ attacker ใช้  
✅ เครื่องมือ reconnaissance  
✅ Field suggestion attacks  
✅ วิธีปิด introspection (Apollo, Yoga)  
✅ Schema auditing  
✅ Security checklist  

### Homework

1. ทดสอบ introspection บน API ของตัวเอง
2. ปิด introspection ใน production และทดสอบว่าทำงานถูกต้อง
3. เขียน script ตรวจหา sensitive fields ใน schema

### ในส่วนถัดไป

➡️ **[Part 045](./part-045.md)** — GraphQL Injection Attacks

---

*Part 044 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
