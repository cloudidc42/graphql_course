# Part 077 — GraphQL Internationalization (i18n) 🌏

> **ระดับ:** Expert | **เวลาเรียน:** 70 นาที | **ขั้นตอนที่:** 1976–2010

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Locale handling ใน GraphQL
- Translatable content pattern
- Error messages localization
- Currency & number formatting
- Date/time localization
- Multi-language schema
- Translation management

---

## 📌 Step 1976: Locale in Context

```typescript
// src/context/locale.ts

type Locale = 'th' | 'en' | 'zh' | 'ja';

// Extract locale from request
export function extractLocale(req: Request): Locale {
  // Priority: query param > Accept-Language header > default
  const queryLocale = req.query.locale as string;
  if (isValidLocale(queryLocale)) return queryLocale as Locale;
  
  const acceptLanguage = req.headers['accept-language'];
  if (acceptLanguage) {
    const primary = acceptLanguage.split(',')[0]?.split('-')[0];
    if (primary && isValidLocale(primary)) return primary as Locale;
  }
  
  return 'th';  // default locale
}

function isValidLocale(locale: string): locale is Locale {
  return ['th', 'en', 'zh', 'ja'].includes(locale);
}

// Context includes locale
interface AppContext {
  user: UserPayload | null;
  locale: Locale;
  currency: Currency;
  prisma: PrismaClient;
}
```

---

## 📌 Step 1977: Translatable Content Schema

```graphql
# Pattern 1: Translation field per locale (simple)
type Product {
  id: ID!
  
  # Default locale (always Thai)
  name: String!
  description: String
  
  # Translations
  translations: [ProductTranslation!]!
  
  # ดึง localized content ตาม context locale
  localizedName: String!
  localizedDescription: String
}

type ProductTranslation {
  locale: Locale!
  name: String!
  description: String
}

enum Locale {
  TH
  EN
  ZH
  JA
}

# Pattern 2: Locale argument per field (flexible)
type Product {
  id: ID!
  name(locale: Locale = TH): String!
  description(locale: Locale = TH): String
}

# Pattern 3: Separate translation query
type Query {
  product(id: ID!): Product
  productTranslation(id: ID!, locale: Locale!): ProductTranslation
}
```

```typescript
// Resolver สำหรับ translatable content
const productResolvers = {
  Product: {
    // ดึง localized name ตาม context locale
    localizedName: async (product, _, ctx) => {
      const { locale } = ctx;
      
      if (locale === 'th') return product.name;  // default
      
      // ดึง translation จาก DB หรือ cache
      const translation = await ctx.loaders.productTranslation.load({
        productId: product.id,
        locale,
      });
      
      return translation?.name ?? product.name;  // fallback to default
    },
    
    // Pattern 2: locale argument
    name: async (product, { locale }, ctx) => {
      const targetLocale = locale ?? ctx.locale;
      
      if (targetLocale === 'th') return product.name;
      
      const translation = await ctx.prisma.productTranslation.findUnique({
        where: { productId_locale: { productId: product.id, locale: targetLocale } },
      });
      
      return translation?.name ?? product.name;
    },
  },
  
  Mutation: {
    updateProductTranslation: async (_, { productId, locale, input }, ctx) => {
      requireRole(ctx, 'ADMIN');
      
      return ctx.prisma.productTranslation.upsert({
        where: { productId_locale: { productId, locale } },
        create: { productId, locale, ...input },
        update: input,
      });
    },
  },
};
```

---

## 📌 Step 1978: Error Message Localization

```typescript
// src/i18n/messages.ts

type MessageKey = 
  | 'NOT_FOUND'
  | 'UNAUTHORIZED'
  | 'FORBIDDEN'
  | 'VALIDATION_REQUIRED'
  | 'VALIDATION_TOO_SHORT'
  | 'VALIDATION_TOO_LONG'
  | 'OUT_OF_STOCK'
  | 'PAYMENT_FAILED';

const messages: Record<Locale, Record<MessageKey, string>> = {
  th: {
    NOT_FOUND: 'ไม่พบข้อมูลที่ต้องการ',
    UNAUTHORIZED: 'กรุณาเข้าสู่ระบบก่อน',
    FORBIDDEN: 'คุณไม่มีสิทธิ์ดำเนินการนี้',
    VALIDATION_REQUIRED: 'ข้อมูลนี้จำเป็นต้องกรอก',
    VALIDATION_TOO_SHORT: 'ข้อมูลสั้นเกินไป (ต้องการอย่างน้อย {min} ตัวอักษร)',
    VALIDATION_TOO_LONG: 'ข้อมูลยาวเกินไป (สูงสุด {max} ตัวอักษร)',
    OUT_OF_STOCK: 'สินค้าหมดสต็อก',
    PAYMENT_FAILED: 'การชำระเงินล้มเหลว กรุณาลองใหม่',
  },
  en: {
    NOT_FOUND: 'The requested resource was not found',
    UNAUTHORIZED: 'Please log in to continue',
    FORBIDDEN: 'You are not authorized to perform this action',
    VALIDATION_REQUIRED: 'This field is required',
    VALIDATION_TOO_SHORT: 'Must be at least {min} characters',
    VALIDATION_TOO_LONG: 'Must be at most {max} characters',
    OUT_OF_STOCK: 'This product is out of stock',
    PAYMENT_FAILED: 'Payment failed. Please try again',
  },
  zh: {
    NOT_FOUND: '未找到请求的资源',
    UNAUTHORIZED: '请先登录',
    FORBIDDEN: '您没有权限执行此操作',
    VALIDATION_REQUIRED: '此字段为必填项',
    VALIDATION_TOO_SHORT: '最少需要 {min} 个字符',
    VALIDATION_TOO_LONG: '最多允许 {max} 个字符',
    OUT_OF_STOCK: '此商品缺货',
    PAYMENT_FAILED: '付款失败，请重试',
  },
  ja: {
    NOT_FOUND: '要求されたリソースが見つかりません',
    UNAUTHORIZED: 'ログインしてください',
    FORBIDDEN: 'この操作を実行する権限がありません',
    VALIDATION_REQUIRED: 'この項目は必須です',
    VALIDATION_TOO_SHORT: '最低{min}文字必要です',
    VALIDATION_TOO_LONG: '最大{max}文字まで入力できます',
    OUT_OF_STOCK: 'この商品は在庫切れです',
    PAYMENT_FAILED: '支払いに失敗しました。もう一度お試しください',
  },
};

export function getMessage(key: MessageKey, locale: Locale, params?: Record<string, string | number>): string {
  const template = messages[locale]?.[key] ?? messages.en[key] ?? key;
  
  if (!params) return template;
  
  return template.replace(/\{(\w+)\}/g, (_, k) => String(params[k] ?? k));
}

// Usage ใน resolver
const getLocale = (ctx: AppContext): Locale => ctx.locale ?? 'th';

throw new AppError(
  getMessage('NOT_FOUND', getLocale(ctx)),
  ErrorCode.NOT_FOUND
);
```

---

## 📌 Step 1979: Currency & Price Formatting

```typescript
// src/utils/currency.ts

type Currency = 'THB' | 'USD' | 'JPY' | 'CNY';

// Price scalar: store ใน smallest unit (satang for THB, cents for USD)
// Display: format ตาม locale

export function formatPrice(
  amountInSmallestUnit: number,
  currency: Currency,
  locale: Locale
): string {
  const amount = amountInSmallestUnit / 100;  // satang → baht
  
  const localeMap: Record<Locale, string> = {
    th: 'th-TH',
    en: 'en-US',
    zh: 'zh-CN',
    ja: 'ja-JP',
  };
  
  return new Intl.NumberFormat(localeMap[locale], {
    style: 'currency',
    currency,
    minimumFractionDigits: currency === 'JPY' ? 0 : 2,
    maximumFractionDigits: currency === 'JPY' ? 0 : 2,
  }).format(amount);
}

// GraphQL resolver: return formatted price
const productResolvers = {
  Product: {
    formattedPrice: (product, _, ctx) => {
      return formatPrice(
        product.priceInSatang,
        ctx.currency ?? 'THB',
        ctx.locale ?? 'th'
      );
      // THB 1,250.00 หรือ ฿1,250.00 หรือ US$37.50
    },
  },
};
```

---

## 📌 Step 1980: สรุป Part 077

### เนื้อหาที่เรียนรู้

✅ Locale extraction จาก HTTP headers  
✅ Translatable content patterns  
✅ Localized error messages  
✅ Currency & price formatting  
✅ Multi-language schema design  

### i18n Best Practices

```
Schema Design:
□ เก็บ default language ใน main type
□ Translation ใน separate table
□ Fallback: translation → default language
□ Context-aware locale (ไม่ใช่ global variable)

Error Messages:
□ Localize ทุก user-facing messages
□ Template literals สำหรับ dynamic values
□ Fallback ไปยัง English ถ้าไม่มี translation

Data Storage:
□ Store money ใน integer (satang/cents)
□ Format เฉพาะตอน display
□ Timezone: เก็บ UTC, display ตาม user timezone
□ Sort/collation per locale
```

### ในส่วนถัดไป

➡️ **[Part 078](./part-078.md)** — GraphQL Schema Migration & Breaking Changes

---

*Part 077 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
