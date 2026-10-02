# Part 027 — CI/CD Pipeline for GraphQL 🚀

> **ระดับ:** Advanced | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 961–1000

---

## 🎯 สิ่งที่จะได้เรียนรู้

- GitHub Actions CI/CD
- Schema breaking change detection
- Automated testing
- Docker build + push
- Deploy to staging/production
- Schema registry
- Rollback strategies
- Monitoring deployment

---

## 📌 Step 961: GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # Job 1: Test
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run type check
        run: npm run typecheck
      
      - name: Run linter
        run: npm run lint
      
      - name: Run database migrations
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/test_db
        run: npx prisma migrate deploy
      
      - name: Run tests
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
          JWT_SECRET: test-secret-key
        run: npm test -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
  
  # Job 2: Schema Validation
  schema-check:
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'pull_request'
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install deps
        run: npm ci
      
      - name: Check breaking changes
        run: |
          # Download baseline schema
          curl -s $SCHEMA_REGISTRY_URL/schema/latest > baseline.graphql
          
          # Compare
          npx @graphql-inspector/cli diff baseline.graphql src/schema.graphql
        env:
          SCHEMA_REGISTRY_URL: ${{ secrets.SCHEMA_REGISTRY_URL }}
      
      - name: Validate schema
        run: npx graphql-schema-linter src/**/*.graphql
  
  # Job 3: Build & Push Docker
  build:
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push'
    
    permissions:
      contents: read
      packages: write
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  # Job 4: Deploy to Staging
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Staging
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /app
            export IMAGE_TAG=${{ needs.build.outputs.image-tag }}
            docker-compose pull
            docker-compose up -d --no-deps app
            docker-compose exec app npx prisma migrate deploy
  
  # Job 5: Deploy to Production
  deploy-production:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.0'
      
      - name: Configure kubectl
        run: |
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > kubeconfig.yaml
          echo "KUBECONFIG=kubeconfig.yaml" >> $GITHUB_ENV
      
      - name: Deploy to K8s
        run: |
          IMAGE=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          kubectl set image deployment/graphql-api graphql-api=$IMAGE -n production
          kubectl rollout status deployment/graphql-api -n production --timeout=5m
      
      - name: Run migrations
        run: |
          kubectl exec deploy/graphql-api -n production -- npx prisma migrate deploy
      
      - name: Verify deployment
        run: |
          kubectl get pods -n production -l app=graphql-api
          sleep 30
          HEALTH=$(kubectl exec deploy/graphql-api -n production -- curl -s http://localhost:4000/health)
          echo "Health: $HEALTH"
```

---

## 📌 Step 962: Schema Breaking Change Detection

```javascript
// scripts/check-schema-breaking.js
import { diff, BreakingChangeType } from '@graphql-inspector/core';
import { loadSchema } from '@graphql-tools/load';

async function checkBreakingChanges() {
  const oldSchema = await loadSchema('./schema-baseline.graphql');
  const newSchema = await loadSchema('./src/schema.graphql');
  
  const changes = diff(oldSchema, newSchema);
  
  const breaking = changes.filter(c => c.criticality.level === 'BREAKING');
  const dangerous = changes.filter(c => c.criticality.level === 'DANGEROUS');
  const safe = changes.filter(c => c.criticality.level === 'NON_BREAKING');
  
  console.log(`✅ Safe changes: ${safe.length}`);
  console.log(`⚠️ Dangerous changes: ${dangerous.length}`);
  console.log(`❌ Breaking changes: ${breaking.length}`);
  
  if (breaking.length > 0) {
    console.error('\nBreaking changes detected:');
    breaking.forEach(change => {
      console.error(`  - ${change.message}`);
    });
    process.exit(1);
  }
  
  if (dangerous.length > 0) {
    console.warn('\nDangerous changes:');
    dangerous.forEach(change => {
      console.warn(`  - ${change.message}`);
    });
  }
}

checkBreakingChanges().catch(err => {
  console.error(err);
  process.exit(1);
});
```

---

## 📌 Step 963: Rollback Strategy

```bash
#!/bin/bash
# scripts/rollback.sh

NAMESPACE=production
DEPLOYMENT=graphql-api
REVISION=${1:-0}  # 0 = previous revision

if [ "$REVISION" -eq 0 ]; then
  echo "Rolling back to previous revision..."
  kubectl rollout undo deployment/$DEPLOYMENT -n $NAMESPACE
else
  echo "Rolling back to revision $REVISION..."
  kubectl rollout undo deployment/$DEPLOYMENT -n $NAMESPACE --to-revision=$REVISION
fi

# Wait for rollback
kubectl rollout status deployment/$DEPLOYMENT -n $NAMESPACE --timeout=5m

echo "Rollback complete!"
kubectl get pods -n $NAMESPACE -l app=$DEPLOYMENT
```

---

## 📌 Step 964: สรุป Part 027

### เนื้อหาที่เรียนรู้

✅ GitHub Actions CI/CD pipeline  
✅ Test automation  
✅ Schema breaking change detection  
✅ Docker build + push to registry  
✅ Deploy to staging/production  
✅ Kubernetes rolling deployment  
✅ Rollback strategy  

### Homework

1. สร้าง CI/CD pipeline ด้วย GitHub Actions
2. เพิ่ม schema breaking change check ใน PR
3. Configure automatic rollback ถ้า health check fail

### ในส่วนถัดไป

➡️ **[Part 028](./part-028.md)** — Monitoring & Observability

---

*Part 027 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
