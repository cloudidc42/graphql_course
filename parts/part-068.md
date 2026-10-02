# Part 068 — Apollo Federation: Router & Supergraph 🛣️

> **ระดับ:** Expert | **เวลาเรียน:** 80 นาที | **ขั้นตอนที่:** 1621–1660

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Apollo Router configuration
- Supergraph composition
- Query planning
- Subgraph communication
- Authentication ใน Router
- Rate limiting ใน Router
- Router plugins
- Schema Registry workflow

---

## 📌 Step 1621: Apollo Router Setup

```bash
# Download Apollo Router
curl -sSL https://router.apollo.dev/download/nix/latest | sh

# หรือ Docker
docker pull ghcr.io/apollographql/router:latest
```

```yaml
# router.yaml
supergraph:
  # Compose supergraph schema จาก subgraphs
  path: ./supergraph.graphql
  
server:
  port: 4000
  host: 0.0.0.0

# Subgraph overrides (routing URLs)
override_subgraph_url:
  users: http://users-service:4001/graphql
  products: http://products-service:4002/graphql
  orders: http://orders-service:4003/graphql

# Authentication
authentication:
  router:
    jwt:
      jwks:
        - url: https://auth.myapp.com/.well-known/jwks.json
          poll_interval: 60s
      
      # Extract claims ไป context
      map_claims:
        - field: sub
          key: userId
        - field: role
          key: userRole

# Authorization (coarse-grained)
authorization:
  preview_directives:
    enabled: true

# Caching
supergraph:
  query_planning:
    cache:
      in_memory:
        limit: 512
      redis:
        urls: ["redis://localhost:6379"]
        ttl: 24h

# Traffic management
traffic_shaping:
  global:
    deduplicate_variables: true
  router:
    timeout: 30s
  all:
    timeout: 25s
    retry:
      service_unavailable: true
      idempotency_mode: request

# Telemetry
telemetry:
  tracing:
    common:
      service_name: apollo-router
    otlp:
      endpoint: http://otel-collector:4317
      protocol: grpc
  metrics:
    prometheus:
      enabled: true
      listen: 0.0.0.0:9090
      path: /metrics
```

---

## 📌 Step 1622: Supergraph Composition

```yaml
# supergraph.yaml
federation_version: =2.6.0

subgraphs:
  users:
    routing_url: http://users-service/graphql
    schema:
      subgraph_url: http://users-service/graphql
  
  products:
    routing_url: http://products-service/graphql
    schema:
      subgraph_url: http://products-service/graphql
  
  orders:
    routing_url: http://orders-service/graphql
    schema:
      subgraph_url: http://orders-service/graphql
  
  reviews:
    routing_url: http://reviews-service/graphql
    schema:
      subgraph_url: http://reviews-service/graphql
```

```bash
# Compose supergraph
rover supergraph compose --config supergraph.yaml > supergraph.graphql

# Watch for changes (dev mode)
rover supergraph compose --config supergraph.yaml --watch

# Start router
./router --config router.yaml --supergraph supergraph.graphql

# Or with Docker
docker run -p 4000:4000 \
  -v $(pwd)/router.yaml:/dist/config/router.yaml \
  -v $(pwd)/supergraph.graphql:/dist/config/supergraph.graphql \
  ghcr.io/apollographql/router:latest \
  --config /dist/config/router.yaml \
  --supergraph /dist/config/supergraph.graphql
```

---

## 📌 Step 1623: Query Planning

```graphql
# Query ที่ client ส่ง:
query GetUserWithOrders($userId: ID!) {
  user(id: $userId) {    # ← Users subgraph
    id
    name
    email
    orders {             # ← Orders subgraph (extends User)
      id
      status
      total
      items {
        product {        # ← Products subgraph
          id
          name
          price
        }
        quantity
      }
    }
  }
}

# Query plan ที่ Router สร้าง:
# 1. Fetch user from Users subgraph
# 2. Parallel:
#    2a. Fetch user.orders from Orders subgraph (using user.id)
# 3. For each order.item: batch fetch products from Products subgraph
#    (DataLoader-style batching)
```

```typescript
// ดู query plan
// POST /graphql
// Header: Apollo-Query-Plan-Experimental: true

// Response includes:
// "extensions": {
//   "apolloQueryPlan": {
//     "object": {
//       "kind": "QueryPlan",
//       "node": {
//         "kind": "Sequence",
//         "nodes": [...]
//       }
//     }
//   }
// }
```

---

## 📌 Step 1624: Authorization Directives

```graphql
# schema.graphql (supergraph)

# Router-level authorization
type Query {
  # ต้องมี JWT token
  me: User @authenticated
  
  # ต้องมี role
  adminDashboard: AdminStats @requiresScopes(scopes: [["ADMIN"]])
  
  # Public
  products: [Product!]!
}

type Order {
  id: ID!
  items: [OrderItem!]!
  total: Float!
  
  # เฉพาะ admin เท่านั้น
  internalNotes: String @requiresScopes(scopes: [["ADMIN"]])
  
  # Owner หรือ admin
  customerDetails: CustomerInfo @policy(policies: ["owner-or-admin"])
}
```

```yaml
# router.yaml additions
authorization:
  preview_directives:
    enabled: true

# Custom policy
coprocessor:
  url: http://auth-service:4000/graphql-coprocessor
  router:
    request:
      headers: true
      body: true
  subgraph:
    all:
      request:
        headers: true
```

---

## 📌 Step 1625: Schema Registry Workflow

```bash
# CI/CD pipeline สำหรับ schema changes

# 1. Local development
rover subgraph check myapp@staging \
  --schema ./schema.graphql \
  --name users \
  --background  # ไม่ block CI

# 2. Check breaking changes
rover subgraph check myapp@prod \
  --schema ./schema.graphql \
  --name users

# Output:
# ✓ 0 schema changes detected
# ✓ Composition checks passed
# ✓ 0 operations broken

# 3. Publish on merge
rover subgraph publish myapp@prod \
  --schema ./schema.graphql \
  --name users \
  --routing-url https://users.myapp.com/graphql

# 4. Router auto-reloads when supergraph changes
# (polling Apollo Uplink every 10 seconds)
```

```yaml
# .github/workflows/schema-check.yml
name: Schema Check
on: [pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Rover
        run: curl -sSL https://rover.apollo.dev/nix/latest | sh
      
      - name: Schema Check
        run: |
          rover subgraph check ${{ vars.APOLLO_GRAPH_REF }} \
            --schema ./schema.graphql \
            --name ${{ vars.SUBGRAPH_NAME }}
        env:
          APOLLO_KEY: ${{ secrets.APOLLO_KEY }}

  publish:
    if: github.ref == 'refs/heads/main'
    needs: check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Publish Schema
        run: |
          rover subgraph publish ${{ vars.APOLLO_GRAPH_REF }} \
            --schema ./schema.graphql \
            --name ${{ vars.SUBGRAPH_NAME }} \
            --routing-url ${{ vars.ROUTING_URL }}
        env:
          APOLLO_KEY: ${{ secrets.APOLLO_KEY }}
```

---

## 📌 Step 1626: สรุป Part 068

### เนื้อหาที่เรียนรู้

✅ Apollo Router configuration  
✅ Supergraph composition  
✅ Query planning visualization  
✅ Router-level auth (JWT + directives)  
✅ Schema Registry workflow  
✅ CI/CD integration  

### Federation Architecture Summary

```
Development Flow:
1. Team writes subgraph schema
2. rover subgraph check → validate no breaking changes
3. Merge PR → CI publishes schema to Apollo Registry
4. Apollo Router auto-polls Registry → reloads supergraph

Production Setup:
- Apollo Router (no Node.js, written in Rust, very fast)
- Subgraphs: any language/framework (Apollo Server, Yoga, etc.)
- Schema Registry: Apollo GraphOS (or self-hosted)
- Metrics: Apollo Studio (or OpenTelemetry)
```

### ในส่วนถัดไป

➡️ **[Part 069](./part-069.md)** — Advanced GraphQL Security

---

*Part 068 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
