# Part 012 — Error Handling: Production-Grade ⚠️

> **ระดับ:** Intermediate | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 331–370

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Error model ของ GraphQL
- GraphQLError และ extensions
- Custom error classes
- formatError สำหรับ production
- Error masking
- Union-based error handling
- Validation errors
- Logging errors
- Client-side error handling

---

## 📌 Step 331: GraphQL Error Model

GraphQL response มีทั้ง `data` และ `errors` ได้พร้อมกัน:

```json
{
  "data": {
    "user": null,
    "posts": [{ "id": "1", "title": "Hello" }]
  },
  "errors": [
    {
      "message": "User not found",
      "locations": [{ "line": 2, "column": 3 }],
      "path": ["user"],
      "extensions": {
        "code": "NOT_FOUND",
        "id": "user_999"
      }
    }
  ]
}
```

### Partial Success

```graphql
query {
  user(id: "999") {    # ← นี่ fail
    name
  }
  posts {              # ← นี่สำเร็จ
    title
  }
}
```

---

## 📌 Step 332: GraphQLError

```javascript
import { GraphQLError } from 'graphql';

// Basic error
throw new GraphQLError('Something went wrong');

// Error with extensions
throw new GraphQLError('User not found', {
  extensions: {
    code: 'NOT_FOUND',
    http: { status: 404 },
  },
});

// Error with path (usually auto-set by GraphQL)
throw new GraphQLError('Access denied', {
  extensions: {
    code: 'FORBIDDEN',
    http: { status: 403 },
  },
});
```

### Error Codes มาตรฐาน

```javascript
// Apollo Error Codes (convention)
const ERROR_CODES = {
  // Authentication
  UNAUTHENTICATED: 'UNAUTHENTICATED',    // 401
  FORBIDDEN: 'FORBIDDEN',                 // 403
  
  // Resource
  NOT_FOUND: 'NOT_FOUND',                 // 404
  CONFLICT: 'CONFLICT',                   // 409
  
  // Input
  BAD_USER_INPUT: 'BAD_USER_INPUT',       // 400
  VALIDATION_ERROR: 'VALIDATION_ERROR',   // 422
  
  // System
  INTERNAL_SERVER_ERROR: 'INTERNAL_SERVER_ERROR',  // 500
  SERVICE_UNAVAILABLE: 'SERVICE_UNAVAILABLE',       // 503
  
  // Business logic
  RATE_LIMITED: 'RATE_LIMITED',           // 429
  PAYMENT_REQUIRED: 'PAYMENT_REQUIRED',   // 402
};
```

---

## 📌 Step 333: Custom Error Classes

```javascript
// src/errors/index.js
import { GraphQLError } from 'graphql';

export class AuthenticationError extends GraphQLError {
  constructor(message = 'You must be logged in') {
    super(message, {
      extensions: {
        code: 'UNAUTHENTICATED',
        http: { status: 401 },
      },
    });
    this.name = 'AuthenticationError';
  }
}

export class ForbiddenError extends GraphQLError {
  constructor(message = 'Access denied', { role } = {}) {
    super(message, {
      extensions: {
        code: 'FORBIDDEN',
        http: { status: 403 },
        ...(role && { requiredRole: role }),
      },
    });
    this.name = 'ForbiddenError';
  }
}

export class NotFoundError extends GraphQLError {
  constructor(resource, id) {
    super(`${resource} with id "${id}" not found`, {
      extensions: {
        code: 'NOT_FOUND',
        http: { status: 404 },
        resource,
        id,
      },
    });
    this.name = 'NotFoundError';
  }
}

export class ValidationError extends GraphQLError {
  constructor(message, { field, errors } = {}) {
    super(message, {
      extensions: {
        code: 'VALIDATION_ERROR',
        http: { status: 422 },
        ...(field && { field }),
        ...(errors && { errors }),
      },
    });
    this.name = 'ValidationError';
  }
}

export class ConflictError extends GraphQLError {
  constructor(message, { conflictOn } = {}) {
    super(message, {
      extensions: {
        code: 'CONFLICT',
        http: { status: 409 },
        ...(conflictOn && { conflictOn }),
      },
    });
    this.name = 'ConflictError';
  }
}

export class RateLimitError extends GraphQLError {
  constructor(resetIn) {
    super(`Too many requests. Try again in ${resetIn} seconds.`, {
      extensions: {
        code: 'RATE_LIMITED',
        http: { status: 429 },
        resetIn,
      },
    });
    this.name = 'RateLimitError';
  }
}
```

### ใช้ Custom Errors ใน Resolvers

```javascript
import {
  AuthenticationError,
  ForbiddenError,
  NotFoundError,
  ValidationError,
  ConflictError,
} from '../errors/index.js';

const resolvers = {
  Query: {
    user: async (_, { id }, { user: currentUser }) => {
      if (!currentUser) throw new AuthenticationError();
      
      const user = await User.findById(id);
      if (!user) throw new NotFoundError('User', id);
      
      // ดู profile ของตัวเองหรือ admin เท่านั้น
      if (user.id !== currentUser.id && currentUser.role !== 'ADMIN') {
        throw new ForbiddenError('Cannot view other users profiles');
      }
      
      return user;
    },
  },
  
  Mutation: {
    createUser: async (_, { input }) => {
      // Validate
      if (input.password.length < 8) {
        throw new ValidationError('Password too short', {
          field: 'password',
          errors: { password: ['Must be at least 8 characters'] },
        });
      }
      
      // Check duplicate
      const existing = await User.findByEmail(input.email);
      if (existing) {
        throw new ConflictError(`Email ${input.email} is already taken`, {
          conflictOn: 'email',
        });
      }
      
      return User.create(input);
    },
  },
};
```

---

## 📌 Step 334: formatError — Production Error Masking

```javascript
// src/server.js
const server = new ApolloServer({
  typeDefs,
  resolvers,
  
  formatError: (formattedError, error) => {
    // Log ทุก error ใน server
    console.error('GraphQL Error:', {
      message: error.message,
      extensions: formattedError.extensions,
      path: formattedError.path,
      stack: error.originalError?.stack,
    });
    
    // Production: ซ่อน internal errors
    if (process.env.NODE_ENV === 'production') {
      // Internal errors — ซ่อน details
      if (!formattedError.extensions?.code ||
          formattedError.extensions.code === 'INTERNAL_SERVER_ERROR') {
        return {
          message: 'An internal error occurred',
          extensions: {
            code: 'INTERNAL_SERVER_ERROR',
          },
        };
      }
    }
    
    return formattedError;
  },
});
```

### Advanced formatError

```javascript
formatError: (formattedError, error) => {
  const originalError = error.originalError;
  
  // Log to monitoring service
  if (originalError && !(originalError instanceof GraphQLError)) {
    // Unexpected error — log with full context
    logger.error('Unexpected GraphQL error', {
      message: originalError.message,
      stack: originalError.stack,
      path: formattedError.path,
    });
    
    // Sentry
    Sentry.captureException(originalError, {
      contexts: {
        graphql: {
          path: formattedError.path?.join('.'),
          query: formattedError.extensions?.query,
        },
      },
    });
  }
  
  // Development — return full error
  if (process.env.NODE_ENV !== 'production') {
    return {
      ...formattedError,
      stack: originalError?.stack,
    };
  }
  
  // Production — mask internals
  const code = formattedError.extensions?.code;
  const safeErrors = [
    'UNAUTHENTICATED',
    'FORBIDDEN',
    'NOT_FOUND',
    'BAD_USER_INPUT',
    'VALIDATION_ERROR',
    'CONFLICT',
    'RATE_LIMITED',
  ];
  
  if (!safeErrors.includes(code)) {
    return {
      message: 'An error occurred',
      extensions: { code: 'INTERNAL_SERVER_ERROR' },
    };
  }
  
  return formattedError;
},
```

---

## 📌 Step 335: Union-based Error Handling

แนวคิด: แทนที่จะใช้ errors array — encode errors ใน type system:

```graphql
# ✅ Union Result Pattern
type CreateUserSuccess {
  user: User!
  message: String!
}

type CreateUserError {
  message: String!
  code: String!
  field: String
}

union CreateUserResult = CreateUserSuccess | CreateUserError

type Mutation {
  createUser(input: CreateUserInput!): CreateUserResult!
}
```

### Resolver สำหรับ Union Result

```javascript
const resolvers = {
  CreateUserResult: {
    __resolveType(obj) {
      return obj.__typename;
    },
  },
  
  Mutation: {
    createUser: async (_, { input }) => {
      // Validate email format
      if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(input.email)) {
        return {
          __typename: 'CreateUserError',
          message: 'Invalid email format',
          code: 'VALIDATION_ERROR',
          field: 'email',
        };
      }
      
      // Check duplicate
      const existing = await User.findByEmail(input.email);
      if (existing) {
        return {
          __typename: 'CreateUserError',
          message: 'Email already exists',
          code: 'CONFLICT',
          field: 'email',
        };
      }
      
      // Success
      const user = await User.create(input);
      return {
        __typename: 'CreateUserSuccess',
        user,
        message: 'User created successfully',
      };
    },
  },
};
```

### Client Query

```graphql
mutation CreateUser($input: CreateUserInput!) {
  createUser(input: $input) {
    __typename
    ... on CreateUserSuccess {
      user {
        id
        name
        email
      }
      message
    }
    ... on CreateUserError {
      message
      code
      field
    }
  }
}
```

### Generic Error Pattern

```graphql
# Generic MutationResult
interface MutationResult {
  success: Boolean!
  message: String
}

type UserMutationSuccess implements MutationResult {
  success: Boolean!
  message: String
  user: User!
}

type MutationError implements MutationResult {
  success: Boolean!
  message: String!
  code: String!
  fieldErrors: [FieldError!]
}

type FieldError {
  field: String!
  message: String!
}
```

---

## 📌 Step 336: Validation Error Handling

```javascript
// src/validators/user.validator.js
import { z } from 'zod';
import { ValidationError } from '../errors/index.js';

const CreateUserSchema = z.object({
  name: z.string()
    .min(2, 'Name must be at least 2 characters')
    .max(100, 'Name must not exceed 100 characters'),
  email: z.string()
    .email('Invalid email format'),
  password: z.string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
    .regex(/[0-9]/, 'Password must contain at least one number'),
  age: z.number()
    .int()
    .min(18, 'Must be at least 18 years old')
    .max(120)
    .optional(),
});

export function validateCreateUser(input) {
  const result = CreateUserSchema.safeParse(input);
  
  if (!result.success) {
    const fieldErrors = result.error.flatten().fieldErrors;
    
    throw new ValidationError('Input validation failed', {
      errors: Object.fromEntries(
        Object.entries(fieldErrors).map(([field, messages]) => [
          field,
          messages[0], // first error per field
        ])
      ),
    });
  }
  
  return result.data;
}
```

### Error Response สำหรับ Validation

```json
{
  "errors": [
    {
      "message": "Input validation failed",
      "extensions": {
        "code": "VALIDATION_ERROR",
        "errors": {
          "name": "Name must be at least 2 characters",
          "password": "Password must contain at least one uppercase letter",
          "age": "Must be at least 18 years old"
        }
      }
    }
  ]
}
```

---

## 📌 Step 337: Client-side Error Handling

```javascript
// Apollo Client error handling
import { ApolloClient, InMemoryCache, HttpLink, ApolloLink } from '@apollo/client';
import { onError } from '@apollo/client/link/error';

// Global error handler
const errorLink = onError(({ graphQLErrors, networkError, operation }) => {
  if (graphQLErrors) {
    for (const { message, extensions, path } of graphQLErrors) {
      console.log(`GraphQL Error [${extensions?.code}]: ${message} at ${path?.join('.')}`);
      
      switch (extensions?.code) {
        case 'UNAUTHENTICATED':
          // Redirect to login
          window.location.href = '/login';
          break;
        
        case 'FORBIDDEN':
          // Show permission error toast
          showToast({ type: 'error', message: 'Access denied' });
          break;
        
        case 'RATE_LIMITED':
          showToast({
            type: 'warning',
            message: `Too many requests. Try again in ${extensions.resetIn}s`,
          });
          break;
        
        default:
          if (process.env.NODE_ENV !== 'production') {
            console.error('GraphQL error:', message);
          }
      }
    }
  }
  
  if (networkError) {
    console.error('Network error:', networkError);
    showToast({ type: 'error', message: 'Network error. Please check your connection.' });
  }
});

const client = new ApolloClient({
  link: ApolloLink.from([errorLink, new HttpLink({ uri: '/graphql' })]),
  cache: new InMemoryCache(),
});
```

### React Component Error Handling

```javascript
function CreateUserForm() {
  const [createUser, { loading, error }] = useMutation(CREATE_USER_MUTATION);
  const [fieldErrors, setFieldErrors] = useState({});
  
  async function handleSubmit(formData) {
    setFieldErrors({});
    
    try {
      await createUser({ variables: { input: formData } });
      // Success
    } catch (err) {
      const gqlError = err.graphQLErrors?.[0];
      
      if (gqlError?.extensions?.code === 'VALIDATION_ERROR') {
        // แสดง field errors
        setFieldErrors(gqlError.extensions.errors || {});
        return;
      }
      
      if (gqlError?.extensions?.code === 'CONFLICT') {
        setFieldErrors({ email: gqlError.message });
        return;
      }
      
      // Generic error
      console.error('Create user failed:', err);
    }
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="email" />
      {fieldErrors.email && <span className="error">{fieldErrors.email}</span>}
      
      <input name="password" type="password" />
      {fieldErrors.password && <span className="error">{fieldErrors.password}</span>}
      
      <button type="submit" disabled={loading}>
        {loading ? 'Creating...' : 'Create Account'}
      </button>
    </form>
  );
}
```

---

## 📌 Step 338: สรุป Part 012

### เนื้อหาที่เรียนรู้

✅ GraphQL error model (partial success)  
✅ GraphQLError class และ extensions  
✅ Custom error classes  
✅ formatError สำหรับ production  
✅ Error masking  
✅ Union-based error handling pattern  
✅ Validation errors  
✅ Client-side error handling  

### Homework

1. implement custom error classes ทั้งหมด
2. ทำ formatError ที่ log ไปยัง Sentry
3. แปลง mutations เป็น Union result pattern

### ในส่วนถัดไป

➡️ **[Part 013](./part-013.md)** — Authentication & JWT

---

*Part 012 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
