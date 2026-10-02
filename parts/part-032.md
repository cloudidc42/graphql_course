# Part 032 — Input Validation & Sanitization 🔒

> **ระดับ:** Intermediate | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 341–380

---

## 🎯 สิ่งที่จะได้เรียนรู้

- GraphQL built-in type validation
- Custom scalar validation
- Input type validation patterns
- Schema-level validation ด้วย directives
- Runtime validation ด้วย zod/joi
- String sanitization
- File upload validation
- ป้องกัน injection ผ่าน validation

---

## 📌 Step 341: ทำไม Input Validation จำเป็น

```
GraphQL มีการ validate types อัตโนมัติ แต่ยังไม่พอ:

✅ GraphQL validates:
- Type correctness (String, Int, etc.)
- Required fields (!!)
- Enum values

❌ GraphQL ไม่ validates:
- String length
- Email format
- URL format
- Number range
- Business rules
- Content safety (XSS)
```

---

## 📌 Step 342: Custom Scalar สำหรับ Validation

```javascript
// src/scalars/email.scalar.js
import { GraphQLScalarType, GraphQLError } from 'graphql';

const EMAIL_REGEX = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;

export const EmailScalar = new GraphQLScalarType({
  name: 'Email',
  description: 'Valid email address',
  
  serialize: (value) => {
    if (typeof value !== 'string') throw new GraphQLError('Email must be a string');
    return value.toLowerCase().trim();
  },
  
  parseValue: (value) => {
    if (typeof value !== 'string') throw new GraphQLError('Email must be a string');
    const email = value.toLowerCase().trim();
    if (!EMAIL_REGEX.test(email)) throw new GraphQLError(`Invalid email: ${value}`);
    return email;
  },
  
  parseLiteral: (ast) => {
    if (ast.kind !== 'StringValue') throw new GraphQLError('Email must be a string');
    const email = ast.value.toLowerCase().trim();
    if (!EMAIL_REGEX.test(email)) throw new GraphQLError(`Invalid email: ${ast.value}`);
    return email;
  },
});

// src/scalars/url.scalar.js
export const URLScalar = new GraphQLScalarType({
  name: 'URL',
  
  parseValue: (value) => {
    try {
      const url = new URL(value);
      if (!['http:', 'https:'].includes(url.protocol)) {
        throw new GraphQLError('Only HTTP/HTTPS URLs allowed');
      }
      return url.toString();
    } catch {
      throw new GraphQLError(`Invalid URL: ${value}`);
    }
  },
  
  serialize: (value) => value,
  parseLiteral: (ast) => URLScalar.parseValue(ast.value),
});
```

---

## 📌 Step 343: @constraint Directive

```bash
npm install graphql-constraint-directive
```

```graphql
# schema.graphql
directive @constraint(
  minLength: Int
  maxLength: Int
  startsWith: String
  endsWith: String
  contains: String
  notContains: String
  pattern: String
  format: String

  min: Float
  max: Float
  exclusiveMin: Float
  exclusiveMax: Float
  multipleOf: Float
) on INPUT_FIELD_DEFINITION | ARGUMENT_DEFINITION | FIELD_DEFINITION

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateProfile(input: UpdateProfileInput!): User!
}

input CreateUserInput {
  name: String! @constraint(minLength: 2, maxLength: 100)
  email: Email!
  password: String! @constraint(minLength: 8, maxLength: 128, pattern: "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)")
  age: Int @constraint(min: 13, max: 120)
  website: URL @constraint(startsWith: "https://")
  bio: String @constraint(maxLength: 500)
}

input UpdateProfileInput {
  name: String @constraint(minLength: 2, maxLength: 100)
  phone: String @constraint(pattern: "^\\+?[1-9]\\d{1,14}$")
  birthDate: Date
}
```

```javascript
// ใช้ constraint directive ใน server
import { createServer } from '@graphql-yoga/node';
import { useValidationCache } from '@envelop/validation-cache';

const schema = makeExecutableSchema({ typeDefs, resolvers });
const constrainedSchema = applyConstraintDirective(schema);
```

---

## 📌 Step 344: Validation ด้วย Zod

```bash
npm install zod
```

```javascript
// src/validation/schemas.js
import { z } from 'zod';

export const createUserSchema = z.object({
  name: z.string()
    .min(2, 'Name too short')
    .max(100, 'Name too long')
    .trim(),
  
  email: z.string()
    .email('Invalid email')
    .toLowerCase(),
  
  password: z.string()
    .min(8, 'Password must be at least 8 characters')
    .max(128, 'Password too long')
    .regex(/[A-Z]/, 'Password must contain uppercase')
    .regex(/[a-z]/, 'Password must contain lowercase')
    .regex(/\d/, 'Password must contain number')
    .regex(/[!@#$%^&*]/, 'Password must contain special character'),
  
  age: z.number().int().min(13).max(120).optional(),
  
  bio: z.string().max(500).optional().nullable(),
});

export const createPostSchema = z.object({
  title: z.string().min(5).max(200).trim(),
  content: z.string().min(10).max(50000),
  tags: z.array(z.string().min(1).max(30)).max(10).optional(),
  status: z.enum(['DRAFT', 'PUBLISHED']).default('DRAFT'),
});

// Validation helper
export function validateInput(schema, input) {
  const result = schema.safeParse(input);
  
  if (!result.success) {
    const errors = result.error.issues.map(issue => ({
      field: issue.path.join('.'),
      message: issue.message,
    }));
    
    throw new GraphQLError('Validation failed', {
      extensions: {
        code: 'BAD_USER_INPUT',
        errors,
      },
    });
  }
  
  return result.data;
}
```

```javascript
// ใน resolver
export const userResolvers = {
  Mutation: {
    createUser: async (_, { input }, { prisma }) => {
      // Validate input
      const validatedInput = validateInput(createUserSchema, input);
      
      // Check email uniqueness
      const existing = await prisma.user.findUnique({
        where: { email: validatedInput.email }
      });
      
      if (existing) {
        throw new GraphQLError('Email already registered', {
          extensions: { code: 'CONFLICT', field: 'email' }
        });
      }
      
      // Hash password
      const hashedPassword = await bcrypt.hash(validatedInput.password, 12);
      
      return prisma.user.create({
        data: { ...validatedInput, password: hashedPassword }
      });
    },
  },
};
```

---

## 📌 Step 345: String Sanitization

```bash
npm install dompurify jsdom validator
```

```javascript
// src/utils/sanitize.js
import DOMPurify from 'dompurify';
import { JSDOM } from 'jsdom';
import validator from 'validator';

const { window } = new JSDOM('');
const purify = DOMPurify(window);

// Sanitize HTML content (สำหรับ rich text fields)
export function sanitizeHTML(dirty) {
  return purify.sanitize(dirty, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'ul', 'li', 'ol', 'br'],
    ALLOWED_ATTR: ['href', 'title', 'target'],
    ALLOW_DATA_ATTR: false,
  });
}

// Strip all HTML (สำหรับ plain text fields)
export function stripHTML(input) {
  return validator.stripLow(DOMPurify.sanitize(input, { ALLOWED_TAGS: [] })).trim();
}

// Sanitize SQL fragments (defense in depth)
export function sanitizeForSQL(input) {
  return validator.escape(input);
}

// Normalize email
export function normalizeEmail(email) {
  return validator.normalizeEmail(email, {
    gmail_lowercase: true,
    gmail_remove_dots: false,
    outlookdotcom_lowercase: true,
    yahoo_lowercase: true,
  }) || email.toLowerCase();
}
```

---

## 📌 Step 346: File Upload Validation

```javascript
// src/validation/fileValidation.js
import { createReadStream } from 'fs';
import fileType from 'file-type';

const ALLOWED_MIME_TYPES = {
  image: ['image/jpeg', 'image/png', 'image/gif', 'image/webp'],
  document: ['application/pdf', 'application/msword'],
  video: ['video/mp4', 'video/webm'],
};

const MAX_FILE_SIZES = {
  image: 5 * 1024 * 1024,    // 5MB
  document: 20 * 1024 * 1024, // 20MB
  video: 100 * 1024 * 1024,   // 100MB
};

export async function validateUpload(file, type = 'image') {
  const { filename, mimetype, createReadStream: stream } = await file;
  
  // 1. Check filename
  if (!filename || filename.length > 255) {
    throw new GraphQLError('Invalid filename');
  }
  
  // 2. Check MIME type from header (ไม่น่าเชื่อถือ 100%)
  if (!ALLOWED_MIME_TYPES[type].includes(mimetype)) {
    throw new GraphQLError(`File type not allowed. Allowed: ${ALLOWED_MIME_TYPES[type].join(', ')}`);
  }
  
  // 3. Read file content to verify actual type
  const readStream = stream();
  const chunks = [];
  let totalSize = 0;
  
  for await (const chunk of readStream) {
    chunks.push(chunk);
    totalSize += chunk.length;
    
    // Check size while reading
    if (totalSize > MAX_FILE_SIZES[type]) {
      throw new GraphQLError(`File too large. Max: ${MAX_FILE_SIZES[type] / 1024 / 1024}MB`);
    }
    
    // Only need first chunk for type detection
    if (chunks.length === 1) {
      const detectedType = await fileType.fromBuffer(Buffer.concat(chunks));
      if (!detectedType || !ALLOWED_MIME_TYPES[type].includes(detectedType.mime)) {
        throw new GraphQLError('File content does not match declared type');
      }
    }
  }
  
  return {
    buffer: Buffer.concat(chunks),
    mimetype,
    filename: sanitizeFilename(filename),
    size: totalSize,
  };
}

function sanitizeFilename(filename) {
  // Remove path traversal, special chars
  return filename
    .replace(/[^a-zA-Z0-9.-]/g, '_')
    .replace(/\.{2,}/g, '_')
    .slice(0, 100);
}
```

---

## 📌 Step 347: Validation Middleware Pattern

```javascript
// src/middleware/validateGraphQL.js
// Validate all mutations automatically

export function withValidation(validationMap) {
  return function validatedResolvers(resolvers) {
    const result = {};
    
    for (const [typeName, typeResolvers] of Object.entries(resolvers)) {
      result[typeName] = {};
      
      for (const [fieldName, resolver] of Object.entries(typeResolvers)) {
        const schema = validationMap[fieldName];
        
        if (schema) {
          result[typeName][fieldName] = async (parent, args, context, info) => {
            // Validate args.input if it exists
            if (args.input) {
              const validated = validateInput(schema, args.input);
              args = { ...args, input: validated };
            }
            
            return resolver(parent, args, context, info);
          };
        } else {
          result[typeName][fieldName] = resolver;
        }
      }
    }
    
    return result;
  };
}

// ใช้งาน
const validatedResolvers = withValidation({
  createUser: createUserSchema,
  updateProfile: updateProfileSchema,
  createPost: createPostSchema,
})(resolvers);
```

---

## 📌 Step 348: สรุป Part 032

### เนื้อหาที่เรียนรู้

✅ Custom scalar validation  
✅ @constraint directive  
✅ Zod schema validation  
✅ String sanitization  
✅ File upload validation  
✅ Validation middleware pattern  

### Security Rules

```
1. Never trust client input
2. Validate at schema level (custom scalars, directives)
3. Validate at resolver level (zod/joi)
4. Sanitize all string inputs
5. Validate file content (not just MIME type header)
6. Return clear validation error messages
7. Never expose internal error details
```

### ในส่วนถัดไป

➡️ **[Part 033](./part-033.md)** — CORS & Security Headers

---

*Part 032 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
