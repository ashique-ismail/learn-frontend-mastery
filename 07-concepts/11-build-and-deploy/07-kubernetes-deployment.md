# Kubernetes Deployment

## Overview

Kubernetes (K8s) is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications. It provides declarative configuration, self-healing, load balancing, and robust infrastructure management for modern cloud-native applications.

## Table of Contents
- [Core Concepts](#core-concepts)
- [Kubernetes Architecture](#kubernetes-architecture)
- [Deployments](#deployments)
- [Services](#services)
- [Ingress](#ingress)
- [ConfigMaps and Secrets](#configmaps-and-secrets)
- [Scaling Strategies](#scaling-strategies)
- [Rolling Updates](#rolling-updates)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Core Concepts

### Kubernetes Object Model

```yaml
# Basic structure of a Kubernetes object
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    version: v1.0.0
spec:
  # Desired state specification
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:1.0.0
        ports:
        - containerPort: 3000
```

### Pods

Pods are the smallest deployable units in Kubernetes, containing one or more containers.

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: react-app-pod
  labels:
    app: react-app
    tier: frontend
spec:
  containers:
  - name: react-app
    image: my-react-app:1.0.0
    ports:
    - containerPort: 80
    env:
    - name: API_URL
      value: "https://api.example.com"
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
        cpu: "500m"
    livenessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
```

## Kubernetes Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Control Plane                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ API Server   │  │  Scheduler   │  │  Controller  │     │
│  │              │  │              │  │   Manager    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│         │                  │                  │             │
│         └──────────────────┴──────────────────┘             │
│                           │                                 │
│                    ┌──────▼──────┐                         │
│                    │    etcd     │                         │
│                    │  (Storage)  │                         │
│                    └─────────────┘                         │
└─────────────────────────────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
┌───────▼────────┐  ┌──────▼──────┐  ┌───────▼────────┐
│   Worker Node  │  │ Worker Node │  │  Worker Node   │
│  ┌──────────┐  │  │ ┌──────────┐│  │ ┌──────────┐   │
│  │ Kubelet  │  │  │ │ Kubelet  ││  │ │ Kubelet  │   │
│  └──────────┘  │  │ └──────────┘│  │ └──────────┘   │
│  ┌──────────┐  │  │ ┌──────────┐│  │ ┌──────────┐   │
│  │Kube-Proxy│  │  │ │Kube-Proxy││  │ │Kube-Proxy│   │
│  └──────────┘  │  │ └──────────┘│  │ └──────────┘   │
│  ┌──────────┐  │  │ ┌──────────┐│  │ ┌──────────┐   │
│  │  Pods    │  │  │ │  Pods    ││  │ │  Pods    │   │
│  └──────────┘  │  │ └──────────┘│  │ └──────────┘   │
└────────────────┘  └─────────────┘  └────────────────┘
```

## Deployments

Deployments provide declarative updates for Pods and ReplicaSets.

### Basic Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: react-frontend
  namespace: production
  labels:
    app: react-frontend
    tier: frontend
spec:
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: react-frontend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: react-frontend
        version: v1.2.0
    spec:
      containers:
      - name: react-app
        image: myregistry.azurecr.io/react-frontend:1.2.0
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
        env:
        - name: NODE_ENV
          value: "production"
        - name: API_BASE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: api.url
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: api-key
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      imagePullSecrets:
      - name: registry-credentials
```

### Multi-Container Pod Deployment

```yaml
# multi-container-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: angular-app-with-sidecar
spec:
  replicas: 2
  selector:
    matchLabels:
      app: angular-app
  template:
    metadata:
      labels:
        app: angular-app
    spec:
      containers:
      # Main application container
      - name: angular-app
        image: angular-app:2.0.0
        ports:
        - containerPort: 4200
        volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
      
      # Logging sidecar container
      - name: log-shipper
        image: fluent/fluent-bit:1.9
        volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
          readOnly: true
        - name: fluentbit-config
          mountPath: /fluent-bit/etc/
      
      # Metrics exporter sidecar
      - name: prometheus-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
      
      volumes:
      - name: shared-logs
        emptyDir: {}
      - name: fluentbit-config
        configMap:
          name: fluentbit-config
```

### Deployment with Init Containers

```yaml
# deployment-with-init.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-with-init
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      initContainers:
      # Wait for database to be ready
      - name: wait-for-db
        image: busybox:1.35
        command: ['sh', '-c', 'until nc -z postgres-service 5432; do echo waiting for db; sleep 2; done;']
      
      # Run database migrations
      - name: run-migrations
        image: my-app:1.0.0
        command: ['npm', 'run', 'migrate']
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
      
      containers:
      - name: web-app
        image: my-app:1.0.0
        ports:
        - containerPort: 3000
```

## Services

Services expose Pods to network traffic within or outside the cluster.

### ClusterIP Service (Internal)

```yaml
# service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: production
  labels:
    app: backend
spec:
  type: ClusterIP  # Default type
  selector:
    app: backend
    tier: api
  ports:
  - name: http
    protocol: TCP
    port: 8080        # Service port
    targetPort: 8080  # Container port
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

### NodePort Service

```yaml
# service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
spec:
  type: NodePort
  selector:
    app: react-frontend
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080  # Exposed on all nodes (30000-32767)
```

### LoadBalancer Service

```yaml
# service-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "false"
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: react-frontend
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
  - name: https
    protocol: TCP
    port: 443
    targetPort: 443
  loadBalancerIP: 203.0.113.5  # Optional static IP
```

### Headless Service

```yaml
# service-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: stateful-service
spec:
  clusterIP: None  # Headless service
  selector:
    app: stateful-app
  ports:
  - name: http
    port: 8080
    targetPort: 8080
```

## Ingress

Ingress manages external access to services, providing HTTP/HTTPS routing.

### Basic Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - myapp.example.com
    - www.myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 8080
```

### Advanced Ingress with Multiple Hosts

```yaml
# ingress-advanced.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    - api.example.com
    - admin.example.com
    secretName: wildcard-tls
  rules:
  # Frontend application
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: react-frontend
            port:
              number: 80
  
  # API backend
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 8080
  
  # Admin panel
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 3000
```

## ConfigMaps and Secrets

### ConfigMap

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  # Simple key-value pairs
  api.url: "https://api.example.com"
  environment: "production"
  log.level: "info"
  
  # Multi-line configuration
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;
      
      location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;
      }
      
      location /api {
        proxy_pass http://backend-service:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
      }
    }
  
  # JSON configuration
  app-config.json: |
    {
      "features": {
        "enableAnalytics": true,
        "enableDebugMode": false,
        "maxUploadSize": "10MB"
      },
      "endpoints": {
        "api": "https://api.example.com",
        "cdn": "https://cdn.example.com"
      }
    }
```

### Using ConfigMap in Deployment

```yaml
# deployment-with-configmap.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-config
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
        
        # Method 1: Environment variables from ConfigMap
        env:
        - name: API_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: api.url
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: log.level
        
        # Method 2: All ConfigMap keys as environment variables
        envFrom:
        - configMapRef:
            name: app-config
        
        # Method 3: Mount ConfigMap as volume
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
      
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      - name: nginx-config
        configMap:
          name: app-config
          items:
          - key: nginx.conf
            path: nginx.conf
```

### Secrets

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
data:
  # Base64 encoded values
  api-key: YXBpLWtleS12YWx1ZQ==
  database-url: cG9zdGdyZXNxbDovL3VzZXI6cGFzc0BkYjozMjE2L2RibmFtZQ==
stringData:
  # Plain text (will be base64 encoded automatically)
  jwt-secret: "my-super-secret-jwt-key"
  oauth-client-id: "oauth-client-id-value"
---
# TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi...  # Base64 encoded certificate
  tls.key: LS0tLS1CRUdJTi...  # Base64 encoded private key
---
# Docker registry secret
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6eyJteXJlZ2lzdHJ5LmF6dXJlY3IuaW8iOnsidXNlcm5hbWUiOiJ1c2VyIiwicGFzc3dvcmQiOiJwYXNzIiwiZW1haWwiOiJlbWFpbEBleGFtcGxlLmNvbSIsImF1dGgiOiJkWE5sY2pwd1lYTnoifX19
```

### Using Secrets in Deployment

```yaml
# deployment-with-secrets.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        
        # Environment variables from Secret
        env:
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: api-key
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
        
        # Mount secrets as files
        volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
        - name: tls-certs
          mountPath: /etc/tls
          readOnly: true
      
      volumes:
      - name: secret-volume
        secret:
          secretName: app-secrets
          items:
          - key: api-key
            path: api-key.txt
      - name: tls-certs
        secret:
          secretName: tls-secret
      
      imagePullSecrets:
      - name: registry-credentials
```

## Scaling Strategies

### Horizontal Pod Autoscaler (HPA)

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: react-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: react-frontend
  minReplicas: 3
  maxReplicas: 20
  metrics:
  # CPU-based scaling
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # Memory-based scaling
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # Custom metric scaling
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 4
        periodSeconds: 30
      selectPolicy: Max
```

### Vertical Pod Autoscaler (VPA)

```yaml
# vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: react-frontend
  updatePolicy:
    updateMode: "Auto"  # Auto, Recreate, Initial, or Off
  resourcePolicy:
    containerPolicies:
    - containerName: react-app
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 1
        memory: 1Gi
      controlledResources:
      - cpu
      - memory
```

### Manual Scaling

```bash
# Scale deployment imperatively
kubectl scale deployment react-frontend --replicas=5

# Scale with multiple deployments
kubectl scale deployment react-frontend backend-api --replicas=10

# Auto-scale based on CPU
kubectl autoscale deployment react-frontend --min=3 --max=10 --cpu-percent=80
```

## Rolling Updates

### Rolling Update Strategy

```yaml
# deployment-rolling-update.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-update-app
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # Max pods above desired count
      maxUnavailable: 1  # Max pods unavailable during update
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: v2.0.0
    spec:
      containers:
      - name: myapp
        image: myapp:2.0.0
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Blue-Green Deployment

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
  labels:
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
---
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
  labels:
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:2.0.0
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue  # Switch to 'green' to cutover
  ports:
  - port: 80
    targetPort: 8080
```

### Canary Deployment

```yaml
# stable-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
spec:
  replicas: 9  # 90% of traffic
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
        version: v1.0.0
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
---
# canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1  # 10% of traffic
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
        version: v2.0.0
    spec:
      containers:
      - name: myapp
        image: myapp:2.0.0
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp  # Routes to both stable and canary
  ports:
  - port: 80
    targetPort: 8080
```

### Update Commands

```bash
# Update deployment image
kubectl set image deployment/react-frontend react-app=myapp:2.0.0

# Check rollout status
kubectl rollout status deployment/react-frontend

# View rollout history
kubectl rollout history deployment/react-frontend

# Rollback to previous version
kubectl rollout undo deployment/react-frontend

# Rollback to specific revision
kubectl rollout undo deployment/react-frontend --to-revision=3

# Pause rollout
kubectl rollout pause deployment/react-frontend

# Resume rollout
kubectl rollout resume deployment/react-frontend

# Restart deployment (recreate all pods)
kubectl rollout restart deployment/react-frontend
```

## Common Mistakes

### 1. Missing Resource Limits

```yaml
# Bad: No resource limits
spec:
  containers:
  - name: app
    image: app:1.0.0

# Good: Define resource requests and limits
spec:
  containers:
  - name: app
    image: app:1.0.0
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
```

### 2. Missing Health Checks

```yaml
# Bad: No probes
spec:
  containers:
  - name: app
    image: app:1.0.0

# Good: Define liveness and readiness probes
spec:
  containers:
  - name: app
    image: app:1.0.0
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

### 3. Using 'latest' Tag

```yaml
# Bad: Using 'latest' tag
spec:
  containers:
  - name: app
    image: myapp:latest

# Good: Use specific version tags
spec:
  containers:
  - name: app
    image: myapp:1.2.3
    imagePullPolicy: IfNotPresent
```

### 4. Storing Secrets in ConfigMaps

```yaml
# Bad: Sensitive data in ConfigMap
apiVersion: v1
kind: ConfigMap
data:
  database-password: "supersecret123"

# Good: Use Secrets
apiVersion: v1
kind: Secret
type: Opaque
stringData:
  database-password: "supersecret123"
```

### 5. Single Replica for Critical Services

```yaml
# Bad: Single point of failure
spec:
  replicas: 1

# Good: Multiple replicas for high availability
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```

## Best Practices

### 1. Use Namespaces for Organization

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
```

### 2. Label Resources Consistently

```yaml
metadata:
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-production
    app.kubernetes.io/version: "1.2.3"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: ecommerce-platform
    app.kubernetes.io/managed-by: helm
    environment: production
    team: frontend
```

### 3. Use Pod Disruption Budgets

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2  # or maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

### 4. Implement Network Policies

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-network-policy
spec:
  podSelector:
    matchLabels:
      app: react-frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend-api
    ports:
    - protocol: TCP
      port: 8080
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 53  # DNS
```

### 5. Use Resource Quotas

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
    persistentvolumeclaims: "10"
    services.loadbalancers: "5"
    pods: "100"
```

### 6. Implement Service Mesh (Istio Example)

```yaml
# virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app-routes
spec:
  hosts:
  - myapp.example.com
  gateways:
  - app-gateway
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: app-canary
        port:
          number: 8080
  - route:
    - destination:
        host: app-stable
        port:
          number: 8080
      weight: 90
    - destination:
        host: app-canary
        port:
          number: 8080
      weight: 10
```

## When to Use/Not to Use

### When to Use Kubernetes

1. **Microservices Architecture**: Managing multiple containerized services
2. **High Availability Requirements**: Need self-healing and automatic failover
3. **Dynamic Scaling**: Traffic patterns require auto-scaling
4. **Multi-Cloud Strategy**: Portable across cloud providers
5. **Complex Deployments**: Blue-green, canary, rolling updates
6. **Container Orchestration**: Managing hundreds of containers
7. **Cloud-Native Applications**: Built for distributed systems
8. **DevOps Maturity**: Team has infrastructure management capabilities

### When Not to Use Kubernetes

1. **Simple Applications**: Single container or monolith applications
2. **Small Teams**: Lack of Kubernetes expertise
3. **Low Traffic**: Static, predictable workloads
4. **Tight Budget**: Infrastructure and operational costs
5. **Quick Prototypes**: Fast time-to-market requirements
6. **Legacy Applications**: Not containerized or container-ready
7. **Minimal Scaling Needs**: Static resource requirements
8. **Serverless-Friendly**: Better suited for FaaS platforms

## Interview Questions

### 1. Explain the difference between a Deployment and a StatefulSet.

**Answer**: Deployments are for stateless applications where pods are interchangeable. StatefulSets are for stateful applications requiring stable network identities, persistent storage, and ordered deployment/scaling. StatefulSets provide guarantees about pod ordering and uniqueness with stable hostnames (app-0, app-1, app-2).

### 2. How does a rolling update work in Kubernetes?

**Answer**: Rolling updates gradually replace old pods with new ones based on `maxSurge` and `maxUnavailable` settings. Kubernetes creates new pods (up to maxSurge), waits for them to become ready, then terminates old pods (respecting maxUnavailable). This ensures zero-downtime deployments. If new pods fail readiness checks, the rollout is automatically halted.

### 3. What's the difference between ClusterIP, NodePort, and LoadBalancer services?

**Answer**: 
- **ClusterIP**: Internal service accessible only within the cluster (default)
- **NodePort**: Exposes service on each node's IP at a static port (30000-32767)
- **LoadBalancer**: Provisions cloud provider load balancer, exposing service externally

### 4. How do you handle secrets securely in Kubernetes?

**Answer**: Use Kubernetes Secrets with base64 encoding (not encryption), enable encryption at rest in etcd, use RBAC to control access, integrate external secret managers (HashiCorp Vault, AWS Secrets Manager), mount secrets as volumes (not environment variables when possible), use sealed-secrets or external-secrets operators, and rotate secrets regularly.

### 5. Explain the purpose of readiness and liveness probes.

**Answer**: 
- **Liveness probe**: Determines if container is running; if it fails, Kubernetes restarts the container
- **Readiness probe**: Determines if container is ready to serve traffic; if it fails, pod is removed from service endpoints but not restarted

### 6. How do you troubleshoot a CrashLoopBackOff error?

**Answer**: Check pod logs (`kubectl logs pod-name`), describe pod for events (`kubectl describe pod pod-name`), verify image availability, check resource limits, validate configurations and environment variables, review init containers, examine liveness/readiness probe configurations, and check for application-level errors.

### 7. What is the difference between ConfigMap and Secret?

**Answer**: Both store configuration data, but Secrets are designed for sensitive information. Secrets are base64 encoded (not encrypted), stored in etcd (optionally encrypted at rest), and have stricter RBAC controls. ConfigMaps store non-sensitive data in plain text. Secrets have a 1MB size limit per object.

### 8. How would you implement zero-downtime deployments?

**Answer**: Use rolling updates with proper `maxSurge` and `maxUnavailable` settings, implement readiness probes to ensure new pods are healthy before receiving traffic, use Pod Disruption Budgets to maintain minimum availability, configure proper preStop hooks for graceful shutdown, and consider blue-green or canary deployment strategies for critical services.

## Key Takeaways

1. **Kubernetes provides declarative infrastructure**: Define desired state in YAML; Kubernetes maintains it
2. **Pods are ephemeral**: Design applications to be stateless and resilient to pod restarts
3. **Services provide stable networking**: Use Services for reliable pod discovery and load balancing
4. **Ingress manages external access**: Centralized HTTP/HTTPS routing and SSL termination
5. **ConfigMaps and Secrets separate configuration**: Never hardcode configuration in container images
6. **Resource limits are critical**: Prevent resource starvation and ensure fair scheduling
7. **Health checks enable self-healing**: Liveness and readiness probes are essential for reliability
8. **Namespaces provide isolation**: Organize resources by environment, team, or application
9. **Rolling updates enable zero-downtime**: Proper deployment strategies prevent service disruption
10. **Autoscaling optimizes resources**: HPA and VPA automatically adjust to demand

## Resources

### Official Documentation
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [kubectl Reference](https://kubernetes.io/docs/reference/kubectl/)
- [Kubernetes API Reference](https://kubernetes.io/docs/reference/kubernetes-api/)

### Tools
- [kubectl](https://kubernetes.io/docs/tasks/tools/) - CLI tool
- [Helm](https://helm.sh/) - Package manager
- [k9s](https://k9scli.io/) - Terminal UI
- [Lens](https://k8slens.dev/) - IDE for Kubernetes
- [Skaffold](https://skaffold.dev/) - Local development

### Learning Resources
- [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [Play with Kubernetes](https://labs.play-with-k8s.com/)
- [Kubernetes Patterns](https://k8spatterns.io/)

### Books
- "Kubernetes in Action" by Marko Luksa
- "Kubernetes Up & Running" by Kelsey Hightower
- "Production Kubernetes" by Josh Rosso, Rich Lander

### Certifications
- Certified Kubernetes Administrator (CKA)
- Certified Kubernetes Application Developer (CKAD)
- Certified Kubernetes Security Specialist (CKS)
