# Part 070 — Production-Grade Error Handling 🚨

> **ระดับ:** Expert | **เวลาเรียน:** 75 นาที | **ขั้นตอนที่:** 1701–1740

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Error taxonomy (technical vs user-facing)
- GraphQL error types
- Partial success pattern
- Error codes & localization
- Error monitoring (Sentry)
- Circuit breaker pattern
- Retry logic
- Error boundaries (client side)

---

## 📌 Step 1701: Error Taxonomy

```typescript
// src/errors/types.ts

// 3 ประเภทหลักของ errors ใน GraphQL:

// 1. User Errors — ปัญหาจาก input ของ user
//    → แสดง message ให้ user เห็น
//    → ไม่ log เป็น "error" (ปกติ)

// 2. Operational Errors — ปัญหาที่คาดได้ (network, DB timeout)
//    → Log เป็น "warn"
//    → Return generic message ให้ user

// 3. Programming Errors — bugs ในโค้ด
//    → Log เป็น "error" + alert
//    → Return generic "Something went wrong"

import { GraphQLError, GraphQLErrorExtensions } from 'graphql';

// Error codes สำหรับ clients
export enum ErrorCode {
  // Authentication
  UNAUTHENTICATED = 'UNAUTHENTICATED',
  FORBIDDEN = 'FORBIDDEN',
  
  // Validation
  VALIDATION_ERROR = 'VALIDATION_ERROR',
  INVALID_INPUT = 'INVALID_INPUT',
  
  // Business logic
  NOT_FOUND = 'NOT_FOUND',
  ALREADY_EXISTS = 'ALREADY_EXISTS',
  CONFLICT = 'CONFLICT',
  PRECONDITION_FAILED = 'PRECONDITION_FAILED',
  
  // Rate limiting
  RATE_LIMITED = 'RATE_LIMITED',
  QUOTA_EXCEEDED = 'QUOTA_EXCEEDED',
  
  // Operational
  SERVICE_UNAVAILABLE = 'SERVICE_UNAVAILABLE',
  UPSTREAM_ERROR = 'UPSTREAM_ERROR',
  
  // Programming
  INTERNAL_ERROR = 'INTERNAL_ERROR',
}

// Structured application errors
export class AppError extends GraphQLError {
  constructor(
    message: string,
    code: ErrorCode,
    options?: {
      userMessage?: string;      // localized, user-facing
      field?: string;            // which field caused the error
      metadata?: Record<string, unknown>;
      cause?: Error;
      httpStatus?: number;
    }
  ) {
    super(message, {
      extensions: {
        code,
        userMessage: options?.userMessage ?? message,
        field: options?.field,
        metadata: options?.metadata,
        httpStatus: options?.httpStatus,
      },
      originalError: options?.cause,
    });
    
    this.name = 'AppError';
  }
}

// Convenience constructors
export const Errors = {
  notFound: (resource: string, id: string) =>
    new AppError(
      `${resource} ${id} not found`,
      ErrorCode.NOT_FOUND,
      { userMessage: `${resource} not found`, httpStatus: 404 }
    ),
  
  forbidden: (action: string = '') =>
    new AppError(
      `Forbidden: ${action}`,
      ErrorCode.FORBIDDEN,
      { userMessage: 'You are not allowed to perform this action', httpStatus: 403 }
    ),
  
  unauthenticated: () =>
    new AppError(
      'Not authenticated',
      ErrorCode.UNAUTHENTICATED,
      { userMessage: 'Please log in to continue', httpStatus: 401 }
    ),
  
  validationError: (field: string, message: string) =>
    new AppError(
      `Validation failed: ${message}`,
      ErrorCode.VALIDATION_ERROR,
      { field, userMessage: message, httpStatus: 400 }
    ),
  
  serviceUnavailable: (service: string) =>
    new AppError(
      `Service unavailable: ${service}`,
      ErrorCode.SERVICE_UNAVAILABLE,
      { userMessage: 'Service temporarily unavailable, please try again', httpStatus: 503 }
    ),
};
```

---

## 📌 Step 1702: Error Formatting

```typescript
// src/server.ts

const server = new ApolloServer({
  typeDefs,
  resolvers,
  
  formatError: (formattedError, error) => {
    // 1. แยก Programming errors
    const isProgrammingError = !(error instanceof AppError) && 
                               !(error instanceof GraphQLError);
    
    if (isProgrammingError) {
      // Log ส่วนที่ sensitive
      console.error('PROGRAMMING ERROR:', {
        message: error instanceof Error ? error.message : String(error),
        stack: error instanceof Error ? error.stack : undefined,
        query: '<<query>>',  // ดึงจาก context
      });
      
      // ส่ง generic message ให้ client
      return {
        message: 'Internal server error',
        extensions: {
          code: ErrorCode.INTERNAL_ERROR,
          traceId: generateTraceId(),  // สำหรับ support
        },
        locations: undefined,
        path: formattedError.path,
      };
    }
    
    // 2. Operational errors: log เป็น warn
    const extensions = formattedError.extensions as GraphQLErrorExtensions;
    if ([ErrorCode.SERVICE_UNAVAILABLE, ErrorCode.UPSTREAM_ERROR].includes(
      extensions?.code as ErrorCode
    )) {
      console.warn('OPERATIONAL ERROR:', formattedError.message);
    }
    
    // 3. User errors: return as-is (sanitized)
    return {
      message: (extensions?.userMessage as string) ?? formattedError.message,
      extensions: {
        code: extensions?.code ?? 'UNKNOWN_ERROR',
        field: extensions?.field,
        // ไม่ return metadata ใน production
        ...(process.env.NODE_ENV !== 'production' && {
          originalMessage: formattedError.message,
          metadata: extensions?.metadata,
        }),
      },
      path: formattedError.path,
      // ซ่อน locations ใน production
      ...(process.env.NODE_ENV !== 'production' && {
        locations: formattedError.locations,
      }),
    };
  },
});
```

---

## 📌 Step 1703: Validation Errors with Zod

```typescript
// src/validation/schema.ts
import { z } from 'zod';
import { Errors, ErrorCode, AppError } from './errors.js';

// Product validation schema
export const CreateProductSchema = z.object({
  name: z.string().min(3).max(255),
  price: z.number().positive().max(9_999_999),
  description: z.string().max(5000).optional(),
  categoryId: z.string().uuid(),
  images: z.array(z.string().url()).min(1).max(10),
  stock: z.number().int().min(0),
});

// Helper: validate and throw GraphQL error
export function validateInput<T>(
  schema: z.ZodSchema<T>,
  input: unknown
): T {
  const result = schema.safeParse(input);
  
  if (!result.success) {
    const firstError = result.error.errors[0];
    throw new AppError(
      `Validation failed: ${firstError?.message}`,
      ErrorCode.VALIDATION_ERROR,
      {
        field: firstError?.path.join('.'),
        userMessage: firstError?.message ?? 'Invalid input',
        metadata: { errors: result.error.errors },
        httpStatus: 400,
      }
    );
  }
  
  return result.data;
}

// Usage in resolver
const createProduct = async (_, { input }, ctx) => {
  requireAuth(ctx);
  
  const validated = validateInput(CreateProductSchema, input);
  
  return ctx.prisma.product.create({ data: validated });
};
```

---

## 📌 Step 1704: Error Monitoring กับ Sentry

```bash
npm install @sentry/node @sentry/profiling-node
```

```typescript
// src/telemetry/sentry.ts
import * as Sentry from '@sentry/node';
import { nodeProfilingIntegration } from '@sentry/profiling-node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: process.env.APP_VERSION,
  
  integrations: [
    nodeProfilingIntegration(),
    Sentry.prismaIntegration(),
  ],
  
  // Sample rates
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  profilesSampleRate: 0.1,
  
  // ไม่ส่ง user data ที่ sensitive
  beforeSend(event) {
    if (event.user) {
      delete event.user.email;
      delete event.user.ip_address;
    }
    return event;
  },
});

// Apollo Server plugin สำหรับ Sentry
export const sentryPlugin = (): ApolloServerPlugin => ({
  async requestDidStart({ request, contextValue }) {
    return {
      async didEncounterErrors({ errors, contextValue }) {
        const ctx = contextValue as AppContext;
        
        for (const error of errors) {
          // ไม่ส่ง user errors ไป Sentry
          const isUserError = error.extensions?.code &&
            [ErrorCode.UNAUTHENTICATED, ErrorCode.FORBIDDEN, 
             ErrorCode.VALIDATION_ERROR, ErrorCode.NOT_FOUND].includes(
              error.extensions.code as ErrorCode
            );
          
          if (!isUserError) {
            Sentry.withScope((scope) => {
              scope.setUser({ id: ctx.user?.id });
              scope.setExtra('graphqlPath', error.path?.join('.'));
              scope.setExtra('query', request.query?.substring(0, 1000));
              
              Sentry.captureException(error.originalError ?? error);
            });
          }
        }
      },
    };
  },
});
```

---

## 📌 Step 1705: Circuit Breaker

```typescript
// src/resilience/circuit-breaker.ts
// npm install opossum

import CircuitBreaker from 'opossum';

// Circuit breaker สำหรับ external service calls
const paymentServiceBreaker = new CircuitBreaker(
  async (input: PaymentInput) => callPaymentService(input),
  {
    timeout: 3000,              // ถ้า > 3s = failure
    errorThresholdPercentage: 50,  // 50% failures → open circuit
    resetTimeout: 10000,        // ลอง reset หลัง 10s
    volumeThreshold: 10,        // ต้องมี 10 calls ก่อน trip
  }
);

paymentServiceBreaker.fallback(() => ({
  success: false,
  error: 'Payment service temporarily unavailable',
}));

paymentServiceBreaker.on('open', () => {
  console.warn('Payment service circuit breaker OPENED');
  metrics.circuitBreakerOpen.inc({ service: 'payment' });
});

paymentServiceBreaker.on('halfOpen', () => {
  console.info('Payment service circuit breaker HALF-OPEN');
});

paymentServiceBreaker.on('close', () => {
  console.info('Payment service circuit breaker CLOSED');
});

// ใช้ใน resolver
const checkout = async (_, { input }, ctx) => {
  const result = await paymentServiceBreaker.fire(input);
  
  if (!result.success) {
    throw Errors.serviceUnavailable('payment');
  }
  
  return result;
};
```

---

## 📌 Step 1706: สรุป Part 070

### เนื้อหาที่เรียนรู้

✅ Error taxonomy (user/operational/programming)  
✅ Structured error types with AppError  
✅ formatError สำหรับ production  
✅ Zod validation  
✅ Sentry integration  
✅ Circuit breaker pattern  

### Error Handling Decision Tree

```
Error occurred?
├─ User input invalid?    → ValidationError (400)
├─ Not authenticated?     → UnauthenticatedError (401)
├─ Not authorized?        → ForbiddenError (403)
├─ Resource not found?    → NotFoundError (404)
├─ External service down? → ServiceUnavailableError (503) + circuit breaker
└─ Bug in our code?       → InternalError (500) + Sentry alert
```

### ในส่วนถัดไป

➡️ **[Part 071](./part-071.md)** — GraphQL Testing Strategies

---

*Part 070 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
