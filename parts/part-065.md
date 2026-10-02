# Part 065 — Kubernetes & Container Orchestration ☸️

> **ระดับ:** Expert | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1501–1540

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Kubernetes deployment สำหรับ GraphQL API
- HorizontalPodAutoscaler (HPA)
- ConfigMaps & Secrets
- Service mesh basics (Istio)
- Ingress configuration
- Resource limits & requests
- Pod disruption budgets
- Helm charts

---

## 📌 Step 1501: Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graphql-api
  namespace: production
  labels:
    app: graphql-api
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: graphql-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # เพิ่ม 1 pod ระหว่าง update
      maxUnavailable: 0   # ไม่ลด pod ระหว่าง update (zero-downtime)
  template:
    metadata:
      labels:
        app: graphql-api
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "4000"
        prometheus.io/path: "/metrics"
    spec:
      # Spread pods across nodes
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: graphql-api
      
      # Terminate old pods gracefully
      terminationGracePeriodSeconds: 60
      
      containers:
        - name: api
          image: myapp/graphql-api:1.2.3
          imagePullPolicy: Always
          ports:
            - containerPort: 4000
              name: http
          
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
          
          env:
            - name: NODE_ENV
              value: production
            - name: PORT
              value: "4000"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: redis-secrets
                  key: url
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: jwt-secret
          
          envFrom:
            - configMapRef:
                name: app-config
          
          livenessProbe:
            httpGet:
              path: /health
              port: 4000
            initialDelaySeconds: 30
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /ready
              port: 4000
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          
          lifecycle:
            preStop:
              exec:
                # Allow time for load balancer to stop routing
                command: ["/bin/sh", "-c", "sleep 5"]
      
      # Graceful shutdown
      preStopHook:
        exec:
          command: ["/bin/sh", "-c", "sleep 10 && kill -SIGTERM 1"]
```

---

## 📌 Step 1502: HorizontalPodAutoscaler

```yaml
# k8s/hpa.yaml
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
  
  minReplicas: 3
  maxReplicas: 20
  
  metrics:
    # Scale บน CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60   # Scale เมื่อ avg CPU > 60%
    
    # Scale บน Memory
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
    
    # Scale บน custom metric (requests per second)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"    # 100 req/s per pod
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # รอ 5 นาทีก่อน scale down
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60     # ลดได้ 25% ต่อนาที
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30     # เพิ่มได้ 100% ทุก 30 วินาที
        - type: Pods
          value: 4
          periodSeconds: 30     # หรือเพิ่มได้ 4 pods ทุก 30 วินาที
      selectPolicy: Max         # ใช้ policy ที่เพิ่มมากกว่า
```

---

## 📌 Step 1503: Service & Ingress

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: graphql-api
  namespace: production
  annotations:
    # AWS NLB: stick sessions
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  selector:
    app: graphql-api
  ports:
    - name: http
      port: 80
      targetPort: 4000
      protocol: TCP
  type: ClusterIP
  # sessionAffinity: ClientIP  # Sticky sessions สำหรับ WebSocket
---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: graphql-api
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"    # สำหรับ subscriptions
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/upstream-hash-by: "$remote_addr" # Sticky sessions
    nginx.ingress.kubernetes.io/websocket-services: "graphql-api"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.myapp.com
      secretName: api-tls
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /graphql
            pathType: Prefix
            backend:
              service:
                name: graphql-api
                port:
                  number: 80
```

---

## 📌 Step 1504: Pod Disruption Budget

```yaml
# k8s/pdb.yaml
# ป้องกันการ disrupt หลาย pods พร้อมกัน
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: graphql-api-pdb
  namespace: production
spec:
  selector:
    matchLabels:
      app: graphql-api
  minAvailable: 2   # ต้องมี pod ที่ running อย่างน้อย 2 ตลอดเวลา
  # maxUnavailable: 1  # หรือระบุ max unavailable
```

---

## 📌 Step 1505: Helm Chart Structure

```
graphql-api/          ← Helm chart
├── Chart.yaml
├── values.yaml
├── values-prod.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   └── _helpers.tpl
```

```yaml
# values.yaml
replicaCount: 3
image:
  repository: myapp/graphql-api
  tag: latest
  pullPolicy: Always

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  hostname: api.myapp.com

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 60

env:
  NODE_ENV: production
  
secrets:
  DATABASE_URL: ""
  REDIS_URL: ""
  JWT_SECRET: ""
```

```bash
# Deploy
helm upgrade --install graphql-api ./charts/graphql-api \
  -f values-prod.yaml \
  --set image.tag=1.2.3 \
  --set secrets.DATABASE_URL="${DATABASE_URL}" \
  --namespace production \
  --create-namespace \
  --wait \
  --timeout 5m

# Rollback
helm rollback graphql-api 1 --namespace production
```

---

## 📌 Step 1506: สรุป Part 065

### เนื้อหาที่เรียนรู้

✅ Kubernetes Deployment  
✅ HPA (auto-scaling)  
✅ Service & Ingress (WebSocket support)  
✅ Pod Disruption Budget  
✅ Helm charts  

### Production Kubernetes Checklist

```
Deployment:
□ Resource requests & limits set
□ Liveness & readiness probes
□ Topology spread constraints
□ terminationGracePeriodSeconds ≥ 60
□ preStop hook (sleep 5-10s)

Reliability:
□ minReplicas >= 2 (HA)
□ PodDisruptionBudget
□ topologySpreadConstraints
□ HPA configured

Security:
□ Non-root user
□ readOnlyRootFilesystem
□ Secrets ไม่ใช่ environment variables plain text
□ Network policies
```

### ในส่วนถัดไป

➡️ **[Part 066](./part-066.md)** — Observability: OpenTelemetry & Distributed Tracing

---

*Part 065 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
