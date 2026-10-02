# Part 035 — OAuth 2.0 & Social Login 🔗

> **ระดับ:** Advanced | **เวลาเรียน:** 120 นาที | **ขั้นตอนที่:** 451–490

---

## 🎯 สิ่งที่จะได้เรียนรู้

- OAuth 2.0 flows
- Authorization Code Flow (recommended)
- PKCE (Proof Key for Code Exchange)
- Google OAuth integration
- GitHub OAuth integration
- Facebook OAuth integration
- Account linking
- GraphQL mutations สำหรับ OAuth
- Secure state parameter

---

## 📌 Step 451: OAuth 2.0 Basics

```
OAuth 2.0 Flows:

1. Authorization Code + PKCE (สำหรับ web/mobile apps)
   → ปลอดภัยที่สุด
   
2. Client Credentials (สำหรับ server-to-server)
   → ไม่มี user involvement

3. Implicit Flow (เก่า, ไม่แนะนำ)
   → ถูก replace ด้วย Authorization Code + PKCE

Authorization Code Flow:
1. User คลิก "Login with Google"
2. Redirect → Google
3. User approve
4. Google redirect กลับพร้อม code
5. Server แลก code → access token
6. ใช้ token ดึงข้อมูล user
```

---

## 📌 Step 452: Setup Passport.js

```bash
npm install passport passport-google-oauth20 passport-github2 passport-facebook
npm install express-session connect-redis
```

```javascript
// src/auth/passport.js
import passport from 'passport';
import { Strategy as GoogleStrategy } from 'passport-google-oauth20';
import { Strategy as GitHubStrategy } from 'passport-github2';
import { prisma } from '../db.js';

// Google Strategy
passport.use(new GoogleStrategy(
  {
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: `${process.env.BASE_URL}/auth/google/callback`,
    scope: ['profile', 'email'],
  },
  async (accessToken, refreshToken, profile, done) => {
    try {
      const email = profile.emails[0].value;
      
      // หา user ที่มี Google account นี้แล้ว
      let user = await prisma.user.findFirst({
        where: {
          oauthAccounts: {
            some: { provider: 'GOOGLE', providerId: profile.id }
          }
        },
        include: { oauthAccounts: true },
      });
      
      if (!user) {
        // ดู email ว่ามีใน DB แล้วไหม (account linking)
        user = await prisma.user.findUnique({ where: { email } });
        
        if (user) {
          // Link Google account ให้ user เดิม
          await prisma.oAuthAccount.create({
            data: {
              userId: user.id,
              provider: 'GOOGLE',
              providerId: profile.id,
              accessToken,
            },
          });
        } else {
          // สร้าง user ใหม่
          user = await prisma.user.create({
            data: {
              name: profile.displayName,
              email,
              emailVerified: true, // ← Google verify email แล้ว
              avatar: profile.photos[0]?.value,
              oauthAccounts: {
                create: [{
                  provider: 'GOOGLE',
                  providerId: profile.id,
                  accessToken,
                }],
              },
            },
          });
        }
      }
      
      done(null, user);
    } catch (error) {
      done(error);
    }
  }
));

// GitHub Strategy
passport.use(new GitHubStrategy(
  {
    clientID: process.env.GITHUB_CLIENT_ID,
    clientSecret: process.env.GITHUB_CLIENT_SECRET,
    callbackURL: `${process.env.BASE_URL}/auth/github/callback`,
    scope: ['user:email'],
  },
  async (accessToken, refreshToken, profile, done) => {
    try {
      const email = profile.emails?.[0]?.value || `${profile.username}@github.com`;
      
      let user = await prisma.user.findFirst({
        where: { oauthAccounts: { some: { provider: 'GITHUB', providerId: String(profile.id) } } },
      });
      
      if (!user) {
        user = await prisma.user.create({
          data: {
            name: profile.displayName || profile.username,
            email,
            avatar: profile.photos[0]?.value,
            oauthAccounts: {
              create: [{
                provider: 'GITHUB',
                providerId: String(profile.id),
                accessToken,
              }],
            },
          },
        });
      }
      
      done(null, user);
    } catch (err) {
      done(err);
    }
  }
));
```

---

## 📌 Step 453: OAuth Routes

```javascript
// src/routes/auth.js
import express from 'express';
import passport from 'passport';
import crypto from 'crypto';
import { generateAccessToken, createRefreshToken } from '../auth/jwt.service.js';
import { setRefreshTokenCookie } from '../middleware/cookies.js';

const router = express.Router();

// ใช้ state parameter เพื่อป้องกัน CSRF
function generateState() {
  return crypto.randomBytes(32).toString('hex');
}

// Google OAuth
router.get('/auth/google', (req, res, next) => {
  const state = generateState();
  
  // เก็บ state ใน session
  req.session.oauthState = state;
  
  // Optional: เก็บ return URL
  req.session.oauthReturnTo = req.query.returnTo || '/dashboard';
  
  passport.authenticate('google', { state })(req, res, next);
});

router.get('/auth/google/callback',
  (req, res, next) => {
    // ตรวจสอบ state (CSRF protection)
    if (req.query.state !== req.session.oauthState) {
      return res.status(403).json({ error: 'Invalid state parameter' });
    }
    
    passport.authenticate('google', { session: false })(req, res, next);
  },
  async (req, res) => {
    const user = req.user;
    
    // สร้าง tokens
    const accessToken = generateAccessToken(user);
    const refreshToken = await createRefreshToken(user.id);
    
    // Set refresh token cookie
    setRefreshTokenCookie(res, refreshToken);
    
    // Redirect ไป frontend พร้อม access token
    const returnTo = req.session.oauthReturnTo || '/dashboard';
    delete req.session.oauthState;
    delete req.session.oauthReturnTo;
    
    // Redirect ด้วย token (หรือส่งผ่าน postMessage)
    res.redirect(`${process.env.FRONTEND_URL}${returnTo}?token=${accessToken}`);
  }
);

// GitHub OAuth (คล้ายกัน)
router.get('/auth/github', passport.authenticate('github'));
router.get('/auth/github/callback',
  passport.authenticate('github', { session: false }),
  async (req, res) => {
    const user = req.user;
    const accessToken = generateAccessToken(user);
    const refreshToken = await createRefreshToken(user.id);
    setRefreshTokenCookie(res, refreshToken);
    res.redirect(`${process.env.FRONTEND_URL}/dashboard?token=${accessToken}`);
  }
);

export default router;
```

---

## 📌 Step 454: GraphQL Mutations สำหรับ OAuth

```graphql
type Mutation {
  # เริ่ม OAuth flow — return URL สำหรับ redirect
  initiateOAuth(provider: OAuthProvider!, returnTo: String): OAuthInitResponse!
  
  # หลัง OAuth callback — แลก code เป็น token
  exchangeOAuthCode(provider: OAuthProvider!, code: String!, state: String!): AuthPayload!
  
  # เชื่อม OAuth provider กับ account ปัจจุบัน
  linkOAuthAccount(provider: OAuthProvider!, code: String!, state: String!): User!
  
  # ยกเลิกการเชื่อม
  unlinkOAuthAccount(provider: OAuthProvider!): User!
}

enum OAuthProvider { GOOGLE GITHUB FACEBOOK TWITTER LINE }

type OAuthInitResponse {
  authorizationUrl: String!
  state: String!
}

type AuthPayload {
  accessToken: String!
  user: User!
}
```

```javascript
// Mutation implementations
const resolvers = {
  Mutation: {
    initiateOAuth: async (_, { provider, returnTo }, { redis }) => {
      const state = crypto.randomBytes(32).toString('hex');
      
      // Store state ใน Redis (10 minutes TTL)
      await redis.setex(
        `oauth:state:${state}`,
        600,
        JSON.stringify({ provider, returnTo: returnTo || '/' })
      );
      
      const urls = {
        GOOGLE: `https://accounts.google.com/o/oauth2/v2/auth?` + new URLSearchParams({
          client_id: process.env.GOOGLE_CLIENT_ID,
          redirect_uri: `${process.env.BASE_URL}/auth/google/callback`,
          response_type: 'code',
          scope: 'profile email',
          state,
          access_type: 'offline',
          prompt: 'consent',
        }),
        GITHUB: `https://github.com/login/oauth/authorize?` + new URLSearchParams({
          client_id: process.env.GITHUB_CLIENT_ID,
          redirect_uri: `${process.env.BASE_URL}/auth/github/callback`,
          scope: 'user:email',
          state,
        }),
      };
      
      return { authorizationUrl: urls[provider], state };
    },
    
    linkOAuthAccount: async (_, { provider, code, state }, { user, prisma }) => {
      if (!user) throw new GraphQLError('Must be logged in to link accounts');
      
      // Exchange code for token
      const tokenData = await exchangeCodeForToken(provider, code);
      const profile = await getProfileFromToken(provider, tokenData.access_token);
      
      // Check if this OAuth account is already linked to someone else
      const existing = await prisma.oAuthAccount.findFirst({
        where: { provider, providerId: profile.id }
      });
      
      if (existing && existing.userId !== user.sub) {
        throw new GraphQLError('This account is already linked to another user');
      }
      
      // Link to current user
      await prisma.oAuthAccount.upsert({
        where: { provider_providerId: { provider, providerId: profile.id } },
        create: { provider, providerId: profile.id, userId: user.sub, accessToken: tokenData.access_token },
        update: { accessToken: tokenData.access_token },
      });
      
      return prisma.user.findUnique({ where: { id: user.sub } });
    },
  },
};
```

---

## 📌 Step 455: PKCE Implementation

```javascript
// PKCE สำหรับ mobile/SPA apps (ไม่มี client secret)

import crypto from 'crypto';

// สร้าง code_verifier (random string 43-128 chars)
function generateCodeVerifier() {
  return crypto.randomBytes(64).toString('base64url').slice(0, 128);
}

// สร้าง code_challenge จาก verifier
function generateCodeChallenge(verifier) {
  return crypto
    .createHash('sha256')
    .update(verifier)
    .digest('base64url');
}

// Flow:
// 1. Client สร้าง verifier + challenge
// 2. ส่ง challenge ไปกับ auth request
// 3. หลัง callback, ส่ง verifier เพื่อ exchange code
// 4. Auth server ตรวจสอบว่า hash(verifier) == challenge

// Example request
const verifier = generateCodeVerifier();
const challenge = generateCodeChallenge(verifier);

const authUrl = `https://accounts.google.com/o/oauth2/v2/auth?` + new URLSearchParams({
  client_id: CLIENT_ID,
  redirect_uri: REDIRECT_URI,
  response_type: 'code',
  scope: 'profile email',
  code_challenge: challenge,
  code_challenge_method: 'S256',
  state: state,
});
```

---

## 📌 Step 456: สรุป Part 035

### เนื้อหาที่เรียนรู้

✅ OAuth 2.0 Authorization Code Flow  
✅ PKCE  
✅ Google + GitHub OAuth integration  
✅ State parameter (CSRF protection)  
✅ Account linking  
✅ GraphQL mutations สำหรับ OAuth  

### Security Best Practices

```
1. ใช้ state parameter เสมอ (CSRF)
2. ใช้ PKCE สำหรับ public clients
3. ตรวจสอบ state ก่อน exchange code
4. เก็บ state ใน Redis (server-side)
5. Client secret ต้องไม่อยู่ใน frontend
6. ตรวจสอบ email domain ถ้าต้องการ
7. Handle account linking อย่างระมัดระวัง
```

### ในส่วนถัดไป

➡️ **[Part 036](./part-036.md)** — Advanced RBAC & Permission Systems

---

*Part 035 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
