# Part 048 — Custom Scalars Deep Dive 🔢

> **ระดับ:** Intermediate-Advanced | **เวลาเรียน:** 75 นาที | **ขั้นตอนที่:** 841–870

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Custom scalar implementation
- DateTime/Date scalars
- JSON scalar
- Money/Currency scalar
- PhoneNumber scalar
- Latitude/Longitude scalar
- UUID scalar
- Positive/NonNegative Integer scalars
- graphql-scalars library

---

## 📌 Step 841: Custom Scalar Basics

```typescript
// Custom Scalar ต้องกำหนด 3 methods:
// 1. serialize   — JavaScript value → JSON (response)
// 2. parseValue  — JSON input → JavaScript value (variables)
// 3. parseLiteral — AST literal → JavaScript value (inline args)

import { GraphQLScalarType, Kind } from 'graphql';

const EmailScalar = new GraphQLScalarType({
  name: 'Email',
  description: 'A valid email address',
  
  // JavaScript → JSON (เมื่อส่ง response)
  serialize(value: unknown): string {
    if (typeof value !== 'string') throw new Error('Email must be a string');
    const email = value.toLowerCase().trim();
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      throw new Error(`Invalid email: ${value}`);
    }
    return email;
  },
  
  // Variables → JavaScript (parseValue รับ JSON จาก variables)
  parseValue(value: unknown): string {
    if (typeof value !== 'string') throw new Error('Email must be a string');
    const email = value.toLowerCase().trim();
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      throw new Error(`Invalid email: ${value}`);
    }
    return email;
  },
  
  // Literal → JavaScript (parseLiteral รับ AST node จาก inline values)
  parseLiteral(ast): string {
    if (ast.kind !== Kind.STRING) throw new Error('Email must be a string');
    const email = ast.value.toLowerCase().trim();
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      throw new Error(`Invalid email: ${ast.value}`);
    }
    return email;
  },
});
```

---

## 📌 Step 842: DateTime Scalar

```typescript
// src/scalars/DateTime.ts
import { GraphQLScalarType, Kind } from 'graphql';

export const DateTimeScalar = new GraphQLScalarType({
  name: 'DateTime',
  description: 'ISO 8601 DateTime string',
  
  serialize(value: unknown): string {
    if (value instanceof Date) {
      if (isNaN(value.getTime())) throw new Error('Invalid Date');
      return value.toISOString();
    }
    if (typeof value === 'string' || typeof value === 'number') {
      const date = new Date(value);
      if (isNaN(date.getTime())) throw new Error(`Cannot serialize ${value} as DateTime`);
      return date.toISOString();
    }
    throw new Error(`Cannot serialize ${typeof value} as DateTime`);
  },
  
  parseValue(value: unknown): Date {
    if (value instanceof Date) return value;
    if (typeof value === 'string') {
      const date = new Date(value);
      if (isNaN(date.getTime())) throw new Error(`Invalid DateTime: ${value}`);
      return date;
    }
    if (typeof value === 'number') {
      return new Date(value);
    }
    throw new Error(`Cannot parse ${typeof value} as DateTime`);
  },
  
  parseLiteral(ast): Date {
    if (ast.kind === Kind.STRING) {
      const date = new Date(ast.value);
      if (isNaN(date.getTime())) throw new Error(`Invalid DateTime: ${ast.value}`);
      return date;
    }
    if (ast.kind === Kind.INT) {
      return new Date(parseInt(ast.value, 10));
    }
    throw new Error(`Cannot parse DateTime literal of kind ${ast.kind}`);
  },
});

// Date only (no time)
export const DateScalar = new GraphQLScalarType({
  name: 'Date',
  description: 'ISO 8601 Date string (YYYY-MM-DD)',
  
  serialize(value: unknown): string {
    if (value instanceof Date) return value.toISOString().split('T')[0]!;
    if (typeof value === 'string' && /^\d{4}-\d{2}-\d{2}$/.test(value)) return value;
    throw new Error(`Cannot serialize ${value} as Date`);
  },
  
  parseValue(value: unknown): Date {
    if (typeof value !== 'string') throw new Error('Date must be a string');
    if (!/^\d{4}-\d{2}-\d{2}$/.test(value)) throw new Error(`Invalid date format: ${value}`);
    const date = new Date(`${value}T00:00:00.000Z`);
    if (isNaN(date.getTime())) throw new Error(`Invalid date: ${value}`);
    return date;
  },
  
  parseLiteral(ast): Date {
    if (ast.kind !== Kind.STRING) throw new Error('Date must be a string');
    return DateScalar.parseValue!(ast.value) as Date;
  },
});
```

---

## 📌 Step 843: JSON Scalar

```typescript
// src/scalars/JSON.ts
import { GraphQLScalarType, Kind, ValueNode } from 'graphql';

function parseAstLiteral(ast: ValueNode): unknown {
  switch (ast.kind) {
    case Kind.STRING:
    case Kind.BOOLEAN:
      return ast.value;
    case Kind.INT:
    case Kind.FLOAT:
      return parseFloat(ast.value);
    case Kind.OBJECT:
      return ast.fields.reduce((acc, field) => {
        acc[field.name.value] = parseAstLiteral(field.value);
        return acc;
      }, {} as Record<string, unknown>);
    case Kind.LIST:
      return ast.values.map(parseAstLiteral);
    case Kind.NULL:
      return null;
    default:
      throw new Error(`Unsupported AST kind: ${ast.kind}`);
  }
}

export const JSONScalar = new GraphQLScalarType({
  name: 'JSON',
  description: 'Arbitrary JSON value',
  
  serialize(value: unknown): unknown {
    return value; // JSON.stringify ทำให้ระบบ
  },
  
  parseValue(value: unknown): unknown {
    return value; // รับ JSON object โดยตรง
  },
  
  parseLiteral(ast: ValueNode): unknown {
    return parseAstLiteral(ast);
  },
});

// Usage in schema
// type Product {
//   id: ID!
//   name: String!
//   metadata: JSON  # flexible attributes
// }
//
// type Mutation {
//   updateMetadata(id: ID!, data: JSON!): Product
// }
```

---

## 📌 Step 844: Money/Currency Scalar

```typescript
// src/scalars/Money.ts
// เก็บเงินเป็น integer (cents/satang) เพื่อความแม่นยำ

import { GraphQLScalarType, Kind } from 'graphql';

// Money object
export interface Money {
  amount: number;  // integer: 1000 = 10.00 บาท
  currency: string; // ISO 4217: "THB", "USD"
}

export const MoneyScalar = new GraphQLScalarType({
  name: 'Money',
  description: 'Money value as { amount: integer (cents), currency: string }',
  
  serialize(value: unknown): { amount: number; currency: string } {
    if (typeof value !== 'object' || value === null) {
      throw new Error('Money must be an object');
    }
    const { amount, currency } = value as Record<string, unknown>;
    if (typeof amount !== 'number' || !Number.isInteger(amount)) {
      throw new Error('Money.amount must be an integer');
    }
    if (typeof currency !== 'string' || !/^[A-Z]{3}$/.test(currency)) {
      throw new Error('Money.currency must be a 3-letter ISO 4217 code');
    }
    return { amount, currency };
  },
  
  parseValue(value: unknown): Money {
    if (typeof value !== 'object' || value === null) {
      throw new Error('Money input must be an object');
    }
    const { amount, currency } = value as Record<string, unknown>;
    if (typeof amount !== 'number' || !Number.isInteger(amount)) {
      throw new Error('Money.amount must be an integer (cents)');
    }
    if (typeof currency !== 'string' || !/^[A-Z]{3}$/.test(currency)) {
      throw new Error('Money.currency must be ISO 4217');
    }
    return { amount, currency };
  },
  
  parseLiteral(ast) {
    if (ast.kind !== Kind.OBJECT) throw new Error('Money must be an object literal');
    const obj: Record<string, unknown> = {};
    for (const field of ast.fields) {
      if (field.value.kind === Kind.INT || field.value.kind === Kind.FLOAT) {
        obj[field.name.value] = parseFloat(field.value.value);
      } else if (field.value.kind === Kind.STRING) {
        obj[field.name.value] = field.value.value;
      }
    }
    return MoneyScalar.parseValue!(obj) as Money;
  },
});

// Helper: format สำหรับ display
export function formatMoney(money: Money): string {
  return new Intl.NumberFormat('th-TH', {
    style: 'currency',
    currency: money.currency,
    minimumFractionDigits: 2,
  }).format(money.amount / 100);
}
// formatMoney({ amount: 9900, currency: 'THB' }) → "฿99.00"
```

---

## 📌 Step 845: UUID & PositiveInt Scalars

```typescript
// src/scalars/UUID.ts
import { GraphQLScalarType, Kind } from 'graphql';

const UUID_REGEX = /^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;

export const UUIDScalar = new GraphQLScalarType({
  name: 'UUID',
  description: 'UUID v1-v5 string',
  
  serialize(value: unknown): string {
    if (typeof value !== 'string' || !UUID_REGEX.test(value)) {
      throw new Error(`Invalid UUID: ${value}`);
    }
    return value.toLowerCase();
  },
  
  parseValue(value: unknown): string {
    if (typeof value !== 'string' || !UUID_REGEX.test(value)) {
      throw new Error(`Invalid UUID: ${value}`);
    }
    return value.toLowerCase();
  },
  
  parseLiteral(ast): string {
    if (ast.kind !== Kind.STRING) throw new Error('UUID must be a string');
    if (!UUID_REGEX.test(ast.value)) throw new Error(`Invalid UUID: ${ast.value}`);
    return ast.value.toLowerCase();
  },
});

// PositiveInt scalar
export const PositiveIntScalar = new GraphQLScalarType({
  name: 'PositiveInt',
  description: 'Integer > 0',
  
  serialize: validate,
  parseValue: validate,
  parseLiteral(ast) {
    if (ast.kind !== Kind.INT) throw new Error('PositiveInt must be an integer');
    return validate(parseInt(ast.value, 10));
  },
});

function validate(value: unknown): number {
  if (typeof value !== 'number' || !Number.isInteger(value) || value <= 0) {
    throw new Error(`Invalid PositiveInt: ${value}`);
  }
  return value;
}

// NonNegativeInt scalar (>= 0)
export const NonNegativeIntScalar = new GraphQLScalarType({
  name: 'NonNegativeInt',
  description: 'Integer >= 0',
  
  serialize: validateNonNeg,
  parseValue: validateNonNeg,
  parseLiteral(ast) {
    if (ast.kind !== Kind.INT) throw new Error('NonNegativeInt must be an integer');
    return validateNonNeg(parseInt(ast.value, 10));
  },
});

function validateNonNeg(value: unknown): number {
  if (typeof value !== 'number' || !Number.isInteger(value) || value < 0) {
    throw new Error(`Invalid NonNegativeInt: ${value}`);
  }
  return value;
}
```

---

## 📌 Step 846: graphql-scalars Library

```bash
npm install graphql-scalars
```

```typescript
// src/scalars/index.ts
// ใช้ graphql-scalars แทนเขียนเอง
import {
  DateTimeResolver,
  DateResolver,
  EmailAddressResolver,
  URLResolver,
  UUIDResolver,
  PositiveIntResolver,
  NonNegativeIntResolver,
  NonNegativeFloatResolver,
  PhoneNumberResolver,
  PostalCodeResolver,
  JSONResolver,
  BigIntResolver,
  ByteResolver,
  HexColorCodeResolver,
  CountryCodeResolver,
  LocaleResolver,
  TimeZoneResolver,
  IPv4Resolver,
  IPv6Resolver,
  MACAddressResolver,
  PortResolver,
} from 'graphql-scalars';

// Register all in resolver map
export const scalarResolvers = {
  DateTime: DateTimeResolver,
  Date: DateResolver,
  EmailAddress: EmailAddressResolver,
  URL: URLResolver,
  UUID: UUIDResolver,
  PositiveInt: PositiveIntResolver,
  NonNegativeInt: NonNegativeIntResolver,
  NonNegativeFloat: NonNegativeFloatResolver,
  PhoneNumber: PhoneNumberResolver,
  PostalCode: PostalCodeResolver,
  JSON: JSONResolver,
  BigInt: BigIntResolver,
  HexColorCode: HexColorCodeResolver,
  CountryCode: CountryCodeResolver,
  Locale: LocaleResolver,
  TimeZone: TimeZoneResolver,
  IPv4: IPv4Resolver,
  IPv6: IPv6Resolver,
  Port: PortResolver,
};

// SDL type definitions
export const scalarTypeDefs = `
  scalar DateTime
  scalar Date
  scalar EmailAddress
  scalar URL
  scalar UUID
  scalar PositiveInt
  scalar NonNegativeInt
  scalar NonNegativeFloat
  scalar PhoneNumber
  scalar PostalCode
  scalar JSON
  scalar BigInt
  scalar HexColorCode
  scalar CountryCode
  scalar Locale
  scalar TimeZone
  scalar IPv4
  scalar IPv6
  scalar Port
`;
```

---

## 📌 Step 847: สรุป Part 048

### เนื้อหาที่เรียนรู้

✅ Custom scalar implementation (serialize/parseValue/parseLiteral)  
✅ DateTime/Date scalars  
✅ JSON scalar  
✅ Money/Currency scalar  
✅ UUID/PositiveInt scalars  
✅ graphql-scalars library  

### เมื่อไรควรใช้ Custom Scalar

```
ใช้ Custom Scalar เมื่อ:
1. ต้องการ validation ที่ String/Int ธรรมดาทำไม่ได้
2. ต้องการ serialization/deserialization พิเศษ
3. ต้องการ type safety สูง (UUID, Email, Money)
4. ต้องการ self-documenting schema

ไม่ต้องใช้ Custom Scalar เมื่อ:
1. String ธรรมดาพอ (ไม่ต้องการ validation)
2. Validation ทำใน resolver/service layer แล้ว
3. Performance critical paths (scalar parsing ใช้เวลา)
```

### ในส่วนถัดไป

➡️ **[Part 049](./part-049.md)** — Union Types & Interface Patterns

---

*Part 048 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
