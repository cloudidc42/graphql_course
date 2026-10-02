# Part 078 — GraphQL Schema Migration & Breaking Changes 🔄

> **ระดับ:** World-Class | **เวลาเรียน:** 80 นาที | **ขั้นตอนที่:** 2011–2050

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Breaking change classification
- Safe migration patterns
- Schema versioning strategies
- Deprecation workflows
- Client migration tools
- Operation tracking
- Automated breaking change detection
- Schema changelog

---

## 📌 Step 2011: Breaking Change Classification

```
Breaking Changes (ทำลาย clients):

Field Removal:
- ลบ field → clients ที่ query field นี้จะ error
- ลบ argument → clients ที่ส่ง argument นี้จะ error

Type Changes:
- String! → Int! → type mismatch
- [Product!]! → Product → แตกต่าง shape

Enum Changes:
- ลบ enum value → clients ที่ match on it จะ miss
- เปลี่ยนชื่อ enum value

Input Changes:
- เพิ่ม required input field → clients ไม่ส่งจะ error
- เปลี่ยน type ของ input field

Non-Breaking Changes (ปลอดภัย):
- เพิ่ม optional field
- เพิ่ม enum value (ระวัง client exhaustive switch)
- เพิ่ม optional input argument
- เพิ่ม new type
- Nullable → Non-null (แค่ตาม spec, ระวัง resolvers)
- เพิ่ม interface implementation
```

---

## 📌 Step 2012: Deprecation Workflow

```typescript
// src/schema/product.graphql
// Week 1: Add new field + deprecate old
type Product {
  # เก่า
  price: Float @deprecated(reason: "Use `pricing.basePrice`. Will be removed 2025-06-01")
  
  # ใหม่ (เพิ่มก่อน, ค่อย deprecate เก่า)
  pricing: ProductPricing!
}

type ProductPricing {
  basePrice: Float!
  salePrice: Float
  currency: Currency!
}

# Week 2-8: Monitor usage ใน schema analytics
# ดูว่า clients ยังใช้ price field อยู่ไหม

# Week 8+: ถ้า usage = 0 → remove field
type Product {
  # price: Float ← ลบแล้ว
  pricing: ProductPricing!
}
```

```typescript
// Track deprecated field usage
const deprecationTrackingPlugin = (): ApolloServerPlugin => ({
  async requestDidStart({ request }) {
    return {
      async executionDidStart({ document }) {
        return {
          willResolveField({ info }) {
            const field = info.parentType.getFields()[info.fieldName];
            
            if (field?.deprecationReason) {
              // Track in analytics
              metrics.deprecatedFieldUsage.inc({
                type: info.parentType.name,
                field: info.fieldName,
                operation: request.operationName ?? 'anonymous',
                client: request.http?.headers.get('apollographql-client-name') ?? 'unknown',
              });
            }
          },
        };
      },
    };
  },
});
```

---

## 📌 Step 2013: Automated Breaking Change Detection

```typescript
// scripts/check-breaking-changes.ts
import { buildSchema, findBreakingChanges, findDangerousChanges } from 'graphql';
import { readFileSync } from 'fs';

async function checkBreakingChanges() {
  // ดึง schema ที่ deploy แล้ว
  const currentSchemaSDL = await fetchCurrentSchema();
  const currentSchema = buildSchema(currentSchemaSDL);
  
  // โหลด schema ใหม่
  const newSchemaSDL = readFileSync('./schema.graphql', 'utf-8');
  const newSchema = buildSchema(newSchemaSDL);
  
  // ตรวจสอบ
  const breaking = findBreakingChanges(currentSchema, newSchema);
  const dangerous = findDangerousChanges(currentSchema, newSchema);
  
  if (breaking.length > 0) {
    console.error('BREAKING CHANGES DETECTED:');
    for (const change of breaking) {
      console.error(`  ❌ [${change.type}] ${change.description}`);
    }
    
    // ถ้าไม่มี flag override → fail CI
    if (!process.env.ALLOW_BREAKING_CHANGES) {
      process.exit(1);
    }
  }
  
  if (dangerous.length > 0) {
    console.warn('DANGEROUS CHANGES (may break some clients):');
    for (const change of dangerous) {
      console.warn(`  ⚠️ [${change.type}] ${change.description}`);
    }
  }
  
  if (breaking.length === 0 && dangerous.length === 0) {
    console.log('✅ No breaking changes detected');
  }
}

checkBreakingChanges();
```

```yaml
# .github/workflows/schema-check.yml
- name: Check Breaking Changes
  run: |
    npx ts-node scripts/check-breaking-changes.ts
  env:
    GRAPHQL_URL: ${{ vars.PROD_GRAPHQL_URL }}
    # ALLOW_BREAKING_CHANGES: "true"  ← ต้อง explicit approve
```

---

## 📌 Step 2014: Schema Versioning ด้วย @deprecated + Sunset Header

```typescript
// ส่ง Sunset header เพื่อ notify clients
app.use('/graphql', (req, res, next) => {
  // Deprecated fields ที่จะลบ
  const sunsetDates: Record<string, string> = {
    'Product.price': '2025-06-01T00:00:00Z',
    'Query.oldUserSearch': '2025-03-01T00:00:00Z',
  };
  
  // ตรวจสอบ query ว่าใช้ deprecated fields ไหม
  // (simplified: check ใน plugin แทน)
  
  next();
});

// Sunset header plugin
const sunsetPlugin = (): ApolloServerPlugin => ({
  async requestDidStart({ request }) {
    const deprecatedFieldsUsed: string[] = [];
    
    return {
      async executionDidStart() {
        return {
          willResolveField({ info }) {
            const field = info.parentType.getFields()[info.fieldName];
            
            if (field?.deprecationReason) {
              const key = `${info.parentType.name}.${info.fieldName}`;
              deprecatedFieldsUsed.push(key);
            }
          },
        };
      },
      
      async willSendResponse({ response }) {
        if (deprecatedFieldsUsed.length === 0) return;
        
        // ส่ง Sunset header สำหรับ deprecated fields
        const soonestSunset = deprecatedFieldsUsed
          .map(key => sunsetDates[key])
          .filter(Boolean)
          .sort()[0];
        
        if (soonestSunset) {
          response.http.headers.set('Sunset', soonestSunset);
          response.http.headers.set('Deprecation', 'true');
          response.http.headers.set(
            'Link',
            '<https://docs.myapi.com/migration>; rel="deprecation"'
          );
        }
      },
    };
  },
});
```

---

## 📌 Step 2015: Schema Changelog

```markdown
# Schema Changelog

## 2025-03-01 (v2.4.0)

### Added
- `Product.pricing: ProductPricing!` — Structured pricing object
- `ProductPricing.basePrice: Float!`
- `ProductPricing.salePrice: Float`
- `User.profile: UserProfile!` — Separate profile type

### Deprecated (removal: 2025-09-01)
- `Product.price: Float` → Use `Product.pricing.basePrice`
- `User.bio: String` → Use `User.profile.bio`
- `User.avatar: String` → Use `User.profile.avatarUrl`

### Migration Guide
```graphql
# Before
query { product(id: "1") { price } }

# After  
query { product(id: "1") { pricing { basePrice salePrice } } }
```

## 2025-01-15 (v2.3.0)

### Breaking Changes (approved via RFC-047)
- `Mutation.updateUser` input: `name` → `displayName`
  - Affected: 2 known clients (iOS app v1.x, Partner API v1)
  - Migration: send both fields during transition

### Added
- `Query.userByEmail(email: String!): User`
```

---

## 📌 Step 2016: สรุป Part 078

### เนื้อหาที่เรียนรู้

✅ Breaking change classification  
✅ Deprecation workflow  
✅ Automated breaking change detection  
✅ Sunset headers  
✅ Schema changelog  

### Migration Timeline

```
Week 1:  Add new field/type (non-breaking)
Week 2:  Mark old field @deprecated
Week 3:  Notify clients via changelog + email
Week 4+: Monitor deprecated field usage
Week 8:  Check if usage = 0
         Yes → Schedule removal
         No  → Contact remaining clients
Week 12: Remove deprecated field
```

### ในส่วนถัดไป

➡️ **[Part 079](./part-079.md)** — Enterprise GraphQL: Multi-Team Governance

---

*Part 078 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
