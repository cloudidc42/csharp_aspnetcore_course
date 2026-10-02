# Part 084: Kubernetes เบื้องต้น

## เนื้อหาใน Part นี้
- K8s concepts: Pod, Service, Deployment
- kubectl commands
- ConfigMaps และ Secrets
- Ingress
- Helm charts เบื้องต้น
- โปรแกรมตัวอย่าง: Deploy to K8s

---

## 1. Kubernetes คืออะไร

Kubernetes (K8s) เป็น container orchestration platform ที่จัดการ deployment, scaling, และ operation ของ containerized applications

### ทำไมต้องใช้ K8s?

```
Docker Compose: สำหรับ single host
Kubernetes: สำหรับ production cluster ที่มีหลาย machines

K8s จัดการ:
- Automatic scaling (เพิ่ม/ลด containers ตาม load)
- Self-healing (restart container ที่ crash อัตโนมัติ)
- Rolling updates (update โดยไม่ downtime)
- Load balancing
- Service discovery
- Secret management
- Resource allocation
```

### K8s Architecture

```
                    ┌─────────────────────────────┐
                    │       Control Plane          │
                    │  ┌──────────┐  ┌──────────┐  │
                    │  │  API     │  │  etcd    │  │
                    │  │  Server  │  │ (state)  │  │
                    │  └──────────┘  └──────────┘  │
                    │  ┌──────────┐  ┌──────────┐  │
                    │  │Scheduler │  │Controller│  │
                    │  │          │  │ Manager  │  │
                    │  └──────────┘  └──────────┘  │
                    └──────────────┬──────────────┘
                                   │
           ┌───────────────────────┼───────────────────────┐
           │                       │                       │
  ┌────────┴────────┐    ┌─────────┴───────┐    ┌─────────┴───────┐
  │    Worker Node 1 │    │  Worker Node 2  │    │  Worker Node 3  │
  │  ┌────┐ ┌────┐  │    │  ┌────┐ ┌────┐  │    │  ┌────┐ ┌────┐  │
  │  │Pod │ │Pod │  │    │  │Pod │ │Pod │  │    │  │Pod │ │Pod │  │
  │  └────┘ └────┘  │    │  └────┘ └────┘  │    │  └────┘ └────┘  │
  │  kubelet         │    │  kubelet         │    │  kubelet         │
  └─────────────────┘    └─────────────────┘    └─────────────────┘
```

---

## 2. Core Concepts

### Pod

Pod เป็นหน่วยที่เล็กที่สุดใน K8s - container หนึ่งหรือหลายตัวที่ share network และ storage

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    environment: production
spec:
  containers:
  - name: api
    image: myregistry.azurecr.io/myapp:v1.0
    ports:
    - containerPort: 8080
    env:
    - name: ASPNETCORE_ENVIRONMENT
      value: "Production"
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "500m"
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
```

### Deployment

Deployment จัดการ Pods ให้อยู่ในสถานะที่ต้องการเสมอ

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  namespace: production
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # สร้าง new pods ก่อนลบ old
      maxUnavailable: 0  # ห้าม downtime
  template:
    metadata:
      labels:
        app: myapp
        version: v1.0
    spec:
      containers:
      - name: api
        image: myregistry.azurecr.io/myapp:v1.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__DefaultConnection
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: db-connection-string
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
      imagePullSecrets:
      - name: registry-secret
```

### Service

Service ทำหน้าที่ expose Pods และ load balance traffic

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: production
spec:
  selector:
    app: myapp          # Match pods ที่มี label นี้
  type: ClusterIP       # ClusterIP, NodePort, LoadBalancer
  ports:
  - name: http
    port: 80            # Service port
    targetPort: 8080    # Pod port
    protocol: TCP
```

```yaml
# Service Types:

# ClusterIP - accessible เฉพาะใน cluster (default)
spec:
  type: ClusterIP

# NodePort - accessible จากนอก cluster ผ่าน node IP:port
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # Range 30000-32767

# LoadBalancer - ขอ external load balancer จาก cloud provider
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
```

---

## 3. kubectl Commands

```bash
# Namespace
kubectl create namespace production
kubectl get namespaces
kubectl config set-context --current --namespace=production

# Apply manifests
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f .  # Apply ทุกไฟล์ใน directory

# Get resources
kubectl get pods
kubectl get pods -n production
kubectl get deployments
kubectl get services
kubectl get all

# Describe (เหมาะสำหรับ troubleshooting)
kubectl describe pod myapp-pod-xxx
kubectl describe deployment myapp-deployment
kubectl describe service myapp-service

# Logs
kubectl logs myapp-pod-xxx
kubectl logs -f myapp-pod-xxx          # Follow
kubectl logs myapp-pod-xxx -c api      # Specific container
kubectl logs --previous myapp-pod-xxx  # Previous container logs

# Execute command ใน container
kubectl exec -it myapp-pod-xxx -- bash
kubectl exec myapp-pod-xxx -- dotnet --version

# Port forwarding (local development)
kubectl port-forward service/myapp-service 5000:80
kubectl port-forward pod/myapp-pod-xxx 5000:8080

# Scale
kubectl scale deployment myapp-deployment --replicas=5

# Rolling update
kubectl set image deployment/myapp-deployment api=myregistry.azurecr.io/myapp:v2.0
kubectl rollout status deployment/myapp-deployment
kubectl rollout history deployment/myapp-deployment
kubectl rollout undo deployment/myapp-deployment           # Rollback
kubectl rollout undo deployment/myapp-deployment --to-revision=2

# Delete
kubectl delete pod myapp-pod-xxx
kubectl delete deployment myapp-deployment
kubectl delete -f deployment.yaml

# Resource usage
kubectl top nodes
kubectl top pods
```

---

## 4. ConfigMaps และ Secrets

### ConfigMap

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: production
data:
  # Key-value pairs
  ASPNETCORE_ENVIRONMENT: "Production"
  LOG_LEVEL: "Information"
  CACHE_TTL_MINUTES: "60"
  
  # Multi-line value (appsettings.json)
  appsettings.json: |
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft.AspNetCore": "Warning"
        }
      },
      "AllowedHosts": "*",
      "FeatureFlags": {
        "EnableNewCheckout": true,
        "EnableAnalytics": false
      }
    }
```

```yaml
# ใช้ ConfigMap ใน Deployment
spec:
  containers:
  - name: api
    # Method 1: envFrom (import ทุก key เป็น env var)
    envFrom:
    - configMapRef:
        name: myapp-config

    # Method 2: env (select specific key)
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: myapp-config
          key: LOG_LEVEL

    # Method 3: volumeMount (mount as file)
    volumeMounts:
    - name: config-volume
      mountPath: /app/config
  
  volumes:
  - name: config-volume
    configMap:
      name: myapp-config
      items:
      - key: appsettings.json
        path: appsettings.json
```

### Secrets

```yaml
# secret.yaml (Base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: production
type: Opaque
data:
  # base64 encode: echo -n "value" | base64
  db-password: cGFzc3dvcmQxMjM=    # password123
  jwt-secret: bXktc2VjcmV0LWtleQ== # my-secret-key
  connection-string: SG9zdD1kYjsxMjM=
```

```bash
# สร้าง Secret จาก command line (ดีกว่า - ไม่ commit ไปใน git)
kubectl create secret generic myapp-secrets \
  --from-literal=db-password='MyStrongPass123!' \
  --from-literal=jwt-secret='my-super-secret-jwt-key-here' \
  --namespace production

# สร้างจาก file
kubectl create secret generic myapp-secrets \
  --from-file=connection-string=./connection-string.txt

# Docker registry secret
kubectl create secret docker-registry registry-secret \
  --docker-server=myregistry.azurecr.io \
  --docker-username=myregistry \
  --docker-password=mypassword \
  --namespace production
```

```yaml
# ใช้ Secrets
spec:
  containers:
  - name: api
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: myapp-secrets
          key: db-password
    
    # Mount เป็น file
    volumeMounts:
    - name: secrets
      mountPath: /app/secrets
      readOnly: true
  
  volumes:
  - name: secrets
    secret:
      secretName: myapp-secrets
```

---

## 5. Ingress

Ingress จัดการ HTTP routing จากนอก cluster เข้า services

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.myapp.com
    secretName: myapp-tls
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
      - path: /auth
        pathType: Prefix
        backend:
          service:
            name: auth-service
            port:
              number: 80
```

### NGINX Ingress Controller

```bash
# ติดตั้ง NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/cloud/deploy.yaml

# ตรวจสอบ
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

---

## 6. Helm Charts เบื้องต้น

Helm เป็น package manager สำหรับ Kubernetes

### โครงสร้าง Helm Chart

```
myapp-chart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default values
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml        # Horizontal Pod Autoscaler
│   └── _helpers.tpl    # Template helpers
└── charts/             # Dependencies
```

### Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
description: My ASP.NET Core Application
type: application
version: 0.1.0
appVersion: "1.0.0"
dependencies:
- name: postgresql
  version: "12.5.6"
  repository: "https://charts.bitnami.com/bitnami"
  condition: postgresql.enabled
```

### values.yaml

```yaml
# values.yaml
replicaCount: 3

image:
  repository: myregistry.azurecr.io/myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

imagePullSecrets:
- name: registry-secret

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  host: api.myapp.com
  tls: true

resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

config:
  environment: "Production"
  logLevel: "Information"

postgresql:
  enabled: true
  auth:
    database: myapp
    username: myapp
    # password ตั้งผ่าน --set หรือ secrets
```

### templates/deployment.yaml

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: {{ .Values.config.environment }}
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "myapp.fullname" . }}-secrets
              key: db-password
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
        livenessProbe:
          httpGet:
            path: /health/live
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Helm Commands

```bash
# Install chart
helm install myapp ./myapp-chart

# Install with custom values
helm install myapp ./myapp-chart \
  --values ./values-prod.yaml \
  --set image.tag=v2.0 \
  --set postgresql.auth.password=MyPass123 \
  --namespace production --create-namespace

# Upgrade
helm upgrade myapp ./myapp-chart \
  --values ./values-prod.yaml \
  --set image.tag=v2.1

# Rollback
helm rollback myapp 1  # Rollback to revision 1

# List releases
helm list
helm list -n production

# Get values
helm get values myapp
helm get manifest myapp

# Uninstall
helm uninstall myapp

# Render templates (dry run)
helm template myapp ./myapp-chart --values values-prod.yaml

# Lint chart
helm lint ./myapp-chart
```

---

## 7. โปรแกรมตัวอย่าง: Deploy ASP.NET Core API to K8s

### Project Health Checks

```csharp
// Program.cs - Health checks ที่ K8s ต้องการ
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: new[] { "live" })
    .AddNpgsql(
        builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "database",
        tags: new[] { "ready" })
    .AddRedis(
        builder.Configuration["Redis:ConnectionString"] ?? "localhost:6379",
        name: "redis",
        tags: new[] { "ready" });

// Liveness probe - ตรวจว่า app ยัง live อยู่
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

// Readiness probe - ตรวจว่า app พร้อมรับ traffic
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

### K8s Manifests

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
```

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: production
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  ASPNETCORE_URLS: "http://+:8080"
  Logging__LogLevel__Default: "Information"
  Logging__LogLevel__Microsoft_AspNetCore: "Warning"
```

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: myapp
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: api
        image: myregistry.azurecr.io/myapp:1.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
        envFrom:
        - configMapRef:
            name: myapp-config
        env:
        - name: ConnectionStrings__DefaultConnection
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: db-connection-string
        - name: JWT__SecretKey
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: jwt-secret
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: http
          initialDelaySeconds: 30
          periodSeconds: 15
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]
      imagePullSecrets:
      - name: registry-secret
      terminationGracePeriodSeconds: 60
```

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
spec:
  selector:
    app: myapp
  type: ClusterIP
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

```yaml
# k8s/hpa.yaml - Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
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

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.myapp.com
    secretName: myapp-tls
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

### Deploy Script

```bash
#!/bin/bash
# deploy.sh

set -e

NAMESPACE=production
APP_NAME=myapp
IMAGE_TAG=${1:-latest}
REGISTRY=myregistry.azurecr.io

echo "Deploying ${APP_NAME}:${IMAGE_TAG} to ${NAMESPACE}..."

# Create namespace if not exists
kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

# Apply secrets (from environment)
kubectl create secret generic myapp-secrets \
  --from-literal=db-connection-string="${DB_CONNECTION_STRING}" \
  --from-literal=jwt-secret="${JWT_SECRET}" \
  --namespace ${NAMESPACE} \
  --dry-run=client -o yaml | kubectl apply -f -

# Apply manifests
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa.yaml
kubectl apply -f k8s/ingress.yaml

# Update image
kubectl set image deployment/${APP_NAME} \
  api=${REGISTRY}/${APP_NAME}:${IMAGE_TAG} \
  --namespace ${NAMESPACE}

# Wait for rollout
kubectl rollout status deployment/${APP_NAME} \
  --namespace ${NAMESPACE} \
  --timeout=300s

echo "Deployment completed successfully!"
kubectl get pods -n ${NAMESPACE} -l app=${APP_NAME}
```

---

## Exercises / Project Tasks

### Exercise 1: Local K8s
ติดตั้งและใช้งาน:
```bash
# ติดตั้ง minikube หรือ kind
# Deploy simple API
# ทดสอบ port-forward
# ดู logs
```

### Exercise 2: ConfigMap & Secret
ปรับ application ให้อ่าน config จาก:
- ConfigMap สำหรับ non-sensitive config
- Secret สำหรับ passwords/tokens

### Exercise 3: HPA (Autoscaling)
ทดสอบ autoscaling:
- สร้าง HPA
- Load test ด้วย k6 หรือ hey
- ดู scaling event

### Exercise 4: Rolling Update
ทดสอบ rolling update:
- Deploy version 1
- Update ไป version 2 โดยไม่ downtime
- Rollback ถ้า health check fail

---

## สรุป

- **Pod** = หน่วยเล็กสุด ประกอบด้วยหนึ่งหรือหลาย containers
- **Deployment** = จัดการ Pods ให้ตรงตามที่ต้องการ รองรับ rolling update
- **Service** = Load balance และ service discovery ใน cluster
- **ConfigMap** = เก็บ non-sensitive configuration
- **Secret** = เก็บ sensitive data แบบ encrypted
- **Ingress** = HTTP routing จากนอก cluster เข้า services
- **HPA** = Auto-scale pods ตาม CPU/memory
- **Helm** = Package manager ช่วยจัดการ K8s manifests

---

## Part ถัดไป

**Part 085: CI/CD กับ GitHub Actions** - สร้าง automated pipeline สำหรับ build, test, และ deploy

---

*Part 084/100 | Phase 6/7: ระดับสูง | หลักสูตร C# และ ASP.NET Core*
