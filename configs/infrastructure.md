# Infrastructure Configuration & Deployment

## Kubernetes Deployment Architecture

### Cluster Configuration

```yaml
# cluster-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-config
  namespace: videogen
data:
  region: "us-west-1"
  environment: "production"
  version: "v1.0.0"
  gpu_scheduler_enabled: "true"
  multi_region: "true"

---
# Node pools configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: node-pools
  namespace: videogen
data:
  node_pools: |
    - name: "gpu-a100"
      machine_type: "a2-highgpu-1g"
      min_nodes: 2
      max_nodes: 10
      disk_size_gb: 100
      gpu_type: "nvidia-tesla-a100"
      gpu_count: 1
      taints:
        - key: "nvidia.com/gpu"
          value: "a100"
          effect: "NoSchedule"
      labels:
        gpu-type: "a100"
        workload: "premium"
    
    - name: "gpu-v100"
      machine_type: "n1-standard-8"
      min_nodes: 4
      max_nodes: 20
      disk_size_gb: 100
      gpu_type: "nvidia-tesla-v100"
      gpu_count: 1
      taints:
        - key: "nvidia.com/gpu"
          value: "v100"
          effect: "NoSchedule"
      labels:
        gpu-type: "v100"
        workload: "standard"
    
    - name: "gpu-t4"
      machine_type: "n1-standard-4"
      min_nodes: 8
      max_nodes: 40
      disk_size_gb: 50
      gpu_type: "nvidia-tesla-t4"
      gpu_count: 1
      taints:
        - key: "nvidia.com/gpu"
          value: "t4"
          effect: "NoSchedule"
      labels:
        gpu-type: "t4"
        workload: "economy"
    
    - name: "cpu-workers"
      machine_type: "n2-standard-4"
      min_nodes: 10
      max_nodes: 50
      disk_size_gb: 50
      labels:
        workload: "api"
```

### Namespace & RBAC

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: videogen
  labels:
    istio-injection: enabled

---
# Service Account for applications
apiVersion: v1
kind: ServiceAccount
metadata:
  name: videogen-app
  namespace: videogen
  annotations:
    iam.gke.io/gcp-service-account: videogen-app@project-id.iam.gserviceaccount.com

---
# RBAC for GPU scheduler
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gpu-scheduler
rules:
- apiGroups: [""]
  resources: ["nodes", "pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["nodes", "pods"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gpu-scheduler-binding
subjects:
- kind: ServiceAccount
  name: gpu-scheduler
  namespace: videogen
roleRef:
  kind: ClusterRole
  name: gpu-scheduler
  apiGroup: rbac.authorization.k8s.io
```

---

## Application Deployments

### 1. API Gateway (Kong)

```yaml
# kong-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kong-gateway
  namespace: videogen
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kong-gateway
  template:
    metadata:
      labels:
        app: kong-gateway
        version: v1
    spec:
      containers:
      - name: kong
        image: kong:3.4-alpine
        ports:
        - containerPort: 8000
        - containerPort: 8443
        - containerPort: 8001
        env:
        - name: KONG_DATABASE
          value: "postgres"
        - name: KONG_PG_HOST
          value: "postgres-service"
        - name: KONG_PG_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: KONG_PROXY_ACCESS_LOG
          value: "/dev/stdout"
        - name: KONG_ADMIN_ACCESS_LOG
          value: "/dev/stdout"
        - name: KONG_PROXY_ERROR_LOG
          value: "/dev/stderr"
        - name: KONG_ADMIN_ERROR_LOG
          value: "/dev/stderr"
        - name: KONG_PLUGINS
          value: "bundled,rate-limiting,prometheus,jwt,cors"
        livenessProbe:
          httpGet:
            path: /status
            port: 8001
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /status
            port: 8001
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2
            memory: 4Gi

---
apiVersion: v1
kind: Service
metadata:
  name: kong-gateway
  namespace: videogen
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  selector:
    app: kong-gateway
  ports:
  - name: proxy
    port: 80
    targetPort: 8000
  - name: proxy-ssl
    port: 443
    targetPort: 8443
  type: LoadBalancer

---
# Kong configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: kong-config
  namespace: videogen
data:
  kong.yaml: |
    _format_version: "3.0"
    services:
    - name: auth-service
      url: http://auth-service:8080
      plugins:
      - name: rate-limiting
        config:
          minute: 60
          hour: 1000
    - name: generation-service
      url: http://generation-orchestrator:8080
      plugins:
      - name: rate-limiting
        config:
          minute: 10
          hour: 100
      - name: jwt
        config:
          key_claim_name: kid
    routes:
    - name: auth-routes
      service: auth-service
      paths:
      - /v1/auth
    - name: generation-routes
      service: generation-service
      paths:
      - /v1/generations
```

### 2. Authentication Service

```yaml
# auth-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: videogen
spec:
  replicas: 5
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
        version: v1
    spec:
      serviceAccountName: videogen-app
      containers:
      - name: auth-service
        image: gcr.io/project-id/auth-service:v1.2.3
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: jwt-secret
              key: secret
        - name: ENVIRONMENT
          value: "production"
        - name: LOG_LEVEL
          value: "info"
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
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 1
            memory: 2Gi

---
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  namespace: videogen
  labels:
    app: auth-service
spec:
  selector:
    app: auth-service
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: auth-service-hpa
  namespace: videogen
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: auth-service
  minReplicas: 3
  maxReplicas: 20
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

### 3. Generation Orchestrator

```yaml
# generation-orchestrator-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: generation-orchestrator
  namespace: videogen
spec:
  replicas: 3
  selector:
    matchLabels:
      app: generation-orchestrator
  template:
    metadata:
      labels:
        app: generation-orchestrator
        version: v1
    spec:
      serviceAccountName: videogen-app
      containers:
      - name: orchestrator
        image: gcr.io/project-id/generation-orchestrator:v1.4.5
        ports:
        - containerPort: 8080
        - containerPort: 8090  # Metrics port
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: url
        - name: S3_BUCKET_NAME
          value: "videogen-storage-prod"
        - name: AWS_REGION
          value: "us-west-1"
        - name: CELERY_BROKER_URL
          valueFrom:
            secretKeyRef:
              name: celery-secret
              key: broker_url
        - name: GPU_SCHEDULER_URL
          value: "http://gpu-scheduler:8080"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2
            memory: 4Gi

---
apiVersion: v1
kind: Service
metadata:
  name: generation-orchestrator
  namespace: videogen
spec:
  selector:
    app: generation-orchestrator
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: metrics
    port: 8090
    targetPort: 8090
  type: ClusterIP
```

### 4. GPU Inference Workers

```yaml
# gpu-inference-workers.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-inference-workers
  namespace: videogen
spec:
  selector:
    matchLabels:
      app: gpu-inference-workers
  template:
    metadata:
      labels:
        app: gpu-inference-workers
        version: v1
    spec:
      serviceAccountName: videogen-app
      nodeSelector:
        accelerator: nvidia-tesla-a100  # Deploy only on GPU nodes
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      containers:
      - name: inference-worker
        image: gcr.io/project-id/inference-worker:v2.1.0
        ports:
        - containerPort: 8080
        - containerPort: 8000  # Triton inference server
        env:
        - name: NVIDIA_VISIBLE_DEVICES
          value: "all"
        - name: CUDA_VISIBLE_DEVICES
          value: "0"  # Will be overridden by GPU scheduler
        - name: MODEL_REPOSITORY
          value: "/models"
        - name: CELERY_BROKER_URL
          valueFrom:
            secretKeyRef:
              name: celery-secret
              key: broker_url
        - name: S3_BUCKET_NAME
          value: "videogen-storage-prod"
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: model-cache
          mountPath: /models
        - name: temp-storage
          mountPath: /tmp
        resources:
          requests:
            nvidia.com/gpu: 1
            cpu: 4
            memory: 16Gi
          limits:
            nvidia.com/gpu: 1
            cpu: 8
            memory: 32Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 300  # Model loading takes time
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
      volumes:
      - name: model-cache
        hostPath:
          path: /var/lib/models
          type: DirectoryOrCreate
      - name: temp-storage
        emptyDir:
          sizeLimit: 50Gi

---
# Service for inference workers
apiVersion: v1
kind: Service
metadata:
  name: inference-workers
  namespace: videogen
spec:
  selector:
    app: gpu-inference-workers
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: triton
    port: 8000
    targetPort: 8000
  type: ClusterIP
  clusterIP: None  # Headless service for direct pod access
```

### 5. GPU Scheduler

```yaml
# gpu-scheduler-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-scheduler
  namespace: videogen
spec:
  replicas: 2  # High availability
  selector:
    matchLabels:
      app: gpu-scheduler
  template:
    metadata:
      labels:
        app: gpu-scheduler
        version: v1
    spec:
      serviceAccountName: gpu-scheduler
      containers:
      - name: scheduler
        image: gcr.io/project-id/gpu-scheduler:v1.1.2
        ports:
        - containerPort: 8080
        - containerPort: 9090  # Metrics
        env:
        - name: KUBERNETES_NAMESPACE
          value: "videogen"
        - name: PROMETHEUS_URL
          value: "http://prometheus:9090"
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: url
        - name: LOG_LEVEL
          value: "info"
        - name: SCHEDULER_INTERVAL_SECONDS
          value: "30"
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 1
            memory: 2Gi
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

---
apiVersion: v1
kind: Service
metadata:
  name: gpu-scheduler
  namespace: videogen
spec:
  selector:
    app: gpu-scheduler
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: metrics
    port: 9090
    targetPort: 9090
  type: ClusterIP
```

---

## Database Deployments

### PostgreSQL (Primary Database)

```yaml
# postgresql-deployment.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: videogen
spec:
  serviceName: postgresql
  replicas: 3  # Primary + 2 replicas
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
      - name: postgresql
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: "videogen"
        - name: POSTGRES_USER
          value: "videogen_user"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: POSTGRES_REPLICATION_USER
          value: "replicator"
        - name: POSTGRES_REPLICATION_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: replication_password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        - name: postgres-config
          mountPath: /etc/postgresql/postgresql.conf
          subPath: postgresql.conf
        resources:
          requests:
            cpu: 2
            memory: 4Gi
          limits:
            cpu: 4
            memory: 8Gi
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - videogen_user
            - -d
            - videogen
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - videogen_user
            - -d
            - videogen
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: postgres-config
        configMap:
          name: postgres-config
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "ssd-storage"
      resources:
        requests:
          storage: 500Gi

---
apiVersion: v1
kind: Service
metadata:
  name: postgresql
  namespace: videogen
spec:
  selector:
    app: postgresql
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP

---
# PostgreSQL configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: videogen
data:
  postgresql.conf: |
    # Memory settings
    shared_buffers = 2GB
    effective_cache_size = 6GB
    work_mem = 16MB
    maintenance_work_mem = 512MB
    
    # Connection settings
    max_connections = 200
    
    # WAL settings
    wal_level = replica
    max_wal_senders = 3
    wal_keep_segments = 64
    
    # Performance tuning
    checkpoint_completion_target = 0.9
    random_page_cost = 1.1
    effective_io_concurrency = 200
    
    # Logging
    log_destination = 'stderr'
    log_statement = 'mod'
    log_min_duration_statement = 1000
```

### Redis Cluster

```yaml
# redis-cluster-deployment.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
  namespace: videogen
spec:
  serviceName: redis-cluster
  replicas: 6  # 3 masters + 3 slaves
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        - containerPort: 16379  # Cluster bus port
        command:
        - redis-server
        - /etc/redis/redis.conf
        - --cluster-enabled
        - "yes"
        - --cluster-config-file
        - /data/nodes.conf
        - --cluster-node-timeout
        - "5000"
        - --appendonly
        - "yes"
        volumeMounts:
        - name: redis-data
          mountPath: /data
        - name: redis-config
          mountPath: /etc/redis/
        resources:
          requests:
            cpu: 500m
            memory: 2Gi
          limits:
            cpu: 1
            memory: 4Gi
        livenessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: redis-config
        configMap:
          name: redis-config
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "ssd-storage"
      resources:
        requests:
          storage: 100Gi

---
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
  namespace: videogen
spec:
  clusterIP: None  # Headless service
  selector:
    app: redis-cluster
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
  - name: cluster
    port: 16379
    targetPort: 16379

---
# Redis configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: videogen
data:
  redis.conf: |
    # Network
    bind 0.0.0.0
    port 6379
    
    # Memory
    maxmemory 3gb
    maxmemory-policy allkeys-lru
    
    # Persistence
    save 900 1
    save 300 10
    save 60 10000
    
    # Security
    requirepass changeme123
    
    # Cluster
    cluster-enabled yes
    cluster-config-file nodes.conf
    cluster-node-timeout 5000
```

---

## Monitoring & Observability

### Prometheus Stack

```yaml
# prometheus-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:v2.40.0
        ports:
        - containerPort: 9090
        args:
        - --config.file=/etc/prometheus/prometheus.yml
        - --storage.tsdb.path=/prometheus/
        - --web.console.libraries=/etc/prometheus/console_libraries
        - --web.console.templates=/etc/prometheus/consoles
        - --storage.tsdb.retention.time=30d
        - --web.enable-lifecycle
        - --web.enable-admin-api
        volumeMounts:
        - name: prometheus-config
          mountPath: /etc/prometheus/
        - name: prometheus-storage
          mountPath: /prometheus/
        resources:
          requests:
            cpu: 1
            memory: 4Gi
          limits:
            cpu: 2
            memory: 8Gi
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
      - name: prometheus-storage
        persistentVolumeClaim:
          claimName: prometheus-pvc

---
# Prometheus configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 30s
      evaluation_interval: 30s
    
    rule_files:
    - "/etc/prometheus/rules/*.yml"
    
    scrape_configs:
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
    
    - job_name: 'gpu-metrics'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
          - videogen
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: gpu-inference-workers
    
    - job_name: 'api-services'
      kubernetes_sd_configs:
      - role: service
        namespaces:
          names:
          - videogen
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true

  rules.yml: |
    groups:
    - name: videogen-alerts
      rules:
      - alert: HighGPUUtilization
        expr: avg(gpu_utilization_percent) by (node) > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High GPU utilization on {{ $labels.node }}"
      
      - alert: LongQueueWaitTime
        expr: avg(generation_queue_wait_time_seconds) > 300
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Generation queue wait time is too long"
      
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
```

### Grafana Dashboard

```yaml
# grafana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:9.3.0
        ports:
        - containerPort: 3000
        env:
        - name: GF_SECURITY_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: grafana-secret
              key: admin-password
        - name: GF_INSTALL_PLUGINS
          value: "grafana-kubernetes-app"
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
        - name: grafana-datasources
          mountPath: /etc/grafana/provisioning/datasources
        - name: grafana-dashboards
          mountPath: /etc/grafana/provisioning/dashboards
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
      volumes:
      - name: grafana-storage
        persistentVolumeClaim:
          claimName: grafana-pvc
      - name: grafana-datasources
        configMap:
          name: grafana-datasources
      - name: grafana-dashboards
        configMap:
          name: grafana-dashboards

---
# Grafana datasources
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      access: proxy
      url: http://prometheus:9090
      isDefault: true
    - name: InfluxDB
      type: influxdb
      access: proxy
      url: http://influxdb:8086
      database: videogen_metrics
```

---

## Security Configuration

### Network Policies

```yaml
# network-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: videogen
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-services
  namespace: videogen
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: kong-gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgresql
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          app: redis-cluster
    ports:
    - protocol: TCP
      port: 6379

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-gpu-workers
  namespace: videogen
spec:
  podSelector:
    matchLabels:
      app: gpu-inference-workers
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: generation-orchestrator
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - {} # Allow all egress for model downloads and S3 access
```

### Pod Security Standards

```yaml
# pod-security-policy.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: videogen
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Secrets Management

```yaml
# secrets.yaml - Use with external secret management
apiVersion: v1
kind: Secret
metadata:
  name: database-secret
  namespace: videogen
type: Opaque
data:
  url: <base64-encoded-database-url>
  username: <base64-encoded-username>
  password: <base64-encoded-password>

---
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
  namespace: videogen
type: Opaque
data:
  url: <base64-encoded-redis-url>
  password: <base64-encoded-password>

---
apiVersion: v1
kind: Secret
metadata:
  name: jwt-secret
  namespace: videogen
type: Opaque
data:
  secret: <base64-encoded-jwt-secret>

---
# External Secrets Operator configuration
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcpsm-secret-store
  namespace: videogen
spec:
  provider:
    gcpsm:
      projectId: "your-gcp-project"
      auth:
        workloadIdentity:
          clusterLocation: us-west1
          clusterName: videogen-cluster
          serviceAccountRef:
            name: external-secrets-sa
```

---

## Terraform Infrastructure

### GCP Infrastructure

```hcl
# main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 4.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
  
  backend "gcs" {
    bucket = "videogen-terraform-state"
    prefix = "terraform/state"
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# GKE Cluster
resource "google_container_cluster" "videogen_cluster" {
  name     = "videogen-cluster"
  location = var.region
  
  remove_default_node_pool = true
  initial_node_count       = 1
  
  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.subnet.name
  
  networking_mode = "VPC_NATIVE"
  ip_allocation_policy {
    cluster_ipv4_cidr_block  = "10.96.0.0/11"
    services_ipv4_cidr_block = "10.94.0.0/18"
  }
  
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
  
  addons_config {
    http_load_balancing {
      disabled = false
    }
    horizontal_pod_autoscaling {
      disabled = false
    }
    network_policy_config {
      disabled = false
    }
  }
  
  master_auth {
    client_certificate_config {
      issue_client_certificate = false
    }
  }
}

# GPU Node Pools
resource "google_container_node_pool" "gpu_a100_pool" {
  name       = "gpu-a100-pool"
  cluster    = google_container_cluster.videogen_cluster.name
  location   = var.region
  node_count = 2
  
  autoscaling {
    min_node_count = 1
    max_node_count = 10
  }
  
  node_config {
    machine_type = "a2-highgpu-1g"
    disk_size_gb = 100
    disk_type    = "pd-ssd"
    
    guest_accelerator {
      type  = "nvidia-tesla-a100"
      count = 1
    }
    
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
    
    taint {
      key    = "nvidia.com/gpu"
      value  = "a100"
      effect = "NO_SCHEDULE"
    }
    
    labels = {
      gpu-type = "a100"
      workload = "premium"
    }
  }
}

# CPU Node Pool
resource "google_container_node_pool" "cpu_pool" {
  name       = "cpu-pool"
  cluster    = google_container_cluster.videogen_cluster.name
  location   = var.region
  node_count = 3
  
  autoscaling {
    min_node_count = 3
    max_node_count = 20
  }
  
  node_config {
    machine_type = "n2-standard-4"
    disk_size_gb = 50
    disk_type    = "pd-standard"
    
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
    
    labels = {
      workload = "api"
    }
  }
}

# Cloud Storage
resource "google_storage_bucket" "videogen_storage" {
  name     = "${var.project_id}-videogen-storage"
  location = "US"
  
  versioning {
    enabled = true
  }
  
  lifecycle_rule {
    condition {
      age = 30
    }
    action {
      type = "Delete"
    }
  }
}

# Cloud SQL (PostgreSQL)
resource "google_sql_database_instance" "videogen_db" {
  name             = "videogen-db-instance"
  database_version = "POSTGRES_15"
  region           = var.region
  
  settings {
    tier = "db-custom-4-16384"
    
    disk_autoresize = true
    disk_size       = 500
    disk_type       = "PD_SSD"
    
    backup_configuration {
      enabled                        = true
      start_time                     = "03:00"
      point_in_time_recovery_enabled = true
      backup_retention_settings {
        retained_backups = 30
      }
    }
    
    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.vpc.id
    }
  }
}

# Variables
variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "region" {
  description = "GCP Region"
  type        = string
  default     = "us-west1"
}
```

This comprehensive infrastructure configuration provides a production-ready, scalable deployment for the Image-to-Video AI platform with proper resource allocation, monitoring, security, and cost optimization.
