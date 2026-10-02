# Part 095 — Production-Ready Checklist & Study Guide 📚

> **ระดับ:** Expert | **เวลาเรียน:** 70 นาที | **ขั้นตอนที่:** 2751–2795

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Production readiness checklist ครบถ้วน
- Performance targets
- Security audit checklist
- Operational runbooks
- Apollo certification tips
- Learning roadmap
- Resources สำหรับ deep-dive

---

## 📌 Step 2751: Production Readiness Checklist

```
=== SCHEMA DESIGN ===
□ Types มี descriptions ชัดเจน
□ Nullable vs Non-null ถูกต้อง (non-null ยาก → nullable ปลอดภัยกว่า)
□ Pagination: cursor-based สำหรับ large lists
□ Error: unions แทน exceptions สำหรับ business errors
□ Input validation: Zod หรือ class-validator
□ Enums มี values ที่ stable
□ IDs ใช้ ID scalar (ไม่ใช่ String)
□ Dates ใช้ String ISO 8601 หรือ custom scalar

=== PERFORMANCE ===
□ DataLoader ทุก N:1 / N:N relations
□ Select only needed fields (Prisma select)
□ Response cache สำหรับ public queries
□ APQ (Automatic Persisted Queries) enabled
□ Connection pooling (Prisma Accelerate / PgBouncer)
□ DB indexes บน filter/sort columns
□ Query complexity limit
□ Query depth limit
□ P95 < 200ms สำหรับ common operations
□ Load tested ก่อน launch

=== SECURITY ===
□ Introspection disabled ใน production
□ "Did you mean?" suggestions disabled
□ Rate limiting per user
□ Complexity limit
□ CSRF: Content-Type validation
□ CORS configured correctly
□ Input sanitization
□ SQL injection: parameterized queries / ORM
□ IDOR: owner checks ใน WHERE clause
□ Secrets ไม่อยู่ใน code (env vars)
□ HTTPS only
□ Dependency audit: npm audit

=== ERROR HANDLING ===
□ Production errors masked (ไม่ expose stack traces)
□ Error codes consistent (enum)
□ User-friendly messages
□ Internal errors → Sentry/logging
□ Circuit breakers สำหรับ external services

=== OBSERVABILITY ===
□ Structured logging (JSON)
□ Request ID ทุก request
□ Tracing (OpenTelemetry)
□ Metrics: latency P50/P95/P99 by operation
□ Metrics: error rate by operation
□ Alerting: error rate > 1%
□ Alerting: P95 > 500ms
□ Alerting: API down
□ Dashboard: Grafana

=== DEPLOYMENT ===
□ Health endpoint (/health, /ready)
□ Graceful shutdown handler
□ Docker multi-stage build
□ Non-root user ใน container
□ Resource limits (CPU/memory)
□ Horizontal scaling tested
□ Rolling deployment (ไม่มี downtime)
□ Blue/green หรือ canary deployment

=== TESTING ===
□ Unit tests: resolvers
□ Integration tests: full HTTP requests
□ Schema tests: validate ไม่มี breaking changes
□ Contract tests: client operations
□ Load tests: k6 หรือ Artillery
□ Coverage > 80%

=== DOCUMENTATION ===
□ SDL descriptions บน types/fields
□ @deprecated บน deprecated fields
□ Changelog update
□ Client migration guides
□ Runbooks สำหรับ common incidents
```

---

## 📌 Step 2752: Performance Targets

```
Latency (P95):
  Simple query (cache hit):       < 20ms
  Simple query (DB):              < 50ms
  Complex query (joins):          < 200ms
  Mutation (write):               < 300ms
  File upload (confirm):          < 500ms
  Report generation:              async (background job)

Throughput:
  API Gateway:                    10,000+ RPS
  GraphQL server:                  2,000+ RPS per instance
  Subscription connections:       10,000+ per instance (WS)

Availability:
  SLO:                            99.9% (< 8.7 hours downtime/year)
  Error rate:                     < 0.1% (99.9th percentile)
  
Cache Hit Rate:
  Public content:                 > 80%
  DataLoader:                     > 60% (within request)

Database:
  Query time P95:                 < 50ms
  Connection pool utilization:    < 80%
  Slow queries:                   < 1% > 100ms
```

---

## 📌 Step 2753: Incident Runbooks

```markdown
# Runbook: High Error Rate

## Symptoms
- Alert: GraphQLHighErrorRate fired
- Error rate > 5%

## Immediate Steps (< 5 นาที)
1. Check Grafana: errors by operation
   → หา operation ที่ error สูง
2. Check logs: `kubectl logs -l app=graphql-api | grep "ERROR"`
3. Check DB: pg_stat_activity สำหรับ long queries
4. Check Redis: INFO stats สำหรับ memory/connections

## Common Causes
A) DB connection exhausted
   → pg_stat_activity > maxConnections
   → Fix: scale API pods, หรือ restart connection pool

B) Upstream service down
   → Circuit breaker open
   → Fix: check upstream health, wait for recovery

C) Schema deployment ที่ introduce bugs
   → Recent deployment?
   → Fix: rollback deployment

D) Traffic spike
   → Requests/second สูงผิดปกติ
   → Fix: enable rate limiting, scale pods

## Escalation
- > 10 นาที: Notify on-call engineer
- > 30 นาที: Engineering lead
- > 1 ชั่วโมง: CTO notification
```

---

## 📌 Step 2754: Learning Roadmap

```
ระดับ Beginner (1-3 เดือน):
✅ Parts 001-020: GraphQL Basics
✅ Parts 021-040: Queries, Mutations, Subscriptions
✅ Parts 041-060: Authentication, DataLoader, Testing

ระดับ Intermediate (3-6 เดือน):
✅ Parts 061-070: Performance, Deployment, Security
✅ Parts 071-080: Scale, Documentation, Migration

ระดับ Expert (6-12 เดือน):
✅ Parts 081-090: Monitoring, Mobile, TypeScript, Edge
✅ Parts 091-095: Message Queues, Code-First, OSS, Interviews

ระดับ World-Class (ongoing):
✅ Parts 096-100: Architecture Case Studies + Capstone

Additional Resources:
□ graphql.org/learn
□ Apollo Odyssey (free courses)
□ The Guild blog
□ How To GraphQL
□ Production Ready GraphQL (book)
□ GraphQL Conf recordings (YouTube)

Practice:
□ Build: simple blog API → e-commerce → real-time chat
□ Contribute: fix a bug ใน graphql-js หรือ apollo server
□ Write: blog post เกี่ยวกับ pattern ที่ได้เรียน
□ Speak: present ใน meetup
```

---

## 📌 Step 2755: สรุป Part 095

### เนื้อหาที่เรียนรู้

✅ Production readiness checklist ครบถ้วน  
✅ Performance targets  
✅ Incident runbooks  
✅ Learning roadmap  

### สิ่งที่ทำให้ GraphQL ดีมาก

```
ในมุมของ Developer:
✅ Type safety end-to-end
✅ Self-documenting API
✅ Great tooling (codegen, Apollo Studio)
✅ Flexibility for clients

ในมุมของ Organization:
✅ Federation: team autonomy
✅ Schema registry: governance
✅ Analytics: field usage tracking
✅ Backward compatibility: deprecation workflow

ในมุมของ Operations:
✅ Operation-level monitoring
✅ Declarative: easier to analyze queries
✅ Persisted queries: CDN-friendly
✅ APQ: reduces payload size
```

### ในส่วนถัดไป

➡️ **[Part 096](./part-096.md)** — Case Study: Netflix-Scale Architecture

---

*Part 095 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
