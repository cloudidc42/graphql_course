# Part 039 — GraphQL Penetration Testing Methodology 🔍

> **ระดับ:** Expert | **เวลาเรียน:** 120 นาที | **ขั้นตอนที่:** 611–650

> ⚠️ **หมายเหตุ**: เนื้อหาในส่วนนี้มีวัตถุประสงค์เพื่อการศึกษาด้านความปลอดภัยและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำเทคนิคเหล่านี้ไปใช้กับระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Penetration testing methodology
- Reconnaissance สำหรับ GraphQL
- เครื่องมือ GraphQL security testing
- การทดสอบ authorization
- การทดสอบ injection vulnerabilities
- การทดสอบ business logic
- รายงาน findings
- การแก้ไขช่องโหว่

---

## 📌 Step 611: Pentest Methodology

```
GraphQL Pentest Phases:

1. Reconnaissance
   - ค้นหา GraphQL endpoints
   - ตรวจสอบ introspection
   - schema enumeration
   
2. Enumeration
   - Query structure mapping
   - Type analysis
   - Authentication analysis
   
3. Vulnerability Testing
   - Injection attacks
   - Authorization bypass
   - Business logic flaws
   - DoS potential
   
4. Exploitation (Authorized only)
   - Confirm vulnerabilities
   - Assess impact
   
5. Reporting
   - Document findings
   - Remediation recommendations
```

---

## 📌 Step 612: GraphQL Endpoint Discovery

```bash
# Common GraphQL endpoint paths
/graphql
/api/graphql
/v1/graphql
/v2/graphql
/query
/gql
/api/query

# ตรวจสอบ introspection enabled
curl -X POST https://target.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __typename }"}'

# ถ้า response มี data.__ typename = เจอ GraphQL endpoint แล้ว
```

---

## 📌 Step 613: Introspection Testing

```javascript
// ทดสอบว่า introspection เปิดอยู่ไหม
const introspectionQuery = `
  query {
    __schema {
      types { name }
      queryType { name }
      mutationType { name }
      subscriptionType { name }
    }
  }
`;

// Tools สำหรับ schema extraction
// InQL (Burp Suite plugin):
// - Automated introspection
// - Generate mutations/queries
// - Visualize schema

// GraphQL Voyager: visualize schema as graph
// graphql-introspection-json: dump schema ออกมา

// clairvoyance: enumerate schema แม้ introspection ปิด
// ใช้ field suggestion errors เพื่อ guess field names
```

---

## 📌 Step 614: Field Suggestion Exploitation (Educational)

```javascript
// ⚠️ Educational - แสดงให้เห็นว่า server leak ข้อมูลผ่าน error messages

// ถ้าส่ง field ที่ไม่มี → server suggest field ที่ใกล้เคียง
// "Did you mean 'email'?" → เจอ field ชื่อ email

// ป้องกัน: disable field suggestions
// Apollo Server 4+ ทำด้วย:
const server = new ApolloServer({
  // ปิด field suggestions
  dangerouslyDisableValidation: false, // ← ไม่ disable validation
  
  // หรือ format error เพื่อซ่อน suggestions
  formatError: (error) => {
    const msg = error.message;
    if (msg.includes('Did you mean')) {
      return { message: 'Invalid field', extensions: { code: 'BAD_USER_INPUT' } };
    }
    return error;
  },
});
```

---

## 📌 Step 615: Authorization Testing

```javascript
// ทดสอบ IDOR (Insecure Direct Object Reference)

// 1. Login as user A
const tokenA = 'eyJhbGc...';

// 2. ดู order ของ user A
const myOrder = await graphql(`
  query { order(id: "order-A-001") { id total } }
`, { headers: { Authorization: `Bearer ${tokenA}` } });

// 3. ลอง access order ของ user B
const otherOrder = await graphql(`
  query { order(id: "order-B-002") { id total } }
`, { headers: { Authorization: `Bearer ${tokenA}` } });

// ถ้า otherOrder ส่งข้อมูลมา → IDOR vulnerability!

// ทดสอบ Horizontal privilege escalation:
// - User A เข้าถึงข้อมูล User B

// ทดสอบ Vertical privilege escalation:
// - Regular user เข้าถึง admin fields
const adminQuery = `
  query {
    users(filter: { all: true }) { id email role password }
    systemStats { uptime dbConnections memoryUsage }
  }
`;
```

---

## 📌 Step 616: Injection Testing

```javascript
// SQL Injection testing patterns
// ⚠️ ทดสอบเฉพาะระบบของตัวเองหรือที่ได้รับอนุญาต

// Test 1: Basic SQL injection in string inputs
const payloads = [
  "' OR '1'='1",
  "1' OR '1'='1' --",
  "'; DROP TABLE users; --",
  `" OR ""="`,
  "admin'--",
];

for (const payload of payloads) {
  const result = await graphql(`
    query { user(email: "${payload}") { id name } }
  `);
  
  // Red flags:
  // - SQL error messages
  // - Returns unexpected data
  // - Error about syntax
}

// Test 2: Time-based blind SQL injection
const timingPayload = "1'; SELECT SLEEP(5); --";
// ถ้า response ช้า 5 วิ → blind SQL injection possible

// Test 3: NoSQL injection (MongoDB)
const nosqlPayload = { $gt: "" };
// ใน variables: { "email": { "$gt": "" } }
// → อาจ return user ทั้งหมด
```

---

## 📌 Step 617: DoS Testing

```javascript
// ทดสอบ resource exhaustion (ระบบที่ได้รับอนุญาต)

// Test 1: Deep nesting
const deepQuery = `query {` +
  'user(id: "1") { friends { friends { friends { friends { friends { '.repeat(8) +
  'name }}}}}}}}}}'.repeat(8) + `}`;

// Test 2: Complexity bomb (aliases)
const aliasQuery = `query {` +
  Array.from({ length: 100 }, (_, i) => `u${i}: user(id: "${i}") { name email posts { title comments { body } } }`).join('\n') +
  '}';

// Test 3: Batch request abuse
const batchRequest = Array.from({ length: 50 }, (_, i) => ({
  query: `query { user(id: "${i}") { name } }`,
}));

// Expected: server should block these with 400 errors
// Red flag: server processes them all
```

---

## 📌 Step 618: Business Logic Testing

```javascript
// ทดสอบ business logic flaws

// Test 1: Race condition ใน order
// ส่ง 2 orders พร้อมกันโดยใช้ stock เดียวกัน
const [o1, o2] = await Promise.all([
  graphql(`mutation { checkout(cartId: "cart-1") { id } }`),
  graphql(`mutation { checkout(cartId: "cart-1") { id } }`),
]);
// ถ้าทั้งคู่ success → race condition ทำให้ stock ติดลบ

// Test 2: Price manipulation
const addToCartResult = await graphql(`
  mutation { addToCart(productId: "p1", quantity: 1) { total } }
`);
// แล้วดูว่าเราส่ง price เองได้ไหม

// Test 3: Coupon stacking/reuse
await graphql(`mutation { applyCoupon(code: "SAVE10") { discount } }`);
await graphql(`mutation { applyCoupon(code: "SAVE10") { discount } }`); // ใช้ซ้ำ
```

---

## 📌 Step 619: เขียน Security Report

```markdown
# Security Assessment Report — [Application Name]

**Date:** 2024-01-15
**Tester:** [Your Name]
**Scope:** https://api.example.com/graphql

## Executive Summary

พบช่องโหว่ความรุนแรง High 2 รายการ, Medium 3 รายการ, Low 1 รายการ

## Findings

### [HIGH] IDOR in Order Query
**Description:** ผู้ใช้ทั่วไปสามารถอ่านข้อมูล order ของผู้ใช้อื่นได้

**Steps to Reproduce:**
1. Login as User A
2. Query `{ order(id: "ORDER-B-001") { id total customer { email } } }`
3. Response contains User B's order details

**Impact:** Information disclosure ของข้อมูลส่วนตัวและข้อมูลการซื้อขาย

**Recommendation:** เพิ่ม authorization check ใน order resolver

**Severity:** High (CVSS 7.5)

---

### [MEDIUM] GraphQL Introspection Enabled in Production
**Description:** Introspection เปิดอยู่ใน production ทำให้ attacker เห็น schema ทั้งหมด

**Recommendation:** 
```javascript
const server = new ApolloServer({
  introspection: process.env.NODE_ENV !== 'production',
  validationRules: [NoSchemaIntrospectionCustomRule],
});
```

## Remediation Timeline

| Finding | Severity | Fix By |
|---------|----------|--------|
| IDOR in Order | High | 3 days |
| Introspection | Medium | 7 days |
```

---

## 📌 Step 620: สรุป Part 039

### เนื้อหาที่เรียนรู้

✅ Pentest methodology  
✅ Endpoint discovery  
✅ Introspection testing  
✅ Authorization/IDOR testing  
✅ Injection testing patterns  
✅ DoS testing  
✅ Business logic testing  
✅ Security report format  

### ขั้นตอนการทำ Pentest ที่ถูกต้อง

```
1. ขอ written permission เสมอ (Scope of Work)
2. ทดสอบเฉพาะ environment ที่กำหนด
3. ไม่รบกวน production โดยไม่จำเป็น
4. รายงาน findings ทันทีที่พบ (ถ้า critical)
5. ส่ง report ครบถ้วนพร้อม remediation
6. ไม่ leak ข้อมูลที่พบระหว่าง test
```

### ในส่วนถัดไป

➡️ **[Part 040](./part-040.md)** — Complete Security Hardening Checklist

---

*Part 039 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
