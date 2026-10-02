# Part 033 — CORS & Security Headers 🔐

> **ระดับ:** Intermediate | **เวลาเรียน:** 60 นาที | **ขั้นตอนที่:** 381–410

---

## 🎯 สิ่งที่จะได้เรียนรู้

- CORS คืออะไร และทำงานยังไง
- CORS configuration สำหรับ GraphQL
- Preflight requests
- Security headers ด้วย helmet
- Content Security Policy (CSP)
- CSRF protection
- Cookie security
- HTTPS enforcement

---

## 📌 Step 381: CORS คืออะไร

```
Cross-Origin Resource Sharing (CORS)

Browser security policy ป้องกัน:
- javascript ที่ origin A อ่าน response จาก origin B

ตัวอย่าง:
- Frontend: https://myapp.com
- GraphQL API: https://api.myapp.com

→ ต้อง configure CORS ให้ API รู้จัก frontend

⚠️ GraphQL endpoint เดียว ใช้ POST → ต้องการ CORS config ที่ถูกต้อง
```

---

## 📌 Step 382: CORS Configuration

```bash
npm install cors
```

```javascript
// src/middleware/cors.js
import cors from 'cors';

const ALLOWED_ORIGINS = [
  'https://myapp.com',
  'https://www.myapp.com',
  'https://admin.myapp.com',
  
  // Development
  ...(process.env.NODE_ENV !== 'production' ? [
    'http://localhost:3000',
    'http://localhost:3001',
    'http://127.0.0.1:3000',
  ] : []),
];

export const corsMiddleware = cors({
  origin: (origin, callback) => {
    // Allow requests with no origin (e.g., mobile apps, Postman)
    if (!origin) return callback(null, true);
    
    if (ALLOWED_ORIGINS.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error(`Origin ${origin} not allowed by CORS`));
    }
  },
  
  methods: ['GET', 'POST', 'OPTIONS'],
  
  allowedHeaders: [
    'Content-Type',
    'Authorization',
    'Apollo-Require-Preflight',
    'X-Apollo-Operation-Name',
    'X-Request-ID',
  ],
  
  // Allow cookies to be sent
  credentials: true,
  
  // Preflight cache: 24 hours
  maxAge: 86400,
});

// สำหรับ GraphQL WebSocket (subscriptions)
export const wsCorsMiddleware = {
  origin: ALLOWED_ORIGINS,
  credentials: true,
};
```

---

## 📌 Step 383: Preflight Requests

```javascript
// OPTIONS preflight request ต้องตอบก่อน
app.options('/graphql', corsMiddleware); // ← สำคัญ!
app.use('/graphql', corsMiddleware);

// หรือ
app.use(corsMiddleware); // ทั้ง app

// Apollo Server ต้องการ header พิเศษ
// Client ต้องส่ง: Apollo-Require-Preflight: true
// สำหรับ multipart (file upload) ต้องมี preflight เสมอ
```

---

## 📌 Step 384: Security Headers ด้วย Helmet

```bash
npm install helmet
```

```javascript
// src/middleware/security.js
import helmet from 'helmet';

export const securityHeaders = helmet({
  // Content Security Policy
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"],  // ← unsafe-inline สำหรับ GraphQL Playground
      styleSrc: ["'self'", "'unsafe-inline'", 'https://fonts.googleapis.com'],
      fontSrc: ["'self'", 'https://fonts.gstatic.com'],
      imgSrc: ["'self'", 'data:', 'https://cdn.myapp.com'],
      connectSrc: ["'self'", 'wss://api.myapp.com'],  // ← WebSocket
      frameSrc: ["'none'"],
      objectSrc: ["'none'"],
    },
  },
  
  // HSTS: Force HTTPS for 1 year
  hsts: {
    maxAge: 31536000,          // 1 year
    includeSubDomains: true,
    preload: true,
  },
  
  // Prevent clickjacking
  frameguard: { action: 'deny' },
  
  // Prevent MIME sniffing
  noSniff: true,
  
  // XSS protection (legacy browsers)
  xssFilter: true,
  
  // Referrer Policy
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
  
  // Disable powered-by header
  hidePoweredBy: true,
  
  // Disable caching for sensitive responses
  noCache: false, // ← false เพราะ GraphQL responses มักควร cache ได้
});

// สำหรับ production
if (process.env.NODE_ENV === 'production') {
  app.use(securityHeaders);
} else {
  // Development: ผ่อนปรนกว่า (เพื่อ GraphQL Playground)
  app.use(helmet({ contentSecurityPolicy: false }));
}
```

---

## 📌 Step 385: CSRF Protection

```javascript
// GraphQL ปกติใช้ Authorization header → ป้องกัน CSRF ได้ระดับหนึ่ง
// แต่ถ้าใช้ cookies → ต้องป้องกัน CSRF

import { doubleCsrf } from 'csrf-csrf';

const { doubleCsrfProtection, generateToken } = doubleCsrf({
  getSecret: () => process.env.CSRF_SECRET,
  cookieName: '__Host-psifi.x-csrf-token',
  cookieOptions: {
    httpOnly: true,
    sameSite: 'strict',
    secure: process.env.NODE_ENV === 'production',
    path: '/',
  },
  size: 64,
  getTokenFromRequest: (req) => {
    // ดึง token จาก header หรือ body
    return req.headers['x-csrf-token'] || req.body?.csrfToken;
  },
});

// Route สำหรับขอ CSRF token
app.get('/csrf-token', (req, res) => {
  res.json({ csrfToken: generateToken(res, req) });
});

// ป้องกัน GraphQL mutations ด้วย CSRF
app.use('/graphql', (req, res, next) => {
  // อนุญาต GET (introspection, queries via GET)
  if (req.method === 'GET') return next();
  
  // ตรวจสอบ CSRF token สำหรับ POST
  doubleCsrfProtection(req, res, next);
});
```

---

## 📌 Step 386: Secure Cookie Configuration

```javascript
// src/middleware/cookies.js

// ตั้งค่า secure cookies สำหรับ refresh tokens
export function setRefreshTokenCookie(res, token) {
  res.cookie('refresh_token', token, {
    httpOnly: true,         // ← JS ไม่สามารถอ่านได้
    secure: process.env.NODE_ENV === 'production', // HTTPS only
    sameSite: 'strict',     // ป้องกัน CSRF
    maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
    path: '/graphql',       // ← เฉพาะ GraphQL endpoint
    domain: process.env.COOKIE_DOMAIN,
  });
}

export function clearRefreshTokenCookie(res) {
  res.clearCookie('refresh_token', {
    path: '/graphql',
    domain: process.env.COOKIE_DOMAIN,
  });
}

// ใน resolver
const resolvers = {
  Mutation: {
    login: async (_, { email, password }, { res }) => {
      const { accessToken, refreshToken } = await authService.login(email, password);
      
      // Set refresh token as httpOnly cookie
      setRefreshTokenCookie(res, refreshToken);
      
      // Return access token (short-lived, 15 min)
      return { accessToken, expiresIn: 900 };
    },
    
    logout: async (_, __, { res }) => {
      clearRefreshTokenCookie(res);
      return true;
    },
  },
};
```

---

## 📌 Step 387: Testing Security Headers

```javascript
// test/security/headers.test.js
import request from 'supertest';
import app from '../src/app.js';

describe('Security Headers', () => {
  let response;
  
  beforeAll(async () => {
    response = await request(app).post('/graphql').send({ query: '{ health }' });
  });
  
  it('should have HSTS header', () => {
    expect(response.headers['strict-transport-security']).toContain('max-age=');
  });
  
  it('should prevent clickjacking', () => {
    expect(response.headers['x-frame-options']).toBe('DENY');
  });
  
  it('should have Content Security Policy', () => {
    expect(response.headers['content-security-policy']).toBeDefined();
  });
  
  it('should not expose server information', () => {
    expect(response.headers['x-powered-by']).toBeUndefined();
    expect(response.headers['server']).toBeUndefined();
  });
});

describe('CORS', () => {
  it('should reject unknown origins', async () => {
    const res = await request(app)
      .post('/graphql')
      .set('Origin', 'https://evil.com')
      .send({ query: '{ health }' });
    
    expect(res.headers['access-control-allow-origin']).not.toBe('https://evil.com');
  });
  
  it('should allow trusted origins', async () => {
    const res = await request(app)
      .post('/graphql')
      .set('Origin', 'https://myapp.com')
      .send({ query: '{ health }' });
    
    expect(res.headers['access-control-allow-origin']).toBe('https://myapp.com');
  });
});
```

---

## 📌 Step 388: สรุป Part 033

### เนื้อหาที่เรียนรู้

✅ CORS configuration  
✅ Preflight requests  
✅ Security headers ด้วย helmet  
✅ CSRF protection  
✅ Secure cookie configuration  
✅ Security header testing  

### Security Checklist

```
✅ Configure CORS whitelist (ไม่ใช้ * ใน production)
✅ ใส่ credentials: true เฉพาะเมื่อจำเป็น
✅ ใช้ helmet() ครบ directives
✅ HSTS ด้วย preload: true
✅ httpOnly + sameSite=strict สำหรับ auth cookies
✅ CSRF protection ถ้าใช้ cookie-based auth
✅ ทดสอบ security headers ทุก deploy
```

### ในส่วนถัดไป

➡️ **[Part 034](./part-034.md)** — JWT Security Best Practices

---

*Part 033 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
