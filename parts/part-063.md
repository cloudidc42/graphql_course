# Part 063 — Load Testing & Performance Benchmarking 📊

> **ระดับ:** Expert | **เวลาเรียน:** 75 นาที | **ขั้นตอนที่:** 1426–1460

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Load testing กับ k6
- Artillery สำหรับ GraphQL
- Performance profiling
- Bottleneck identification
- Benchmark queries
- Load test scripts
- Interpreting results
- Capacity planning

---

## 📌 Step 1426: k6 Load Testing

```bash
npm install -g k6
# หรือ brew install k6
```

```javascript
// load-tests/graphql.test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Counter, Rate, Trend } from 'k6/metrics';

// Custom metrics
const successRate = new Rate('success_rate');
const queryDuration = new Trend('query_duration');
const errorCount = new Counter('error_count');

// Test configuration
export const options = {
  stages: [
    { duration: '30s', target: 10 },   // Ramp up: 0 → 10 users
    { duration: '1m', target: 50 },    // Stay at 50 users
    { duration: '1m', target: 100 },   // Scale up: 50 → 100
    { duration: '2m', target: 100 },   // Peak load
    { duration: '30s', target: 0 },    // Ramp down
  ],
  
  thresholds: {
    // SLA: 95% ของ requests ต้อง < 500ms
    'http_req_duration': ['p(95)<500'],
    // Error rate ต้อง < 1%
    'success_rate': ['rate>0.99'],
    // P99 < 2000ms
    'query_duration': ['p(99)<2000'],
  },
};

const GRAPHQL_URL = __ENV.GRAPHQL_URL || 'http://localhost:4000/graphql';
const AUTH_TOKEN = __ENV.AUTH_TOKEN || '';

// Helper: send GraphQL request
function gql(query, variables = {}) {
  const payload = JSON.stringify({ query, variables });
  
  const params = {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${AUTH_TOKEN}`,
    },
  };
  
  const start = Date.now();
  const response = http.post(GRAPHQL_URL, payload, params);
  queryDuration.add(Date.now() - start);
  
  return response;
}

// Test scenario
export default function () {
  // Scenario 1: Browse products
  const productsQuery = `
    query GetProducts {
      products(first: 20) {
        edges {
          node {
            id
            name
            price
            category { name }
            images { url }
          }
        }
        pageInfo { hasNextPage }
        totalCount
      }
    }
  `;
  
  const productsRes = gql(productsQuery);
  
  const productsSuccess = check(productsRes, {
    'products status 200': (r) => r.status === 200,
    'products no errors': (r) => {
      const body = JSON.parse(r.body);
      return !body.errors;
    },
    'products has data': (r) => {
      const body = JSON.parse(r.body);
      return body.data?.products?.edges?.length > 0;
    },
  });
  
  successRate.add(productsSuccess);
  if (!productsSuccess) errorCount.add(1);
  
  sleep(0.5);
  
  // Scenario 2: Search
  const searchQuery = `
    query Search($term: String!) {
      search(term: $term) {
        edges { node { ... on Product { id name price } } }
        totalCount
      }
    }
  `;
  
  const searchRes = gql(searchQuery, { term: 'laptop' });
  
  const searchSuccess = check(searchRes, {
    'search status 200': (r) => r.status === 200,
    'search duration < 1s': (r) => r.timings.duration < 1000,
  });
  
  successRate.add(searchSuccess);
  
  sleep(1);
}
```

---

## 📌 Step 1427: Advanced k6 Scenarios

```javascript
// load-tests/e2e-scenario.test.js
// Complete user journey simulation

import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { SharedArray } from 'k6/data';

// Load test data
const users = new SharedArray('users', function () {
  return JSON.parse(open('./test-users.json'));
});

export const options = {
  scenarios: {
    // Scenario 1: Regular shoppers (70% of traffic)
    shoppers: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 70 },
        { duration: '5m', target: 70 },
        { duration: '30s', target: 0 },
      ],
      tags: { scenario: 'shoppers' },
    },
    
    // Scenario 2: Admin users (10% of traffic)
    admins: {
      executor: 'constant-vus',
      vus: 5,
      duration: '6m',
      tags: { scenario: 'admins' },
    },
    
    // Scenario 3: API integrations (20% of traffic)
    api_clients: {
      executor: 'constant-arrival-rate',
      rate: 30,        // 30 requests/second
      timeUnit: '1s',
      duration: '6m',
      preAllocatedVUs: 20,
      maxVUs: 50,
      tags: { scenario: 'api_clients' },
    },
  },
  
  thresholds: {
    'http_req_duration{scenario:shoppers}': ['p(95)<800'],
    'http_req_duration{scenario:admins}': ['p(95)<2000'],
    'http_req_duration{scenario:api_clients}': ['p(95)<300'],
    'http_req_failed': ['rate<0.01'],
  },
};

// Shopper scenario
function shopperJourney(user) {
  group('Browse & Search', () => {
    // View homepage products
    gql('query { featuredProducts(limit: 8) { id name price images { url } } }');
    sleep(2);
    
    // Search
    gql('query Search($q: String!) { search(term: $q) { edges { node { ... on Product { id name price } } } } }',
      { q: 'phone' });
    sleep(1);
  });
  
  group('View Product', () => {
    const productId = 'prod-123'; // ใช้ real product IDs จาก test data
    gql(`query GetProduct($id: ID!) {
      product(id: $id) {
        id name price description
        images { url }
        variants { id name price stock }
        reviews(first: 5) { id rating comment author { name } }
      }
    }`, { id: productId });
    sleep(3);
  });
  
  group('Checkout', () => {
    // Add to cart
    gql('mutation AddToCart($productId: ID!) { addToCart(productId: $productId, quantity: 1) { id total } }',
      { productId: 'prod-123' });
    sleep(1);
    
    // Place order (requires auth)
    if (user.token) {
      gql(`mutation Checkout($input: CheckoutInput!) {
        checkout(input: $input) {
          ... on CheckoutSuccess { orderId total }
          ... on CheckoutError { message }
        }
      }`, { input: { cartId: user.cartId, paymentMethodId: user.paymentMethodId } });
    }
  });
}

export default function () {
  const user = users[__VU % users.length];
  shopperJourney(user);
}
```

---

## 📌 Step 1428: Artillery Load Testing

```bash
npm install -g artillery @artillery/plugin-expect
```

```yaml
# load-tests/artillery.yml
config:
  target: "http://localhost:4000"
  phases:
    - duration: 60
      arrivalRate: 5
      name: Warm up
    - duration: 120
      arrivalRate: 50
      name: Ramp up
    - duration: 300
      arrivalRate: 100
      name: Sustained load
  
  plugins:
    expect: {}
  
  defaults:
    headers:
      Content-Type: "application/json"

scenarios:
  - name: "Browse Products"
    weight: 60  # 60% of traffic
    flow:
      - post:
          url: "/graphql"
          json:
            query: |
              query {
                products(first: 20) {
                  edges { node { id name price } }
                  totalCount
                }
              }
          expect:
            - statusCode: 200
            - contentType: json
            - hasProperty: "data.products"
  
  - name: "Search"
    weight: 30
    flow:
      - post:
          url: "/graphql"
          json:
            query: |
              query Search($term: String!) {
                search(term: $term) {
                  edges { node { ... on Product { id name } } }
                }
              }
            variables:
              term: "{{ $randomString() }}"
          expect:
            - statusCode: 200
  
  - name: "User Profile"
    weight: 10
    flow:
      - post:
          url: "/graphql"
          json:
            query: |
              query { me { id name email } }
          headers:
            Authorization: "Bearer {{ $env.TEST_TOKEN }}"
          expect:
            - statusCode: 200
```

---

## 📌 Step 1429: Profiling กับ clinic.js

```bash
npm install -g clinic
npm install -g autocannon
```

```bash
# 1. Profile CPU usage
clinic doctor -- node dist/server.js

# 2. Flame graph
clinic flame -- node dist/server.js

# 3. Heap profiling
clinic heapprofile -- node dist/server.js

# 4. Benchmark specific operation ด้วย autocannon
autocannon \
  -c 100 \            # 100 concurrent connections
  -d 30 \             # 30 seconds
  -m POST \
  -H "Content-Type: application/json" \
  -b '{"query":"{ products(first:20) { edges { node { id name price } } } }"}' \
  http://localhost:4000/graphql

# Output:
# Running 30s test @ http://localhost:4000/graphql
# 100 connections
#
# Stat    2.5%  50%   97.5%  99%   Avg     Stdev   Max
# Latency 5 ms  12 ms 89 ms  150 ms 15.3 ms 22.1 ms 1234 ms
#
# HTTP codes: 1xx - 0, 2xx - 45231, 3xx - 0, 4xx - 0, 5xx - 0
# Req/Sec: 1507.70
```

---

## 📌 Step 1430: Interpreting Results & Capacity Planning

```
Performance Targets (E-commerce SLA):

Query type    | P50  | P95  | P99  | Max
--------------|------|------|------|------
Product List  | 50ms | 200ms| 500ms| 1s
Search        | 100ms| 500ms| 1s   | 2s
Product Detail| 30ms | 150ms| 300ms| 800ms
Checkout      | 500ms| 2s   | 5s   | 10s
Subscription  | N/A  | N/A  | N/A  | N/A (streaming)

Capacity Planning:
- ถ้า P99 = 500ms ที่ 100 concurrent users
- เพิ่มเป็น 200 users → ดู P99 เปลี่ยนอย่างไร?
- Linear scaling: P99 เพิ่มเป็น 2x → ต้อง scale ตาม load
- Sub-linear scaling: เพิ่ม users 2x แต่ P99 เพิ่มแค่ 20% → ดีมาก

Bottleneck indicators:
1. P99 สูงมาก แต่ P50 ต่ำ → outliers (DB locks, GC pause)
2. Latency เพิ่ม linear กับ users → throughput bound
3. Errors เพิ่มที่ threshold → connection pool exhausted
4. Memory เพิ่มต่อเนื่อง → memory leak
5. CPU 100% → compute bound → horizontal scaling
```

---

## 📌 Step 1431: สรุป Part 063

### เนื้อหาที่เรียนรู้

✅ k6 load testing  
✅ Advanced k6 scenarios (multiple users)  
✅ Artillery YAML config  
✅ clinic.js profiling  
✅ autocannon benchmarking  
✅ Results interpretation  
✅ Capacity planning  

### Load Test Checklist

```
Before Load Test:
□ Test environment ≈ production (same specs)
□ Test data ที่ realistic
□ Auth tokens สำหรับ test users
□ Monitoring ready (Grafana, Prometheus)

During Test:
□ Watch CPU, Memory, DB connections
□ Watch error rates
□ Watch P95, P99 latency
□ Watch GC activity

After Test:
□ Document results
□ Compare กับ baseline
□ Identify bottlenecks
□ Create action items
```

### ในส่วนถัดไป

➡️ **[Part 064](./part-064.md)** — Production Deployment & Zero-Downtime Updates

---

*Part 063 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
