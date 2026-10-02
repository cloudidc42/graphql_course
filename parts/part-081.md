# Part 081 — GraphQL Monitoring & Alerting 📊

> **ระดับ:** Expert | **เวลาเรียน:** 75 นาที | **ขั้นตอนที่:** 2131–2165

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Apollo Studio Operations
- Field-level analytics
- Slow operations dashboard
- Error rate monitoring
- Client-version analytics
- Alert rules & PagerDuty
- Custom Grafana dashboards
- SLO tracking

---

## 📌 Step 2131: Apollo Studio Integration

```typescript
// src/server.ts
import { ApolloServerPluginUsageReporting } from '@apollo/server/plugin/usageReporting';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    ApolloServerPluginUsageReporting({
      // Apollo Studio API key
      // APOLLO_KEY=service:my-graph:key... (set in env)
      
      sendVariableValues: {
        // ไม่ส่ง variables ที่ sensitive ไป Apollo Studio
        exceptNames: ['password', 'token', 'creditCard', 'ssn'],
      },
      
      sendHeaders: {
        // ส่งแค่ headers ที่ไม่ sensitive
        onlyNames: ['x-client-version', 'x-client-name', 'apollographql-client-version'],
      },
      
      // Sampling: ใน production อาจ sample 10%
      reportSchema: true,
      requestCount: true,
      fieldLevelInstrumentation: process.env.NODE_ENV === 'development' ? 1.0 : 0.1,
      
      // Custom key สำหรับ metrics
      generateClientInfo: ({ request }) => ({
        clientName: request.http?.headers.get('apollographql-client-name') ?? 'unknown',
        clientVersion: request.http?.headers.get('apollographql-client-version') ?? '0.0.0',
      }),
    }),
  ],
});
```

---

## 📌 Step 2132: Custom Operation Analytics

```typescript
// src/analytics/operation-analytics.ts
// ถ้าไม่ใช้ Apollo Studio → สร้าง custom analytics

interface OperationStat {
  operationName: string;
  operationType: 'query' | 'mutation' | 'subscription';
  duration: number;
  cacheHit: boolean;
  errorCount: number;
  clientName: string;
  fieldCount: number;
  timestamp: Date;
}

// ClickHouse หรือ TimescaleDB สำหรับ analytics data
class OperationAnalytics {
  async record(stat: OperationStat) {
    // Insert into timeseries DB
    await clickhouse.insert({
      table: 'graphql_operations',
      values: [stat],
      format: 'JSONEachRow',
    });
  }
  
  async getSlowOperations(options: {
    threshold: number;   // ms
    since: Date;
    limit: number;
  }) {
    return clickhouse.query({
      query: `
        SELECT 
          operationName,
          quantile(0.95)(duration) AS p95,
          quantile(0.99)(duration) AS p99,
          count() AS callCount,
          countIf(errorCount > 0) AS errorCount
        FROM graphql_operations
        WHERE timestamp >= {since: DateTime}
          AND duration >= {threshold: Float64}
        GROUP BY operationName
        ORDER BY p99 DESC
        LIMIT {limit: UInt32}
      `,
      query_params: options,
      format: 'JSONEachRow',
    });
  }
  
  async getFieldUsage(since: Date) {
    // ดู fields ที่ถูกใช้บ่อย
    return clickhouse.query({
      query: `
        SELECT
          arrayJoin(fieldsUsed) AS field,
          count() AS usageCount,
          uniqExact(clientName) AS uniqueClients
        FROM graphql_operations
        WHERE timestamp >= {since: DateTime}
        GROUP BY field
        ORDER BY usageCount DESC
      `,
      query_params: { since },
      format: 'JSONEachRow',
    });
  }
}
```

---

## 📌 Step 2133: Alert Rules

```yaml
# prometheus/alerts/graphql.yml
groups:
  - name: graphql-api
    rules:
      # 1. Error rate สูงเกิน threshold
      - alert: GraphQLHighErrorRate
        expr: |
          (
            sum(rate(graphql_errors_total[5m])) /
            sum(rate(graphql_operations_total[5m]))
          ) > 0.05
        for: 2m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "GraphQL error rate {{ $value | humanizePercentage }}"
          runbook: "https://runbooks.myapp.com/graphql-errors"
      
      # 2. P95 latency สูง
      - alert: GraphQLHighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(graphql_operation_duration_seconds_bucket[5m])) by (le, operation)
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "GraphQL P95 latency {{ $value }}s for {{ $labels.operation }}"
      
      # 3. API down
      - alert: GraphQLAPIDown
        expr: up{job="graphql-api"} == 0
        for: 1m
        labels:
          severity: critical
          pagerduty: "true"
        annotations:
          summary: "GraphQL API is down"
      
      # 4. Subscription connections spike (possible DDoS)
      - alert: GraphQLSubscriptionSpike
        expr: graphql_active_subscriptions > 10000
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Active subscriptions: {{ $value }}"
      
      # 5. Rate limit hit บ่อย
      - alert: GraphQLRateLimitExceeded
        expr: |
          sum(rate(graphql_rate_limited_total[5m])) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High rate limit hits: {{ $value }}/s"
```

---

## 📌 Step 2134: Grafana Dashboard สำหรับ Operations

```json
// Grafana panel: Top Slow Operations
{
  "title": "Top 10 Slow Operations (P99)",
  "type": "table",
  "datasource": "prometheus",
  "targets": [{
    "expr": "topk(10, histogram_quantile(0.99, sum(rate(graphql_operation_duration_seconds_bucket[1h])) by (le, operation)))",
    "legendFormat": "{{operation}}"
  }],
  "options": {
    "sortBy": [{ "displayName": "Value", "desc": true }]
  }
}

// Grafana panel: Error rate by operation
{
  "title": "Error Rate by Operation",
  "type": "heatmap",
  "datasource": "prometheus",
  "targets": [{
    "expr": "sum(rate(graphql_errors_total[5m])) by (operation) / sum(rate(graphql_operations_total[5m])) by (operation)",
    "legendFormat": "{{operation}}"
  }]
}
```

---

## 📌 Step 2135: สรุป Part 081

### เนื้อหาที่เรียนรู้

✅ Apollo Studio usage reporting  
✅ Custom operation analytics  
✅ ClickHouse สำหรับ analytics  
✅ Prometheus alert rules  
✅ Grafana dashboards  

### Monitoring Dashboard Checklist

```
Golden Signals:
□ Latency (P50, P95, P99 by operation)
□ Traffic (requests/second)
□ Errors (error rate by operation)
□ Saturation (CPU, memory, connections)

GraphQL-Specific:
□ Cache hit/miss rate
□ Slow operation alerts (P99 > 2s)
□ Deprecated field usage trending
□ Client version breakdown
□ Subscription connection count
□ DataLoader hit/miss rate

Alerts:
□ Error rate > 1% → warning
□ Error rate > 5% → critical + PagerDuty
□ P95 > 500ms → warning
□ P99 > 2s → critical
□ API down → critical + immediate
```

### ในส่วนถัดไป

➡️ **[Part 082](./part-082.md)** — GraphQL with Mobile: iOS & Android

---

*Part 081 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
