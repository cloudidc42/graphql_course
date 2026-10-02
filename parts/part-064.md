# Part 064 — Production Deployment & Zero-Downtime Updates 🚀

> **ระดับ:** Expert | **เวลาเรียน:** 80 นาที | **ขั้นตอนที่:** 1461–1500

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Zero-downtime deployment strategies
- Blue/Green deployment
- Canary releases
- Rolling updates
- Health checks & readiness probes
- Graceful shutdown
- Database migrations with deployment
- Rollback strategies

---

## 📌 Step 1461: Graceful Shutdown

```typescript
// src/server.ts

import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import express from 'express';
import http from 'http';

const app = express();
const httpServer = http.createServer(app);
const apolloServer = new ApolloServer({ typeDefs, resolvers });

await apolloServer.start();

app.use('/graphql', expressMiddleware(apolloServer));

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: process.env.APP_VERSION,
  });
});

// Readiness probe: check DB, Redis, etc.
app.get('/ready', async (req, res) => {
  try {
    await prisma.$queryRaw`SELECT 1`;
    await redis.ping();
    
    res.json({ status: 'ready' });
  } catch (err) {
    res.status(503).json({ status: 'not ready', error: String(err) });
  }
});

const PORT = parseInt(process.env.PORT ?? '4000');
httpServer.listen(PORT, () => {
  console.log(`Server started on port ${PORT}`);
});

// ─── Graceful Shutdown ───────────────────────────────────
let isShuttingDown = false;

async function gracefulShutdown(signal: string) {
  if (isShuttingDown) return;
  isShuttingDown = true;
  
  console.log(`Received ${signal}, starting graceful shutdown...`);
  
  // 1. Stop accepting new requests
  httpServer.close(() => {
    console.log('HTTP server closed');
  });
  
  // 2. Wait for in-flight requests (30s timeout)
  const shutdownTimeout = setTimeout(() => {
    console.error('Shutdown timeout, forcing exit');
    process.exit(1);
  }, 30_000);
  
  try {
    // 3. Drain subscriptions
    await apolloServer.stop();
    
    // 4. Close DB connections
    await prisma.$disconnect();
    
    // 5. Close Redis
    await redis.quit();
    
    // 6. Clean up other resources
    
    clearTimeout(shutdownTimeout);
    console.log('Graceful shutdown complete');
    process.exit(0);
    
  } catch (err) {
    console.error('Error during shutdown:', err);
    process.exit(1);
  }
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

---

## 📌 Step 1462: Dockerfile สำหรับ Production

```dockerfile
# Dockerfile
FROM node:22-alpine AS base
WORKDIR /app

# Install dependencies
FROM base AS deps
COPY package*.json ./
RUN npm ci --only=production

# Build
FROM base AS builder
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production image
FROM base AS runner
ENV NODE_ENV=production

# Non-root user สำหรับ security
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 appuser

COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/prisma ./prisma

USER appuser

EXPOSE 4000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD wget -qO- http://localhost:4000/health || exit 1

CMD ["node", "dist/server.js"]
```

```yaml
# docker-compose.prod.yml
version: '3.9'

services:
  api:
    image: myapp/graphql-api:${VERSION:-latest}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1          # Rolling update: 1 at a time
        delay: 15s
        failure_action: rollback
        monitor: 30s
        max_failure_ratio: 0.3
      rollback_config:
        parallelism: 1
        delay: 10s
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:4000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    environment:
      NODE_ENV: production
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
    secrets:
      - db_password
      - jwt_secret
```

---

## 📌 Step 1463: Blue/Green Deployment

```bash
#!/bin/bash
# scripts/blue-green-deploy.sh

set -e

NEW_VERSION="${1:-latest}"
CURRENT=$(docker service inspect --format '{{.Spec.Labels.slot}}' api-service 2>/dev/null || echo "blue")
NEW_SLOT=$([ "$CURRENT" = "blue" ] && echo "green" || echo "blue")

echo "Current slot: $CURRENT"
echo "Deploying to: $NEW_SLOT (version $NEW_VERSION)"

# 1. Deploy to new slot
docker service create \
  --name "api-${NEW_SLOT}" \
  --label slot="${NEW_SLOT}" \
  --label version="${NEW_VERSION}" \
  --replicas 3 \
  --network backend \
  myapp/graphql-api:${NEW_VERSION}

# 2. Wait for new slot to be healthy
echo "Waiting for ${NEW_SLOT} to be healthy..."
ATTEMPTS=0
MAX_ATTEMPTS=30

until curl -sf "http://api-${NEW_SLOT}/health" | grep -q '"status":"ok"'; do
  ATTEMPTS=$((ATTEMPTS + 1))
  if [ $ATTEMPTS -ge $MAX_ATTEMPTS ]; then
    echo "Health check failed after ${MAX_ATTEMPTS} attempts"
    docker service rm "api-${NEW_SLOT}"
    exit 1
  fi
  echo "Attempt ${ATTEMPTS}/${MAX_ATTEMPTS}..."
  sleep 10
done

echo "${NEW_SLOT} is healthy"

# 3. Switch traffic (update load balancer config)
# For Nginx, HAProxy, or cloud LB: update upstream
./scripts/switch-traffic.sh "$NEW_SLOT"

echo "Traffic switched to ${NEW_SLOT}"

# 4. Wait and verify
sleep 30

ERROR_RATE=$(./scripts/check-error-rate.sh)
if [ "$ERROR_RATE" -gt "5" ]; then
  echo "High error rate (${ERROR_RATE}%), rolling back..."
  ./scripts/switch-traffic.sh "$CURRENT"
  docker service rm "api-${NEW_SLOT}"
  exit 1
fi

# 5. Remove old slot
echo "Deployment successful. Removing old slot: ${CURRENT}"
docker service rm "api-${CURRENT}"
```

---

## 📌 Step 1464: Database Migration Strategy

```typescript
// src/migrations/safe-migration.ts

// ✅ Safe migration patterns

// STEP 1: Add new nullable column (backward-compatible)
// migration: 20240101_add_user_phone.sql
// ALTER TABLE users ADD COLUMN phone VARCHAR(20);

// STEP 2: Backfill data (can run while live)
async function backfillPhoneData() {
  const batchSize = 1000;
  let offset = 0;
  
  while (true) {
    const users = await prisma.user.findMany({
      where: { phone: null, legacyPhone: { not: null } },
      take: batchSize,
      skip: offset,
      select: { id: true, legacyPhone: true },
    });
    
    if (!users.length) break;
    
    await prisma.$transaction(
      users.map(u => 
        prisma.user.update({
          where: { id: u.id },
          data: { phone: normalizePhone(u.legacyPhone!) },
        })
      )
    );
    
    offset += users.length;
    console.log(`Migrated ${offset} users`);
    
    // Rate limit: prevent DB overload
    await new Promise(r => setTimeout(r, 100));
  }
}

// STEP 3: Add NOT NULL constraint (after backfill)
// ALTER TABLE users ALTER COLUMN phone SET NOT NULL;

// STEP 4: Drop old column (after transition period)
// ALTER TABLE users DROP COLUMN legacy_phone;
```

```yaml
# k8s/job-migration.yaml
# Run migrations as K8s Job before deployment
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate-${VERSION}
spec:
  template:
    spec:
      initContainers:
        - name: wait-for-db
          image: busybox
          command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 2; done']
      containers:
        - name: migrate
          image: myapp/graphql-api:${VERSION}
          command: ["npx", "prisma", "migrate", "deploy"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: url
      restartPolicy: Never
  backoffLimit: 3
```

---

## 📌 Step 1465: Canary Release

```typescript
// src/middleware/canary.ts

// Canary: ส่ง % ของ traffic ไปยัง new version
export function canaryMiddleware(canaryPercentage: number) {
  return (req: Request, res: Response, next: NextFunction) => {
    // Consistent hashing: ใช้ user ID เพื่อ sticky canary
    const userId = req.user?.id ?? req.ip ?? '';
    const hash = simpleHash(userId) % 100;
    
    if (hash < canaryPercentage) {
      req.headers['x-canary'] = 'true';
      // Route to canary instances
      req.headers['x-upstream'] = 'canary';
    } else {
      req.headers['x-upstream'] = 'stable';
    }
    
    next();
  };
}

function simpleHash(str: string): number {
  let hash = 0;
  for (let i = 0; i < str.length; i++) {
    hash = ((hash << 5) - hash) + str.charCodeAt(i);
    hash |= 0;
  }
  return Math.abs(hash);
}

// Feature flags สำหรับ canary features
export function isCanaryUser(userId: string): boolean {
  const percentage = parseFloat(process.env.CANARY_PERCENTAGE ?? '0');
  return simpleHash(userId) % 100 < percentage;
}
```

---

## 📌 Step 1466: สรุป Part 064

### เนื้อหาที่เรียนรู้

✅ Graceful shutdown  
✅ Health checks & readiness probes  
✅ Production Dockerfile  
✅ Blue/Green deployment  
✅ Canary releases  
✅ Safe database migrations  
✅ Rollback strategies  

### Deployment Decision Tree

```
Is it a database schema change?
├─ Yes → Use multi-step migration (add → backfill → constraint → drop)
└─ No → Is it high risk?
    ├─ Yes → Blue/Green deployment
    └─ No → Is it new feature?
        ├─ Yes → Canary release (5% → 20% → 50% → 100%)
        └─ No → Rolling update
```

### ในส่วนถัดไป

➡️ **[Part 065](./part-065.md)** — Kubernetes & Container Orchestration

---

*Part 064 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
