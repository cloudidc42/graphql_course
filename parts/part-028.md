# Part 028 — Monitoring & Observability 📊

> **ระดับ:** Advanced | **เวลาเรียน:** 120 นาที | **ขั้นตอนที่:** 1001–1040

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Apollo Studio + Schema Registry
- Metrics ด้วย Prometheus
- Tracing ด้วย OpenTelemetry
- Logging ด้วย Winston + Pino
- Error tracking ด้วย Sentry
- Grafana dashboards
- Alerts
- GraphQL-specific metrics

---

## 📌 Step 1001: Apollo Studio Setup

```javascript
// src/server.js
import { ApolloServer } from '@apollo/server';
import { ApolloServerPluginUsageReporting } from '@apollo/server/plugin/usageReporting';
import { ApolloServerPluginSchemaReporting } from '@apollo/server/plugin/schemaReporting';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    // ส่ง metrics ไป Apollo Studio
    ApolloServerPluginUsageReporting({
      sendVariableValues: { onlyNames: ['userId', 'productId'] }, // ส่ง variables บางตัว
      sendHeaders: { onlyNames: ['user-agent'] },
      rewriteError: (error) => {
        // ซ่อน sensitive info
        if (error.message.includes('password')) {
          return { ...error, message: '[REDACTED]' };
        }
        return error;
      },
    }),
    
    // ส่ง schema ไป Apollo Studio
    ApolloServerPluginSchemaReporting(),
  ],
});
```

---

## 📌 Step 1002: Prometheus Metrics

```javascript
// src/plugins/metrics.plugin.js
import { register, Counter, Histogram, Gauge } from 'prom-client';

// Metrics
export const requestTotal = new Counter({
  name: 'graphql_requests_total',
  help: 'Total GraphQL requests',
  labelNames: ['operation', 'type', 'status'],
});

export const requestDuration = new Histogram({
  name: 'graphql_request_duration_seconds',
  help: 'GraphQL request duration',
  labelNames: ['operation', 'type'],
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5],
});

export const activeSubscriptions = new Gauge({
  name: 'graphql_active_subscriptions',
  help: 'Active GraphQL subscriptions',
});

export const resolverDuration = new Histogram({
  name: 'graphql_resolver_duration_seconds',
  help: 'GraphQL resolver duration',
  labelNames: ['type', 'field'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5],
});

export const metricsPlugin = {
  async requestDidStart({ request }) {
    const startTime = process.hrtime.bigint();
    const operation = request.operationName || 'anonymous';
    const type = request.query?.trim().startsWith('mutation') ? 'mutation' :
                 request.query?.trim().startsWith('subscription') ? 'subscription' : 'query';
    
    return {
      async willSendResponse({ response }) {
        const duration = Number(process.hrtime.bigint() - startTime) / 1e9;
        const hasErrors = !!response.body.singleResult?.errors?.length;
        
        requestTotal.inc({
          operation,
          type,
          status: hasErrors ? 'error' : 'success',
        });
        
        requestDuration.observe({ operation, type }, duration);
      },
      
      async executionDidStart() {
        return {
          willResolveField({ info }) {
            const resolveStart = process.hrtime.bigint();
            
            return () => {
              const resolveDuration = Number(process.hrtime.bigint() - resolveStart) / 1e9;
              
              if (resolveDuration > 0.1) { // Log slow resolvers (>100ms)
                resolverDuration.observe(
                  { type: info.parentType.name, field: info.fieldName },
                  resolveDuration
                );
              }
            };
          },
        };
      },
    };
  },
};

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

---

## 📌 Step 1003: OpenTelemetry Tracing

```javascript
// src/telemetry.js
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'graphql-api',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION,
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV,
  }),
  
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://localhost:4318/v1/traces',
  }),
  
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': { enabled: true },
      '@opentelemetry/instrumentation-pg': { enabled: true },
      '@opentelemetry/instrumentation-redis': { enabled: true },
    }),
  ],
});

sdk.start();
```

### GraphQL Tracing Plugin

```javascript
// src/plugins/tracing.plugin.js
import opentelemetry from '@opentelemetry/api';

const tracer = opentelemetry.trace.getTracer('graphql');

export const tracingPlugin = {
  async requestDidStart({ request }) {
    const span = tracer.startSpan(`graphql.${request.operationName || 'anonymous'}`, {
      attributes: {
        'graphql.operation.name': request.operationName,
        'graphql.operation.type': detectOperationType(request.query),
      },
    });
    
    return {
      async willSendResponse({ response }) {
        if (response.body.singleResult?.errors?.length) {
          span.setStatus({ code: 2, message: 'GraphQL errors' });
        }
        span.end();
      },
      
      async executionDidStart() {
        return {
          willResolveField({ info }) {
            const resolverSpan = tracer.startSpan(
              `resolver.${info.parentType.name}.${info.fieldName}`
            );
            
            return (error) => {
              if (error) {
                resolverSpan.recordException(error);
                resolverSpan.setStatus({ code: 2 });
              }
              resolverSpan.end();
            };
          },
        };
      },
    };
  },
};

function detectOperationType(query = '') {
  if (query.trim().startsWith('mutation')) return 'mutation';
  if (query.trim().startsWith('subscription')) return 'subscription';
  return 'query';
}
```

---

## 📌 Step 1004: Structured Logging ด้วย Pino

```javascript
// src/logger.js
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  
  // Production: JSON format
  // Development: pretty format
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined,
  
  base: {
    service: 'graphql-api',
    version: process.env.APP_VERSION,
    env: process.env.NODE_ENV,
  },
  
  serializers: {
    req: (req) => ({
      method: req.method,
      url: req.url,
      remoteAddress: req.remoteAddress,
      userAgent: req.headers['user-agent'],
    }),
    err: pino.stdSerializers.err,
  },
});

// GraphQL request logging plugin
export const loggingPlugin = {
  async requestDidStart({ request, contextValue }) {
    const reqLogger = logger.child({
      operation: request.operationName,
      userId: contextValue.user?.sub,
    });
    
    reqLogger.info({ query: request.query?.slice(0, 200) }, 'GraphQL request');
    
    return {
      async willSendResponse({ response }) {
        const hasErrors = !!response.body.singleResult?.errors?.length;
        
        if (hasErrors) {
          reqLogger.warn(
            { errors: response.body.singleResult.errors },
            'GraphQL request completed with errors'
          );
        }
      },
      
      async didEncounterErrors({ errors }) {
        for (const error of errors) {
          if (!error.originalError || error.originalError instanceof GraphQLError) {
            reqLogger.warn({ error: error.message }, 'GraphQL client error');
          } else {
            reqLogger.error(
              { error, stack: error.originalError?.stack },
              'Unexpected server error'
            );
          }
        }
      },
    };
  },
};
```

---

## 📌 Step 1005: Sentry Error Tracking

```javascript
// src/sentry.js
import * as Sentry from '@sentry/node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: process.env.APP_VERSION,
  
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  
  beforeSend(event, hint) {
    const error = hint.originalException;
    
    // ไม่ส่ง expected errors ไป Sentry
    if (error instanceof GraphQLError) {
      const code = error.extensions?.code;
      const expectedCodes = ['UNAUTHENTICATED', 'FORBIDDEN', 'NOT_FOUND', 'BAD_USER_INPUT'];
      if (expectedCodes.includes(code)) return null;
    }
    
    return event;
  },
});

// ใน formatError
formatError: (formattedError, error) => {
  const originalError = error.originalError;
  
  if (originalError && !(originalError instanceof GraphQLError)) {
    Sentry.captureException(originalError, {
      tags: {
        graphql_path: formattedError.path?.join('.'),
      },
    });
  }
  
  return formattedError;
},
```

---

## 📌 Step 1006: Grafana Dashboard

```json
// grafana-dashboard.json (import ใน Grafana)
{
  "title": "GraphQL API Dashboard",
  "panels": [
    {
      "title": "Request Rate",
      "type": "stat",
      "targets": [
        {
          "expr": "sum(rate(graphql_requests_total[5m]))",
          "legendFormat": "req/s"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate(graphql_requests_total{status='error'}[5m])) / sum(rate(graphql_requests_total[5m])) * 100",
          "legendFormat": "Error %"
        }
      ]
    },
    {
      "title": "P95 Request Duration",
      "type": "timeseries",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, sum(rate(graphql_request_duration_seconds_bucket[5m])) by (le, operation))",
          "legendFormat": "{{operation}}"
        }
      ]
    },
    {
      "title": "Active Subscriptions",
      "type": "stat",
      "targets": [
        {
          "expr": "graphql_active_subscriptions"
        }
      ]
    }
  ]
}
```

---

## 📌 Step 1007: สรุป Part 028

### เนื้อหาที่เรียนรู้

✅ Apollo Studio metrics  
✅ Prometheus metrics  
✅ OpenTelemetry tracing  
✅ Structured logging ด้วย Pino  
✅ Sentry error tracking  
✅ Grafana dashboard  

### Homework

1. Setup Prometheus + Grafana สำหรับ GraphQL server
2. เพิ่ม tracing ใน slow resolvers
3. Configure Sentry alerts สำหรับ error spikes

### ในส่วนถัดไป

➡️ **[Part 029](./part-029.md)** — GraphQL Schema Versioning & Evolution

---

*Part 028 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
