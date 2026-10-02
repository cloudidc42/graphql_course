# Part 040 — Complete Security Hardening Checklist ✅

> **ระดับ:** Advanced | **เวลาเรียน:** 60 นาที | **ขั้นตอนที่:** 651–680

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Complete production security checklist
- Security hardening script
- Automated security testing
- Security monitoring alerts
- Dependency scanning
- Regular security review process

---

## 📌 Step 651: Production Security Checklist

```
📋 GRAPHQL PRODUCTION SECURITY CHECKLIST

AUTHENTICATION & AUTHORIZATION
□ JWT: ใช้ RS256 ไม่ใช่ HS256
□ Access token TTL: 15 นาที หรือน้อยกว่า
□ Refresh token: opaque, เก็บใน DB
□ Refresh token rotation ทุก use
□ Token revocation list (Redis)
□ Password hashing: bcrypt cost factor >= 12
□ MFA available สำหรับ admin accounts
□ OAuth: state parameter validation
□ OAuth: PKCE สำหรับ public clients

GRAPHQL SPECIFIC
□ Introspection: disabled ใน production
□ Query depth limit: max 7
□ Query complexity limit: max 1000
□ Alias limit: max 15
□ Batch request limit: max 5
□ Field suggestions: disabled
□ Persisted queries: enabled ใน production

INPUT VALIDATION
□ Custom scalars สำหรับ Email, URL, Date
□ String length limits ทุก field
□ @constraint directive
□ Runtime validation (zod/joi)
□ File upload validation (type + size + content)
□ SQL/NoSQL injection prevention

RATE LIMITING
□ Global rate limit: 100 req/15 min per IP
□ Auth rate limit: 10 attempts/hour per IP
□ Mutation rate limit ตาม operation
□ WebSocket connection limit

NETWORK & TRANSPORT
□ HTTPS only (HSTS enabled)
□ TLS 1.2+ only
□ CORS: whitelist specific origins
□ Security headers: helmet configured
□ CSRF protection (ถ้าใช้ cookie auth)
□ WebSocket: origin validation

ERROR HANDLING
□ Mask internal errors ใน production
□ No stack traces ใน error responses
□ No sensitive data ใน error messages
□ Consistent error format

MONITORING & LOGGING
□ Structured logging (JSON)
□ Audit log ทุก operation
□ Sensitive field access logging
□ Failed auth attempt alerting
□ Rate limit violation alerting
□ Error rate monitoring
□ Response time monitoring

DEPENDENCIES
□ npm audit: no HIGH/CRITICAL
□ Dependency updates: automated (Renovate/Dependabot)
□ Node.js version: LTS latest
□ Docker base image: updated monthly

DATA PROTECTION
□ Encryption at rest (database)
□ Encryption in transit (TLS)
□ PII fields: identified and protected
□ Log redaction: passwords, tokens
□ Backup encryption

INFRASTRUCTURE
□ Kubernetes: security contexts set
□ Non-root container user
□ Read-only filesystem (where possible)
□ Network policies: principle of least privilege
□ Secrets: Kubernetes Secrets or Vault
□ No hardcoded credentials
```

---

## 📌 Step 652: Security Configuration File

```javascript
// src/config/security.js
export const securityConfig = {
  // JWT
  jwt: {
    algorithm: 'RS256',
    accessTokenTtl: '15m',
    refreshTokenTtl: '7d',
    issuer: process.env.JWT_ISSUER,
    audience: process.env.JWT_AUDIENCE,
  },
  
  // Password
  password: {
    bcryptRounds: 12,
    minLength: 8,
    requireUppercase: true,
    requireNumber: true,
    requireSpecial: true,
  },
  
  // GraphQL
  graphql: {
    introspection: process.env.NODE_ENV !== 'production',
    maxDepth: 7,
    maxComplexity: 1000,
    maxAliases: 15,
    maxBatchSize: 5,
    queryTimeout: 30000, // 30 seconds
    enablePlayground: process.env.NODE_ENV === 'development',
  },
  
  // Rate limiting
  rateLimit: {
    global: { points: 100, duration: 900 },    // 100/15min
    auth: { points: 10, duration: 3600 },       // 10/hour
    mutations: { points: 30, duration: 60 },    // 30/min
    fileUpload: { points: 5, duration: 3600 },  // 5/hour
  },
  
  // CORS
  cors: {
    origins: process.env.ALLOWED_ORIGINS?.split(',') || [],
    credentials: true,
    maxAge: 86400,
  },
  
  // File upload
  upload: {
    maxFileSize: 10 * 1024 * 1024, // 10MB
    allowedTypes: {
      image: ['image/jpeg', 'image/png', 'image/webp', 'image/gif'],
      document: ['application/pdf'],
    },
  },
};
```

---

## 📌 Step 653: Automated Security Tests

```javascript
// test/security/security.test.js
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import app from '../../src/app.js';

const GQL_ENDPOINT = '/graphql';

describe('Security Tests', () => {
  describe('Introspection', () => {
    it('should disable introspection in production', async () => {
      if (process.env.NODE_ENV !== 'production') return;
      
      const res = await request(app)
        .post(GQL_ENDPOINT)
        .send({ query: '{ __schema { types { name } } }' });
      
      expect(res.body.errors).toBeDefined();
    });
  });
  
  describe('Query depth', () => {
    it('should block deeply nested queries', async () => {
      const deepQuery = `{ user(id: "1") { friends { friends { friends { friends { friends { friends { friends { name } } } } } } } } }`;
      
      const res = await request(app)
        .post(GQL_ENDPOINT)
        .send({ query: deepQuery });
      
      expect(res.body.errors?.[0]?.message).toMatch(/depth/i);
    });
  });
  
  describe('Rate limiting', () => {
    it('should rate limit excessive requests', async () => {
      const promises = Array.from({ length: 200 }, () =>
        request(app)
          .post(GQL_ENDPOINT)
          .set('X-Forwarded-For', '192.168.1.100')
          .send({ query: '{ health }' })
      );
      
      const results = await Promise.all(promises);
      const rateLimited = results.filter(r => r.status === 429);
      expect(rateLimited.length).toBeGreaterThan(0);
    });
  });
  
  describe('Security headers', () => {
    it('should include required security headers', async () => {
      const res = await request(app)
        .get('/health');
      
      expect(res.headers['strict-transport-security']).toBeDefined();
      expect(res.headers['x-content-type-options']).toBe('nosniff');
      expect(res.headers['x-frame-options']).toBeDefined();
      expect(res.headers['x-powered-by']).toBeUndefined();
    });
  });
  
  describe('Error masking', () => {
    it('should not expose internal errors in production', async () => {
      if (process.env.NODE_ENV !== 'production') return;
      
      // Trigger an internal error
      const res = await request(app)
        .post(GQL_ENDPOINT)
        .send({ query: '{ causeInternalError }' });
      
      const msg = res.body.errors?.[0]?.message;
      expect(msg).not.toMatch(/at Object/); // No stack traces
      expect(msg).not.toMatch(/node_modules/); // No file paths
    });
  });
});
```

---

## 📌 Step 654: Dependency Security

```json
// package.json
{
  "scripts": {
    "security:audit": "npm audit --audit-level=high",
    "security:check": "npx snyk test",
    "security:licenses": "npx license-checker --production --excludePrivatePackages"
  }
}
```

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  schedule:
    - cron: '0 2 * * 1'  # Every Monday 2 AM
  push:
    branches: [main]

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm audit --audit-level=high
      
  snyk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
```

---

## 📌 Step 655: สรุป Part 040

### เนื้อหาที่เรียนรู้

✅ Complete security checklist  
✅ Security configuration file  
✅ Automated security tests  
✅ Dependency security scanning  
✅ CI/CD security integration  

### Security Review Process

```
Weekly:
- Review failed auth attempts
- Check rate limit violations
- Audit dependency vulnerabilities

Monthly:
- Full security audit
- Penetration test (internal)
- Review audit logs
- Update dependencies

Quarterly:
- External penetration test
- Security training
- Review and update security policies
- Rotate secrets
```

### ในส่วนถัดไป

➡️ **[Part 041](./part-041.md)** — GraphQL Federation Advanced Patterns

---

*Part 040 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
