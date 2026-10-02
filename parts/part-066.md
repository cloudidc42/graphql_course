# Part 066 — Observability: OpenTelemetry & Distributed Tracing 🔭

> **ระดับ:** Expert | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1541–1580

---

## 🎯 สิ่งที่จะได้เรียนรู้

- OpenTelemetry setup (traces, metrics, logs)
- Distributed tracing กับ Jaeger / Tempo
- GraphQL-specific instrumentation
- Service Level Objectives (SLOs)
- Error budgets
- Prometheus + Grafana dashboards
- Alert rules
- Log aggregation

---

## 📌 Step 1541: OpenTelemetry Setup

```bash
npm install \
  @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-otlp-http \
  @opentelemetry/instrumentation-http \
  @opentelemetry/instrumentation-express \
  @opentelemetry/instrumentation-pg \
  @opentelemetry/instrumentation-ioredis
```

```typescript
// src/telemetry/otel.ts
// ต้อง import ก่อน code อื่นทั้งหมด

import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-otlp-http';
import { OTLPMetricExporter } from '@opentelemetry/exporter-otlp-http';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { Resource } from '@opentelemetry/resources';
import { SEMRESATTRS_SERVICE_NAME, SEMRESATTRS_SERVICE_VERSION } from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: new Resource({
    [SEMRESATTRS_SERVICE_NAME]: 'graphql-api',
    [SEMRESATTRS_SERVICE_VERSION]: process.env.APP_VERSION ?? '0.0.0',
    environment: process.env.NODE_ENV ?? 'development',
  }),
  
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTLP_ENDPOINT ?? 'http://localhost:4318/v1/traces',
    headers: {
      'x-honeycomb-team': process.env.HONEYCOMB_API_KEY ?? '',
    },
  }),
  
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({
      url: process.env.OTLP_ENDPOINT ?? 'http://localhost:4318/v1/metrics',
    }),
    exportIntervalMillis: 15_000,
  }),
  
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': {
        ignoreIncomingRequestHook: (req) => {
          // ไม่ trace health checks
          return req.url === '/health' || req.url === '/ready';
        },
      },
      '@opentelemetry/instrumentation-fs': {
        enabled: false,  // ลด noise
      },
    }),
  ],
});

sdk.start();

process.on('SIGTERM', () => {
  sdk.shutdown()
    .then(() => console.log('Telemetry shutdown complete'))
    .catch(err => console.error('Telemetry shutdown error:', err));
});

export const tracer = trace.getTracer('graphql-api');
```

---

## 📌 Step 1542: GraphQL Tracing Plugin

```typescript
// src/telemetry/graphql-plugin.ts
import { Plugin } from '@apollo/server';
import { trace, context, SpanStatusCode } from '@opentelemetry/api';

export const graphqlTracingPlugin = (): Plugin => {
  const tracer = trace.getTracer('graphql');
  
  return {
    async requestDidStart(requestContext) {
      const { request } = requestContext;
      
      // Extract operation name
      const operationName = request.operationName ?? 'anonymous';
      const operationType = detectOperationType(request.query ?? '');
      
      // Start span
      const span = tracer.startSpan(`graphql.${operationType} ${operationName}`, {
        attributes: {
          'graphql.operation.name': operationName,
          'graphql.operation.type': operationType,
          'graphql.document': request.query?.substring(0, 1000) ?? '',
        },
      });
      
      // Store span in context for child spans
      const ctx = trace.setSpan(context.active(), span);
      
      return {
        async executionDidStart() {
          return {
            willResolveField({ info }) {
              // Trace each resolver
              const fieldName = `${info.parentType.name}.${info.fieldName}`;
              const fieldSpan = tracer.startSpan(`graphql.field ${fieldName}`, {
                attributes: {
                  'graphql.field.name': info.fieldName,
                  'graphql.field.type': info.parentType.name,
                },
              }, ctx);
              
              return (error) => {
                if (error) {
                  fieldSpan.setStatus({ code: SpanStatusCode.ERROR, message: String(error) });
                  fieldSpan.recordException(error as Error);
                }
                fieldSpan.end();
              };
            },
          };
        },
        
        async willSendResponse({ response }) {
          if (response.body.kind === 'single' && response.body.singleResult.errors?.length) {
            span.setStatus({ code: SpanStatusCode.ERROR });
            span.setAttribute('graphql.errors', JSON.stringify(response.body.singleResult.errors));
          }
          span.end();
        },
        
        async didEncounterErrors({ errors }) {
          span.recordException(new Error(errors.map(e => e.message).join('; ')));
          span.setStatus({ code: SpanStatusCode.ERROR });
        },
      };
    },
  };
};

function detectOperationType(query: string): string {
  if (query.trimStart().startsWith('mutation')) return 'mutation';
  if (query.trimStart().startsWith('subscription')) return 'subscription';
  return 'query';
}
```

---

## 📌 Step 1543: Prometheus Metrics

```typescript
// src/telemetry/metrics.ts
import { MeterProvider, PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { PrometheusExporter } from '@opentelemetry/exporter-prometheus';
import { metrics } from '@opentelemetry/api';

// Prometheus exporter สำหรับ scraping
const prometheusExporter = new PrometheusExporter({
  port: 9464,
  endpoint: '/metrics',
});

const meterProvider = new MeterProvider();
meterProvider.addMetricReader(prometheusExporter);
metrics.setGlobalMeterProvider(meterProvider);

const meter = metrics.getMeter('graphql-api');

// Custom metrics
export const graphqlMetrics = {
  // Query/mutation count
  operationCount: meter.createCounter('graphql_operations_total', {
    description: 'Total number of GraphQL operations',
  }),
  
  // Operation duration
  operationDuration: meter.createHistogram('graphql_operation_duration_seconds', {
    description: 'GraphQL operation duration in seconds',
    boundaries: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
  }),
  
  // Error count
  errorCount: meter.createCounter('graphql_errors_total', {
    description: 'Total number of GraphQL errors',
  }),
  
  // Active subscriptions
  activeSubscriptions: meter.createObservableGauge('graphql_active_subscriptions', {
    description: 'Number of active GraphQL subscriptions',
  }),
  
  // Cache hit/miss
  cacheHits: meter.createCounter('graphql_cache_hits_total'),
  cacheMisses: meter.createCounter('graphql_cache_misses_total'),
};

// Record in Apollo plugin
export const metricsPlugin = (): Plugin => ({
  async requestDidStart(ctx) {
    const start = Date.now();
    const operationName = ctx.request.operationName ?? 'anonymous';
    
    return {
      async willSendResponse({ response }) {
        const duration = (Date.now() - start) / 1000;
        const hasErrors = response.body.kind === 'single' && 
          (response.body.singleResult.errors?.length ?? 0) > 0;
        
        graphqlMetrics.operationCount.add(1, {
          operation: operationName,
          type: detectOperationType(ctx.request.query ?? ''),
        });
        
        graphqlMetrics.operationDuration.record(duration, {
          operation: operationName,
        });
        
        if (hasErrors) {
          graphqlMetrics.errorCount.add(1, { operation: operationName });
        }
      },
    };
  },
});
```

---

## 📌 Step 1544: SLO & Error Budget

```yaml
# prometheus/slo-rules.yml
groups:
  - name: graphql-api-slo
    rules:
      # SLO: 99.9% availability (Error budget: 8.76 hours/year)
      - alert: ErrorBudgetBurning
        expr: |
          (
            sum(rate(graphql_errors_total[1h]))
            /
            sum(rate(graphql_operations_total[1h]))
          ) > 0.001  # 0.1% error rate = burning budget
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Error budget burning fast"
          description: "Error rate {{ $value | humanizePercentage }} over last hour"
      
      # SLO: P95 latency < 500ms
      - alert: LatencySLOViolation
        expr: |
          histogram_quantile(0.95,
            sum(rate(graphql_operation_duration_seconds_bucket[5m])) by (le)
          ) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P95 latency above 500ms"
          description: "P95 latency: {{ $value | humanizeDuration }}"
      
      # Critical: P99 > 2s
      - alert: LatencyCritical
        expr: |
          histogram_quantile(0.99,
            sum(rate(graphql_operation_duration_seconds_bucket[5m])) by (le)
          ) > 2
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "P99 latency above 2 seconds"
          runbook: "https://runbooks.myapp.com/latency-high"
```

---

## 📌 Step 1545: Grafana Dashboard

```json
// dashboards/graphql-overview.json
{
  "title": "GraphQL API Overview",
  "panels": [
    {
      "title": "Request Rate",
      "type": "graph",
      "targets": [{
        "expr": "sum(rate(graphql_operations_total[5m])) by (type)",
        "legendFormat": "{{ type }}"
      }]
    },
    {
      "title": "Error Rate",
      "type": "stat",
      "targets": [{
        "expr": "sum(rate(graphql_errors_total[5m])) / sum(rate(graphql_operations_total[5m]))",
        "legendFormat": "Error Rate"
      }]
    },
    {
      "title": "P50 / P95 / P99 Latency",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.50, sum(rate(graphql_operation_duration_seconds_bucket[5m])) by (le))",
          "legendFormat": "P50"
        },
        {
          "expr": "histogram_quantile(0.95, sum(rate(graphql_operation_duration_seconds_bucket[5m])) by (le))",
          "legendFormat": "P95"
        },
        {
          "expr": "histogram_quantile(0.99, sum(rate(graphql_operation_duration_seconds_bucket[5m])) by (le))",
          "legendFormat": "P99"
        }
      ]
    },
    {
      "title": "Active Subscriptions",
      "type": "stat",
      "targets": [{
        "expr": "graphql_active_subscriptions"
      }]
    }
  ]
}
```

---

## 📌 Step 1546: สรุป Part 066

### เนื้อหาที่เรียนรู้

✅ OpenTelemetry SDK setup  
✅ GraphQL resolver tracing  
✅ Prometheus metrics  
✅ SLO definitions  
✅ Error budget alerting  
✅ Grafana dashboards  

### Observability Pillars

```
Traces   → ดู request journey ข้าม services
          Jaeger / Honeycomb / Tempo
          
Metrics  → ดู aggregate numbers
          Prometheus → Grafana
          
Logs     → ดู detailed events
          Loki → Grafana (or ELK stack)

The three must be CORRELATED:
- Trace ID ใน logs
- Exemplars ใน metrics → link to traces
```

### ในส่วนถัดไป

➡️ **[Part 067](./part-067.md)** — Apollo Federation: Subgraph Development

---

*Part 066 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
