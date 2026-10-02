# Part 026 — Docker & Kubernetes Deployment 🐳

> **ระดับ:** Advanced | **เวลาเรียน:** 150 นาที | **ขั้นตอนที่:** 911–960

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Docker สำหรับ GraphQL server
- docker-compose สำหรับ development
- Multi-stage build
- Health checks
- Kubernetes basics
- GraphQL server deployment ใน K8s
- Horizontal scaling
- Zero-downtime deployment
- ConfigMaps + Secrets

---

## 📌 Step 911: Dockerfile สำหรับ GraphQL Server

```dockerfile
# Dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY prisma ./prisma/

# Install ALL dependencies (including dev)
RUN npm ci

# Copy source
COPY . .

# Build TypeScript
RUN npm run build

# Generate Prisma client
RUN npx prisma generate

# Stage 2: Production
FROM node:20-alpine AS production

WORKDIR /app

# Install only production deps
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy build output
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma
COPY prisma ./prisma/

# Non-root user (security)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

ENV NODE_ENV=production
ENV PORT=4000

EXPOSE 4000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD node -e "require('http').get('http://localhost:4000/health', (res) => { process.exit(res.statusCode === 200 ? 0 : 1) })"

CMD ["node", "dist/index.js"]
```

---

## 📌 Step 912: docker-compose สำหรับ Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      target: builder  # ← ใช้ builder stage สำหรับ dev
    ports:
      - "4000:4000"
    volumes:
      - ./src:/app/src    # Hot reload
      - ./prisma:/app/prisma
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/myapp
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=dev-secret-not-for-production
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: npm run dev
    networks:
      - app-network
    
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network
  
  # Development tools
  prisma-studio:
    build:
      context: .
      target: builder
    ports:
      - "5555:5555"
    environment:
      DATABASE_URL: postgresql://postgres:password@postgres:5432/myapp
    command: npx prisma studio --hostname 0.0.0.0
    depends_on:
      - postgres
    networks:
      - app-network
  
  redis-commander:
    image: rediscommander/redis-commander:latest
    ports:
      - "8081:8081"
    environment:
      REDIS_HOSTS: redis:redis:6379
    depends_on:
      - redis
    networks:
      - app-network

volumes:
  postgres-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

---

## 📌 Step 913: docker-compose.prod.yml

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    image: myapp/graphql-api:${APP_VERSION}
    restart: unless-stopped
    deploy:
      replicas: 3
      update_config:
        order: rolling-update
        failure_action: rollback
        delay: 10s
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    ports:
      - "4000:4000"
    environment:
      - NODE_ENV=production
    env_file:
      - .env.production
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - app-network
  
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./ssl:/etc/ssl
    depends_on:
      - app
    networks:
      - app-network

networks:
  app-network:
    external: true
```

---

## 📌 Step 914: Health Check Endpoint

```javascript
// src/health.js
export function setupHealthChecks(app, prisma, redis) {
  // Liveness probe — is the app running?
  app.get('/health/live', (req, res) => {
    res.json({ status: 'ok', timestamp: new Date().toISOString() });
  });
  
  // Readiness probe — can it serve requests?
  app.get('/health/ready', async (req, res) => {
    const checks = {
      database: false,
      redis: false,
    };
    
    try {
      await prisma.$queryRaw`SELECT 1`;
      checks.database = true;
    } catch (err) {
      console.error('DB health check failed:', err);
    }
    
    try {
      await redis.ping();
      checks.redis = true;
    } catch (err) {
      console.error('Redis health check failed:', err);
    }
    
    const allHealthy = Object.values(checks).every(Boolean);
    
    res.status(allHealthy ? 200 : 503).json({
      status: allHealthy ? 'ready' : 'not_ready',
      checks,
      timestamp: new Date().toISOString(),
    });
  });
  
  // Simple health endpoint (backward compat)
  app.get('/health', (req, res) => res.json({ status: 'ok' }));
}
```

---

## 📌 Step 915: Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graphql-api
  namespace: production
  labels:
    app: graphql-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: graphql-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero-downtime
  template:
    metadata:
      labels:
        app: graphql-api
    spec:
      containers:
        - name: graphql-api
          image: myapp/graphql-api:v1.2.3
          ports:
            - containerPort: 4000
          
          # Resources
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          
          # Environment
          env:
            - name: NODE_ENV
              value: "production"
            - name: PORT
              value: "4000"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: graphql-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: graphql-secrets
                  key: redis-url
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: graphql-secrets
                  key: jwt-secret
          
          # Health checks
          livenessProbe:
            httpGet:
              path: /health/live
              port: 4000
            initialDelaySeconds: 15
            periodSeconds: 20
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 4000
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
      
      terminationGracePeriodSeconds: 30
```

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: graphql-api-service
  namespace: production
spec:
  selector:
    app: graphql-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 4000
  type: ClusterIP

---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: graphql-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: graphql-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /graphql
            pathType: Prefix
            backend:
              service:
                name: graphql-api-service
                port:
                  number: 80
```

```yaml
# k8s/secrets.yaml (ใช้ kubectl apply หลัง encode)
apiVersion: v1
kind: Secret
metadata:
  name: graphql-secrets
  namespace: production
type: Opaque
data:
  # base64 encoded values
  # echo -n "value" | base64
  database-url: cG9zdGdyZXNxbDovLy4uLg==
  redis-url: cmVkaXM6Ly8uLi4=
  jwt-secret: c3VwZXItc2VjcmV0LWtleQ==
```

```yaml
# k8s/hpa.yaml — Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: graphql-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: graphql-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## 📌 Step 916: สรุป Part 026

### เนื้อหาที่เรียนรู้

✅ Dockerfile multi-stage build  
✅ docker-compose สำหรับ development  
✅ docker-compose.prod.yml  
✅ Health check endpoints  
✅ Kubernetes Deployment  
✅ Service + Ingress  
✅ Secrets management  
✅ HPA (Horizontal Pod Autoscaler)  

### Homework

1. Containerize GraphQL server ด้วย Docker
2. สร้าง docker-compose สำหรับ dev environment
3. Deploy ใน K8s local (minikube)

### ในส่วนถัดไป

➡️ **[Part 027](./part-027.md)** — CI/CD Pipeline

---

*Part 026 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
