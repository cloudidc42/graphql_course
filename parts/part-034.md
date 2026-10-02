# Part 034 — JWT Security Best Practices 🔑

> **ระดับ:** Intermediate-Advanced | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 411–450

---

## 🎯 สิ่งที่จะได้เรียนรู้

- JWT vulnerabilities
- Algorithm confusion attacks
- Token storage security
- Short-lived access tokens + refresh tokens
- Token rotation
- Token revocation strategies
- Key rotation
- JWT best practices สำหรับ GraphQL

---

## 📌 Step 411: JWT Vulnerabilities

```
⚠️ JWT Attack Vectors:

1. Algorithm confusion (alg=none)
2. RS256 → HS256 attack
3. Weak secret keys
4. Long expiry times
5. No token revocation
6. Storing in localStorage (XSS risk)
7. Leaking in URL
8. Missing audience/issuer validation
```

---

## 📌 Step 412: Algorithm Confusion Attack (Educational)

```javascript
// ⚠️ ตัวอย่าง attack — เพื่อเข้าใจ ไม่ใช่ทำจริง

// Attacker เปลี่ยน alg จาก RS256 → HS256
// ใช้ public key เป็น HMAC secret (public key หาได้จาก JWKS endpoint)
const header = { alg: 'HS256', typ: 'JWT' };
const payload = { sub: 'admin', role: 'ADMIN' };
const fakeToken = sign(payload, PUBLIC_KEY, { algorithm: 'HS256' });
// Server ที่ไม่ validate algorithm จะ verify สำเร็จ!

// ✅ ป้องกัน: ระบุ algorithm ที่อนุญาตเสมอ
import jwt from 'jsonwebtoken';

function verifyToken(token) {
  return jwt.verify(token, process.env.JWT_PUBLIC_KEY, {
    algorithms: ['RS256'],  // ← ระบุเฉพาะ algorithm ที่อนุญาต
    issuer: 'https://api.myapp.com',
    audience: 'myapp-client',
  });
}
```

---

## 📌 Step 413: Secure JWT Implementation

```javascript
// src/auth/jwt.service.js
import jwt from 'jsonwebtoken';
import crypto from 'crypto';

// สร้าง RSA key pair (ทำครั้งเดียวตอน setup)
// openssl genrsa -out private.key 2048
// openssl rsa -in private.key -pubout -out public.key

const ACCESS_TOKEN_TTL = '15m';   // 15 minutes
const REFRESH_TOKEN_TTL = '7d';   // 7 days

function generateAccessToken(user) {
  return jwt.sign(
    {
      sub: user.id,
      email: user.email,
      role: user.role,
      jti: crypto.randomUUID(), // JWT ID สำหรับ revocation
    },
    process.env.JWT_PRIVATE_KEY,
    {
      algorithm: 'RS256',
      expiresIn: ACCESS_TOKEN_TTL,
      issuer: process.env.JWT_ISSUER,
      audience: process.env.JWT_AUDIENCE,
      notBefore: '0s',
    }
  );
}

function generateRefreshToken(user) {
  const token = crypto.randomBytes(64).toString('hex'); // Opaque token
  return token; // ไม่ใช้ JWT สำหรับ refresh token!
}

async function verifyAccessToken(token) {
  try {
    const payload = jwt.verify(token, process.env.JWT_PUBLIC_KEY, {
      algorithms: ['RS256'],
      issuer: process.env.JWT_ISSUER,
      audience: process.env.JWT_AUDIENCE,
    });
    
    // ตรวจสอบ revocation list
    const isRevoked = await tokenBlacklist.has(payload.jti);
    if (isRevoked) throw new Error('Token revoked');
    
    return payload;
  } catch (error) {
    throw new GraphQLError('Invalid or expired token', {
      extensions: { code: 'UNAUTHENTICATED' }
    });
  }
}

export { generateAccessToken, generateRefreshToken, verifyAccessToken };
```

---

## 📌 Step 414: Refresh Token Rotation

```javascript
// src/auth/refresh.service.js
// Refresh token rotation: ทุก refresh ได้ token ใหม่ + ทำลาย token เก่า

import { prisma } from '../db.js';
import crypto from 'crypto';
import { addDays } from 'date-fns';

export async function createRefreshToken(userId) {
  // ทำลาย refresh token เก่าทั้งหมด (single session)
  await prisma.refreshToken.deleteMany({ where: { userId } });
  
  const token = crypto.randomBytes(64).toString('hex');
  const hashedToken = await bcrypt.hash(token, 10);
  
  await prisma.refreshToken.create({
    data: {
      token: hashedToken,
      userId,
      expiresAt: addDays(new Date(), 7),
      userAgent: currentUserAgent,
      ipAddress: currentIp,
    },
  });
  
  return token; // Return raw token (เก็บ hash ใน DB)
}

export async function rotateRefreshToken(rawToken) {
  const tokens = await prisma.refreshToken.findMany({
    where: { expiresAt: { gt: new Date() } },
    include: { user: true },
  });
  
  // Find matching token
  let matchedToken = null;
  for (const t of tokens) {
    if (await bcrypt.compare(rawToken, t.token)) {
      matchedToken = t;
      break;
    }
  }
  
  if (!matchedToken) {
    // Token reuse detected! Invalidate ALL tokens for this user
    // (possible token theft)
    throw new GraphQLError('Invalid refresh token', {
      extensions: { code: 'UNAUTHENTICATED' }
    });
  }
  
  const user = matchedToken.user;
  
  // Delete old token
  await prisma.refreshToken.delete({ where: { id: matchedToken.id } });
  
  // Generate new tokens
  const newAccessToken = generateAccessToken(user);
  const newRefreshToken = await createRefreshToken(user.id);
  
  return { accessToken: newAccessToken, refreshToken: newRefreshToken };
}
```

---

## 📌 Step 415: Token Revocation

```javascript
// src/auth/blacklist.service.js
// Revoke access tokens (ก่อน expire) ด้วย Redis blacklist

import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL);

export const tokenBlacklist = {
  async add(jti, expiresIn) {
    // Store จนถึง expire time ของ token
    await redis.setex(`blacklist:${jti}`, expiresIn, '1');
  },
  
  async has(jti) {
    return (await redis.exists(`blacklist:${jti}`)) === 1;
  },
};

// Resolver
const resolvers = {
  Mutation: {
    logout: async (_, __, { user, token }) => {
      if (!user) return false;
      
      // Get token expiry
      const payload = jwt.decode(token);
      const ttl = payload.exp - Math.floor(Date.now() / 1000);
      
      if (ttl > 0) {
        // Add to blacklist until token expires
        await tokenBlacklist.add(payload.jti, ttl);
      }
      
      // Delete refresh tokens
      await prisma.refreshToken.deleteMany({ where: { userId: user.sub } });
      
      return true;
    },
    
    logoutAllDevices: async (_, __, { user }) => {
      if (!user) throw new GraphQLError('Unauthenticated');
      
      // Delete ALL refresh tokens
      await prisma.refreshToken.deleteMany({ where: { userId: user.sub } });
      
      // Mark user as requiring re-auth (bump token version)
      await prisma.user.update({
        where: { id: user.sub },
        data: { tokenVersion: { increment: 1 } },
      });
      
      return true;
    },
  },
};
```

---

## 📌 Step 416: Token Version Strategy

```javascript
// ป้องกัน stale tokens หลัง password change

// เพิ่ม tokenVersion ใน User model
// เมื่อ password เปลี่ยน: tokenVersion++
// ใน JWT: เก็บ tokenVersion
// เมื่อ verify: ตรวจสอบ tokenVersion ตรงกับ DB

function generateAccessToken(user) {
  return jwt.sign({
    sub: user.id,
    role: user.role,
    ver: user.tokenVersion, // ← token version
    jti: crypto.randomUUID(),
  }, process.env.JWT_PRIVATE_KEY, { algorithm: 'RS256', expiresIn: '15m' });
}

async function verifyAndGetUser(token) {
  const payload = jwt.verify(token, process.env.JWT_PUBLIC_KEY, {
    algorithms: ['RS256'],
  });
  
  // Check token version
  const user = await prisma.user.findUnique({ where: { id: payload.sub } });
  
  if (!user || user.tokenVersion !== payload.ver) {
    throw new GraphQLError('Token invalidated', {
      extensions: { code: 'UNAUTHENTICATED' }
    });
  }
  
  return user;
}

// เมื่อ user เปลี่ยน password
await prisma.user.update({
  where: { id: userId },
  data: {
    password: hashedNewPassword,
    tokenVersion: { increment: 1 }, // ← invalidate all existing tokens
  },
});
```

---

## 📌 Step 417: สรุป Part 034

### เนื้อหาที่เรียนรู้

✅ JWT algorithm confusion attack  
✅ RSA keys สำหรับ RS256  
✅ Short-lived access tokens  
✅ Opaque refresh tokens  
✅ Token rotation  
✅ Token revocation ด้วย Redis blacklist  
✅ Token version strategy  

### JWT Best Practices

```
1. ใช้ RS256 ไม่ใช่ HS256 (ถ้า scale ออก)
2. Access token: 15 นาที max
3. Refresh token: opaque (ไม่ใช่ JWT)
4. ทำ token rotation ทุก refresh
5. เก็บ access token ใน memory (ไม่ใช่ localStorage)
6. เก็บ refresh token ใน httpOnly cookie
7. ป้องกัน token reuse detection
8. ทำ token version เมื่อ password เปลี่ยน
```

### ในส่วนถัดไป

➡️ **[Part 035](./part-035.md)** — OAuth 2.0 & Social Login

---

*Part 034 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
