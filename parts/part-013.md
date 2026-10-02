# Part 013 — Authentication & JWT 🔐

> **ระดับ:** Intermediate | **เวลาเรียน:** 120 นาที | **ขั้นตอนที่:** 371–420

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Authentication vs Authorization
- JWT (JSON Web Token) คืออะไร
- Login mutation
- Token verification ใน context
- Refresh tokens
- Role-based access control (RBAC)
- Permission-based access
- Cookie vs Bearer token
- Logout และ token revocation

---

## 📌 Step 371: Authentication vs Authorization

```
Authentication (ตรวจสอบตัวตน):
  "คุณเป็นใคร?"
  → Login, JWT, OAuth
  → ถ้าไม่ผ่าน: 401 Unauthorized

Authorization (ตรวจสอบสิทธิ์):
  "คุณมีสิทธิ์ทำอะไร?"
  → Role check, Permission check
  → ถ้าไม่ผ่าน: 403 Forbidden
```

---

## 📌 Step 372: JWT พื้นฐาน

```
JWT (JSON Web Token) ประกอบด้วย 3 ส่วน คั่นด้วย "."

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.    ← Header (Base64)
eyJzdWIiOiJ1c2VyXzEyMyIsInJvbGUiOiJVU0VSIn0. ← Payload (Base64)
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← Signature (HMAC)

Decode Payload:
{
  "sub": "user_123",      ← subject (user ID)
  "role": "USER",
  "iat": 1700000000,      ← issued at
  "exp": 1700086400       ← expiry
}
```

---

## 📌 Step 373: Schema สำหรับ Auth

```graphql
type AuthPayload {
  accessToken: String!
  refreshToken: String!
  expiresIn: Int!
  user: User!
}

type LogoutPayload {
  success: Boolean!
  message: String!
}

input LoginInput {
  email: String!
  password: String!
}

input RegisterInput {
  name: String!
  email: String!
  password: String!
}

input RefreshTokenInput {
  refreshToken: String!
}

type Mutation {
  login(input: LoginInput!): AuthPayload!
  register(input: RegisterInput!): AuthPayload!
  logout: LogoutPayload! @auth
  refreshToken(input: RefreshTokenInput!): AuthPayload!
  changePassword(oldPassword: String!, newPassword: String!): Boolean! @auth
}

type Query {
  me: User! @auth
}
```

---

## 📌 Step 374: Login Mutation

```javascript
// src/resolvers/auth.resolver.js
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import { GraphQLError } from 'graphql';

const JWT_SECRET = process.env.JWT_SECRET;
const JWT_REFRESH_SECRET = process.env.JWT_REFRESH_SECRET;
const ACCESS_TOKEN_TTL = '15m';
const REFRESH_TOKEN_TTL = '7d';

function generateTokens(user) {
  const payload = {
    sub: user.id,
    email: user.email,
    role: user.role,
  };
  
  const accessToken = jwt.sign(payload, JWT_SECRET, {
    expiresIn: ACCESS_TOKEN_TTL,
  });
  
  const refreshToken = jwt.sign(
    { sub: user.id },
    JWT_REFRESH_SECRET,
    { expiresIn: REFRESH_TOKEN_TTL }
  );
  
  return { accessToken, refreshToken };
}

export const authResolvers = {
  Mutation: {
    login: async (_, { input }) => {
      const { email, password } = input;
      
      // หา user
      const user = await User.findOne({ where: { email: email.toLowerCase() } });
      if (!user) {
        // ไม่บอกว่า email ไม่มีอยู่ (security)
        throw new GraphQLError('Invalid email or password', {
          extensions: { code: 'BAD_USER_INPUT' }
        });
      }
      
      // ตรวจสอบ password
      const isValid = await bcrypt.compare(password, user.password);
      if (!isValid) {
        // Increment failed attempts
        await user.incrementLoginAttempts();
        
        throw new GraphQLError('Invalid email or password', {
          extensions: { code: 'BAD_USER_INPUT' }
        });
      }
      
      // Check account status
      if (!user.isActive) {
        throw new GraphQLError('Account is deactivated', {
          extensions: { code: 'FORBIDDEN' }
        });
      }
      
      // Reset failed attempts
      await user.update({ loginAttempts: 0, lastLoginAt: new Date() });
      
      // Generate tokens
      const { accessToken, refreshToken } = generateTokens(user);
      
      // Save refresh token hash ใน database
      const refreshTokenHash = await bcrypt.hash(refreshToken, 10);
      await RefreshToken.create({
        userId: user.id,
        tokenHash: refreshTokenHash,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
      });
      
      return {
        accessToken,
        refreshToken,
        expiresIn: 15 * 60, // 15 minutes in seconds
        user,
      };
    },
    
    register: async (_, { input }) => {
      const { name, email, password } = input;
      
      // Check duplicate
      const existing = await User.findOne({ where: { email: email.toLowerCase() } });
      if (existing) {
        throw new GraphQLError('Email already registered', {
          extensions: { code: 'CONFLICT' }
        });
      }
      
      // Hash password
      const hashedPassword = await bcrypt.hash(password, 12);
      
      // Create user
      const user = await User.create({
        name,
        email: email.toLowerCase(),
        password: hashedPassword,
        role: 'USER',
      });
      
      const { accessToken, refreshToken } = generateTokens(user);
      
      await RefreshToken.create({
        userId: user.id,
        tokenHash: await bcrypt.hash(refreshToken, 10),
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
      });
      
      return {
        accessToken,
        refreshToken,
        expiresIn: 15 * 60,
        user,
      };
    },
    
    refreshToken: async (_, { input: { refreshToken } }) => {
      // Verify refresh token
      let payload;
      try {
        payload = jwt.verify(refreshToken, JWT_REFRESH_SECRET);
      } catch {
        throw new GraphQLError('Invalid or expired refresh token', {
          extensions: { code: 'UNAUTHENTICATED' }
        });
      }
      
      // หา user
      const user = await User.findById(payload.sub);
      if (!user) {
        throw new GraphQLError('User not found', {
          extensions: { code: 'NOT_FOUND' }
        });
      }
      
      // ตรวจสอบ refresh token ใน database
      const storedTokens = await RefreshToken.findAll({
        where: { userId: user.id, isRevoked: false }
      });
      
      let tokenRecord = null;
      for (const stored of storedTokens) {
        const matches = await bcrypt.compare(refreshToken, stored.tokenHash);
        if (matches) {
          tokenRecord = stored;
          break;
        }
      }
      
      if (!tokenRecord) {
        // Possible token theft! Revoke all tokens
        await RefreshToken.update(
          { isRevoked: true },
          { where: { userId: user.id } }
        );
        throw new GraphQLError('Refresh token not found or revoked', {
          extensions: { code: 'UNAUTHENTICATED' }
        });
      }
      
      // Revoke old token
      await tokenRecord.update({ isRevoked: true });
      
      // Generate new tokens
      const tokens = generateTokens(user);
      
      await RefreshToken.create({
        userId: user.id,
        tokenHash: await bcrypt.hash(tokens.refreshToken, 10),
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
      });
      
      return {
        ...tokens,
        expiresIn: 15 * 60,
        user,
      };
    },
    
    logout: async (_, __, { user }) => {
      // Revoke all refresh tokens ของ user นี้
      await RefreshToken.update(
        { isRevoked: true },
        { where: { userId: user.id } }
      );
      
      return { success: true, message: 'Logged out successfully' };
    },
  },
  
  Query: {
    me: async (_, __, { user }) => {
      return User.findById(user.sub);
    },
  },
};
```

---

## 📌 Step 375: Context — Token Verification

```javascript
// src/context.js
import jwt from 'jsonwebtoken';

export async function createContext({ req }) {
  const context = {
    ip: req.ip,
    userAgent: req.headers['user-agent'],
  };
  
  // Extract token
  const authHeader = req.headers.authorization;
  const token = authHeader?.startsWith('Bearer ')
    ? authHeader.slice(7)
    : null;
  
  if (!token) return context;
  
  try {
    // Verify token
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    
    // Attach user to context (lazy load full user if needed)
    context.userId = payload.sub;
    context.user = payload; // token payload (ไม่ต้อง DB query)
    context.getUser = () => User.findById(payload.sub); // lazy load
  } catch (err) {
    // Token invalid/expired — ไม่ set user ใน context
    if (err.name === 'TokenExpiredError') {
      context.tokenExpired = true;
    }
  }
  
  return context;
}
```

---

## 📌 Step 376: RBAC — Role-Based Access Control

```javascript
// src/auth/rbac.js

const ROLES = {
  ADMIN: 'ADMIN',
  MODERATOR: 'MODERATOR',
  USER: 'USER',
  GUEST: 'GUEST',
};

// Hierarchy: ADMIN > MODERATOR > USER > GUEST
const ROLE_HIERARCHY = {
  [ROLES.ADMIN]: 3,
  [ROLES.MODERATOR]: 2,
  [ROLES.USER]: 1,
  [ROLES.GUEST]: 0,
};

export function hasRole(userRole, requiredRole) {
  return ROLE_HIERARCHY[userRole] >= ROLE_HIERARCHY[requiredRole];
}

// Permissions
const PERMISSIONS = {
  // User management
  'user:read:own': [ROLES.USER, ROLES.MODERATOR, ROLES.ADMIN],
  'user:read:all': [ROLES.ADMIN],
  'user:update:own': [ROLES.USER, ROLES.MODERATOR, ROLES.ADMIN],
  'user:update:all': [ROLES.ADMIN],
  'user:delete': [ROLES.ADMIN],
  
  // Content management
  'post:create': [ROLES.USER, ROLES.MODERATOR, ROLES.ADMIN],
  'post:update:own': [ROLES.USER, ROLES.MODERATOR, ROLES.ADMIN],
  'post:update:all': [ROLES.MODERATOR, ROLES.ADMIN],
  'post:delete:own': [ROLES.USER, ROLES.MODERATOR, ROLES.ADMIN],
  'post:delete:all': [ROLES.MODERATOR, ROLES.ADMIN],
  
  // Admin
  'system:config': [ROLES.ADMIN],
  'analytics:view': [ROLES.MODERATOR, ROLES.ADMIN],
};

export function hasPermission(userRole, permission) {
  const allowedRoles = PERMISSIONS[permission] || [];
  return allowedRoles.includes(userRole);
}

// Helper functions สำหรับ resolvers
export function requireAuth(context) {
  if (!context.user) {
    throw new GraphQLError('You must be logged in', {
      extensions: { code: 'UNAUTHENTICATED' }
    });
  }
  return context.user;
}

export function requireRole(context, role) {
  const user = requireAuth(context);
  if (!hasRole(user.role, role)) {
    throw new GraphQLError(`Requires ${role} role`, {
      extensions: { code: 'FORBIDDEN' }
    });
  }
  return user;
}

export function requirePermission(context, permission) {
  const user = requireAuth(context);
  if (!hasPermission(user.role, permission)) {
    throw new GraphQLError(`No permission: ${permission}`, {
      extensions: { code: 'FORBIDDEN', permission }
    });
  }
  return user;
}
```

### ใช้ใน Resolvers

```javascript
import { requireAuth, requireRole, requirePermission } from '../auth/rbac.js';

const resolvers = {
  Query: {
    me: (_, __, context) => {
      const user = requireAuth(context);
      return User.findById(user.sub);
    },
    
    allUsers: (_, __, context) => {
      requireRole(context, 'ADMIN');
      return User.findAll();
    },
    
    analyticsData: (_, __, context) => {
      requirePermission(context, 'analytics:view');
      return Analytics.getData();
    },
  },
  
  Mutation: {
    updateUser: async (_, { id, input }, context) => {
      const currentUser = requireAuth(context);
      
      // Admin หรือ owner เท่านั้น
      if (id !== currentUser.sub) {
        requirePermission(context, 'user:update:all');
      }
      
      return User.update(id, input);
    },
    
    deleteUser: (_, { id }, context) => {
      requirePermission(context, 'user:delete');
      return User.delete(id);
    },
  },
};
```

---

## 📌 Step 377: Cookie-based Auth (HttpOnly)

```javascript
// ใช้ HttpOnly cookies แทน localStorage (ปลอดภัยกว่า)
export const authResolvers = {
  Mutation: {
    login: async (_, { input }, { res }) => {
      // ... ตรวจสอบ credentials
      const { accessToken, refreshToken } = generateTokens(user);
      
      // Set HttpOnly cookies
      res.cookie('accessToken', accessToken, {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'strict',
        maxAge: 15 * 60 * 1000, // 15 minutes
      });
      
      res.cookie('refreshToken', refreshToken, {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'strict',
        maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
        path: '/graphql/refresh', // เฉพาะ refresh endpoint
      });
      
      return { user };
    },
    
    logout: async (_, __, { res, user }) => {
      await RefreshToken.revokeAll(user.sub);
      
      res.clearCookie('accessToken');
      res.clearCookie('refreshToken');
      
      return { success: true };
    },
  },
};

// Context สำหรับ cookie auth
export async function createContext({ req, res }) {
  const token = req.cookies?.accessToken || 
    req.headers.authorization?.replace('Bearer ', '');
  
  // ...
}
```

---

## 📌 Step 378: สรุป Part 013

### เนื้อหาที่เรียนรู้

✅ Authentication vs Authorization  
✅ JWT structure และ ทำงานอย่างไร  
✅ Login/Register mutations  
✅ Refresh token pattern  
✅ Token verification ใน context  
✅ RBAC (Role-Based Access Control)  
✅ Permission-based access  
✅ Cookie-based auth  

### Homework

1. implement full auth flow: register → login → me → logout
2. เพิ่ม refresh token rotation
3. implement RBAC สำหรับ blog API

### ในส่วนถัดไป

➡️ **[Part 014](./part-014.md)** — Pagination: Cursor & Offset

---

*Part 013 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
