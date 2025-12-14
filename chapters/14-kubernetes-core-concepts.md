# Chapter 14: Kubernetes Core Concepts

> **First Principles Question**: Now that you understand why Kubernetes exists, what are the fundamental building blocks? What mental model should you have for Pods, Deployments, Services, and the other core resources?

---

## Chapter Overview

Kubernetes has many concepts and resources. This chapter focuses on the core building blocks you'll work with daily, building a mental model that makes the rest of Kubernetes make sense.

**What readers will understand after this chapter:**
- The resource model and API structure
- Pods: the atomic unit of deployment
- Workload resources: Deployments, StatefulSets, DaemonSets, Jobs
- Service and Ingress: networking abstractions
- ConfigMaps and Secrets: configuration management
- Namespaces and labels: organization
- kubectl: the primary interface

---

## Section 1: The Kubernetes API Model

### 1.1 Everything Is a Resource

**Core Principle:**
In Kubernetes, everything is represented as a resource (object) with:
- **apiVersion**: Which API group/version
- **kind**: What type of resource
- **metadata**: Identity and annotations
- **spec**: Desired state
- **status**: Current state (managed by Kubernetes)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: default
  labels:
    app: myapp
spec:
  containers:
    - name: app
      image: myapp:v1
status:
  phase: Running
  podIP: 10.244.1.5
```

### 1.2 Declarative Model

**You Declare What You Want:**
```yaml
# "I want 3 replicas of myapp running v2"
spec:
  replicas: 3
  template:
    spec:
      containers:
        - image: myapp:v2
```

**Kubernetes Makes It Happen:**
1. Controller reads your declaration
2. Compares to actual state
3. Takes actions to reconcile
4. Repeats forever

### 1.3 API Groups

**Core API (`/api/v1`):**
- Pods
- Services
- ConfigMaps
- Secrets
- Namespaces

**Named API Groups (`/apis/{group}/{version}`):**
- `apps/v1`: Deployments, StatefulSets, DaemonSets
- `batch/v1`: Jobs, CronJobs
- `networking.k8s.io/v1`: Ingress, NetworkPolicy
- `autoscaling/v2`: HorizontalPodAutoscaler

```yaml
# Core API
apiVersion: v1
kind: Pod

# Named API group
apiVersion: apps/v1
kind: Deployment
```

---

## Section 2: Pods — The Atomic Unit

### 2.1 What Is a Pod?

**Definition:**
The smallest deployable unit in Kubernetes. One or more containers that:
- Share network namespace (same IP, can use localhost)
- Share storage volumes
- Are scheduled together on the same node
- Have a shared lifecycle

### 2.2 Why Pods, Not Just Containers?

**Co-location Requirements:**
```yaml
# App needs a sidecar for logging
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: app
      image: myapp:v1
      ports:
        - containerPort: 8080
    - name: log-shipper
      image: fluent-bit:latest
      # Shares filesystem with app
```

**Common Patterns:**
- **Sidecar**: Logging, monitoring, proxying
- **Ambassador**: Proxy to external services
- **Adapter**: Transform data for consumption

### 2.3 Pod Anatomy

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
    version: v1
spec:
  # Init containers run first, in order
  initContainers:
    - name: init-db
      image: busybox
      command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']

  # Main containers run together
  containers:
    - name: app
      image: myapp:v1
      ports:
        - containerPort: 8080
      env:
        - name: DATABASE_URL
          value: "postgres://db:5432/myapp"
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
          port: 8080
        initialDelaySeconds: 30
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        periodSeconds: 5
      volumeMounts:
        - name: config
          mountPath: /etc/config

  # Volumes available to containers
  volumes:
    - name: config
      configMap:
        name: app-config

  # Restart policy
  restartPolicy: Always
```

### 2.4 Probes

**Liveness Probe:**
"Is this container alive?"
- Fails → Kubernetes kills and restarts container
- Use for: detecting deadlocks, unrecoverable states

**Readiness Probe:**
"Is this container ready to receive traffic?"
- Fails → Pod removed from Service endpoints
- Use for: startup warmup, temporary unavailability

**Startup Probe:**
"Has this container started?"
- Fails → Container killed
- Use for: slow-starting applications

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /started
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
  # 30 * 10 = 300 seconds to start
```

### 2.5 Pod Lifecycle

```
┌──────────────────────────────────────────────────────────────────┐
│                         Pod Phases                                │
│                                                                  │
│  Pending → Running → Succeeded/Failed                            │
│     │         │           │                                      │
│     │         │           └─► Completed or crashed               │
│     │         │                                                  │
│     │         └─► At least one container running                 │
│     │                                                            │
│     └─► Scheduled, waiting for containers to start               │
│                                                                  │
│  Additional states:                                              │
│  • Unknown: Node communication lost                              │
│  • Terminating: Graceful shutdown in progress                    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Section 3: Workload Resources

### 3.1 Deployment

**Purpose:**
Manage stateless application replicas with rolling updates.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
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
    spec:
      containers:
        - name: app
          image: myapp:v1
          ports:
            - containerPort: 8080
```

**Key Features:**
- Manages ReplicaSets (which manage Pods)
- Rolling updates and rollbacks
- Scaling (manual or auto)
- History of deployments

**Common Operations:**
```bash
# Scale
kubectl scale deployment myapp --replicas=5

# Update image
kubectl set image deployment/myapp app=myapp:v2

# View rollout status
kubectl rollout status deployment/myapp

# Rollback
kubectl rollout undo deployment/myapp

# View history
kubectl rollout history deployment/myapp
```

### 3.2 StatefulSet

**Purpose:**
Manage stateful applications with stable identities.

**Differences from Deployment:**
| Deployment | StatefulSet |
|------------|-------------|
| Pods are interchangeable | Pods have stable identities |
| Random pod names (myapp-xyz) | Ordered names (myapp-0, myapp-1) |
| Parallel scaling | Ordered scaling |
| No stable storage | Persistent volume per pod |

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

**What You Get:**
```
Pods:
  postgres-0  (always first)
  postgres-1  (after 0 is ready)
  postgres-2  (after 1 is ready)

DNS:
  postgres-0.postgres.default.svc.cluster.local
  postgres-1.postgres.default.svc.cluster.local
  postgres-2.postgres.default.svc.cluster.local

Volumes:
  data-postgres-0  (bound to postgres-0)
  data-postgres-1  (bound to postgres-1)
  data-postgres-2  (bound to postgres-2)
```

**Use Cases:**
- Databases
- Distributed systems (Kafka, ZooKeeper, etcd)
- Anything needing stable identity

### 3.3 DaemonSet

**Purpose:**
Run exactly one pod on every (or selected) node.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
        - name: node-exporter
          image: prom/node-exporter:latest
          ports:
            - containerPort: 9100
```

**Use Cases:**
- Log collectors (Fluentd, Filebeat)
- Monitoring agents (Node Exporter, Datadog)
- Network plugins (Calico, Cilium)
- Storage daemons

### 3.4 Job and CronJob

**Job:**
Run a task to completion.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: myapp:migrate
          command: ["./migrate.sh"]
```

**CronJob:**
Run jobs on a schedule.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: backup-tool:latest
              command: ["./backup.sh"]
```

---

## Section 4: Services — Network Abstraction

### 4.1 The Problem Services Solve

```
Without Services:
- Pods get random IPs
- Pods can die and be replaced
- How do clients find pods?

With Services:
- Stable DNS name
- Stable cluster IP
- Automatic load balancing to pods
```

### 4.2 Service Types

**ClusterIP (Default):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
```

Result:
- DNS: `myapp.default.svc.cluster.local`
- Cluster IP: `10.96.0.100` (stable)
- Only accessible within cluster

**NodePort:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

Result:
- All above, plus
- Accessible via `<any-node-ip>:30080`

**LoadBalancer:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
```

Result:
- All above, plus
- Cloud provider creates load balancer
- External IP assigned

**Headless (StatefulSet DNS):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None  # Headless
  selector:
    app: postgres
  ports:
    - port: 5432
```

Result:
- No ClusterIP
- DNS returns pod IPs directly
- Enables StatefulSet stable DNS names

### 4.3 How Services Work

```
┌─────────────────────────────────────────────────────────────────┐
│                           Service                               │
│                                                                 │
│  DNS: myapp → ClusterIP (10.96.0.100)                          │
│                                                                 │
│  Endpoints (auto-updated):                                     │
│  10.96.0.100:80 → [10.244.1.5:8080, 10.244.2.6:8080, ...]     │
│                                                                 │
│  kube-proxy on each node:                                      │
│  - Watches endpoints                                           │
│  - Updates iptables/ipvs rules                                 │
│  - Handles load balancing                                      │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 Ingress

**Purpose:**
HTTP/HTTPS routing to services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

**Ingress Controller Required:**
- NGINX Ingress Controller
- Traefik
- AWS ALB Ingress Controller
- Many others

---

## Section 5: Configuration

### 5.1 ConfigMaps

**Purpose:**
Non-sensitive configuration data.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Simple key-value
  LOG_LEVEL: "INFO"
  MAX_CONNECTIONS: "100"

  # File content
  config.yaml: |
    server:
      port: 8080
    logging:
      level: info
```

**Using ConfigMaps:**

**As Environment Variables:**
```yaml
spec:
  containers:
    - name: app
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
```

**As Mounted Files:**
```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - name: config
          mountPath: /etc/config
  volumes:
    - name: config
      configMap:
        name: app-config
```

### 5.2 Secrets

**Purpose:**
Sensitive data (passwords, tokens, keys).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=     # base64 encoded
  password: cGFzc3dvcmQ=  # base64 encoded
```

**Create from Command Line:**
```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=secret123
```

**Using Secrets:**
```yaml
spec:
  containers:
    - name: app
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      volumeMounts:
        - name: secrets
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secrets
      secret:
        secretName: db-credentials
```

**Important:**
- Secrets are base64 encoded, NOT encrypted
- Enable encryption at rest for production
- Consider external secret managers (Vault, AWS Secrets Manager)

---

## Section 6: Organization

### 6.1 Namespaces

**Purpose:**
Logical isolation within a cluster.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

**Use Cases:**
- Environment separation (dev, staging, prod)
- Team separation
- Multi-tenancy

**Built-in Namespaces:**
- `default`: Where resources go without explicit namespace
- `kube-system`: Kubernetes components
- `kube-public`: Public resources

**Working with Namespaces:**
```bash
# List namespaces
kubectl get namespaces

# Create resource in namespace
kubectl apply -f pod.yaml -n production

# Set default namespace for context
kubectl config set-context --current --namespace=production

# List pods in all namespaces
kubectl get pods -A
```

### 6.2 Labels and Selectors

**Labels:**
Key-value pairs attached to resources.

```yaml
metadata:
  labels:
    app: myapp
    version: v2
    environment: production
    team: backend
```

**Selectors:**
Query resources by labels.

```bash
# Get pods with label app=myapp
kubectl get pods -l app=myapp

# Get pods with multiple labels
kubectl get pods -l app=myapp,environment=production

# Get pods NOT in production
kubectl get pods -l 'environment!=production'
```

**In Resources:**
```yaml
# Service selector
spec:
  selector:
    app: myapp
    version: v2

# Deployment selector
spec:
  selector:
    matchLabels:
      app: myapp
    matchExpressions:
      - key: environment
        operator: In
        values:
          - production
          - staging
```

### 6.3 Annotations

**Purpose:**
Non-identifying metadata (tools, documentation).

```yaml
metadata:
  annotations:
    description: "Main application deployment"
    owner: "backend-team@example.com"
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
```

**Common Uses:**
- Build/release information
- Tool configuration
- External integrations

---

## Section 7: kubectl — The CLI

### 7.1 Essential Commands

**Creating/Applying:**
```bash
# Apply configuration
kubectl apply -f deployment.yaml

# Apply all files in directory
kubectl apply -f ./manifests/

# Create from literal
kubectl create deployment nginx --image=nginx
```

**Viewing:**
```bash
# List resources
kubectl get pods
kubectl get pods -o wide              # More detail
kubectl get pods -o yaml              # Full YAML
kubectl get pods -w                   # Watch for changes

# Describe (detailed info)
kubectl describe pod myapp-xyz

# Logs
kubectl logs myapp-xyz
kubectl logs myapp-xyz -f             # Follow
kubectl logs myapp-xyz -c container   # Specific container
kubectl logs myapp-xyz --previous     # Previous container
```

**Interacting:**
```bash
# Execute command
kubectl exec myapp-xyz -- ls /app
kubectl exec -it myapp-xyz -- /bin/bash

# Port forward
kubectl port-forward myapp-xyz 8080:8080
kubectl port-forward svc/myapp 8080:80
```

**Modifying:**
```bash
# Scale
kubectl scale deployment myapp --replicas=5

# Update image
kubectl set image deployment/myapp app=myapp:v2

# Edit in place
kubectl edit deployment myapp

# Delete
kubectl delete pod myapp-xyz
kubectl delete -f deployment.yaml
```

### 7.2 Context and Configuration

```bash
# View current context
kubectl config current-context

# List contexts
kubectl config get-contexts

# Switch context
kubectl config use-context production

# Set default namespace
kubectl config set-context --current --namespace=myapp
```

### 7.3 Debugging Commands

```bash
# Get events
kubectl get events --sort-by='.lastTimestamp'

# Describe for troubleshooting
kubectl describe pod myapp-xyz | tail -20

# Check resource usage
kubectl top pods
kubectl top nodes

# Debug with temporary container
kubectl debug myapp-xyz -it --image=busybox
```

---

## Section 8: Putting It All Together

### 8.1 Complete Application Example

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp

---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: myapp
data:
  LOG_LEVEL: "INFO"

---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: myapp
type: Opaque
data:
  database-password: c2VjcmV0MTIz

---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
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
        - name: app
          image: myapp:v1
          ports:
            - containerPort: 8080
          env:
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: myapp-config
                  key: LOG_LEVEL
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: database-password
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080

---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080

---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80

---
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
  namespace: myapp
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
```

### 8.2 Deploy and Verify

```bash
# Apply all resources
kubectl apply -f ./manifests/

# Verify
kubectl get all -n myapp

# Check pods are running
kubectl get pods -n myapp -w

# Check logs
kubectl logs -n myapp deployment/myapp

# Test service
kubectl port-forward -n myapp svc/myapp 8080:80
curl localhost:8080/health
```

---

## Section 9: Now You Understand Why...

Armed with this foundation, you can now understand:

- **Why Pods are the atomic unit**: Containers need co-location, shared resources

- **Why Deployments manage ReplicaSets**: Enables rollback by keeping history

- **Why Services are separate from Pods**: Stable identity must survive pod changes

- **Why StatefulSets exist**: Some apps need stable identity and storage

- **Why labels are everywhere**: Loose coupling through selection

- **Why namespaces matter**: Organization, isolation, resource quotas

---

## Practical Exercises

### Exercise 1: Deploy an Application
Deploy a 3-tier application:
1. Frontend (Deployment + Service)
2. Backend API (Deployment + Service)
3. Database (StatefulSet + Headless Service)

### Exercise 2: Rolling Update
1. Deploy version 1 of an app
2. Update to version 2
3. Watch the rolling update
4. Roll back to version 1

### Exercise 3: Debugging
1. Deploy a broken application (wrong image)
2. Use kubectl to diagnose
3. Fix and redeploy

### Exercise 4: Auto-scaling
1. Deploy with HPA
2. Generate load
3. Watch scaling happen

---

## Key Takeaways

1. **Pods are the atomic unit**, but you rarely create them directly.

2. **Deployments for stateless**, StatefulSets for stateful workloads.

3. **Services provide stable networking**, independent of pod lifecycle.

4. **Labels connect resources**. Master label selectors.

5. **ConfigMaps for config, Secrets for sensitive data** (but encrypt at rest).

6. **Namespaces organize**, but don't provide strong isolation.

7. **kubectl is your interface**. Learn it well.

---

## Looking Ahead

With Kubernetes core concepts understood, Part 5 explores cloud-native architecture: the cloud mental model, AWS services, infrastructure as code, and cloud architecture patterns.

---

## Chapter 14 Summary Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KUBERNETES CORE CONCEPTS                                  │
│                                                                             │
│  WORKLOAD RESOURCES                                                         │
│  ──────────────────                                                         │
│                                                                             │
│  Deployment       StatefulSet      DaemonSet        Job/CronJob            │
│  ───────────      ───────────      ─────────        ───────────            │
│  Stateless apps   Stateful apps    One per node     Run to completion      │
│  Rolling updates  Stable identity  Log collectors   Batch processing       │
│  Scaling          Persistent data  Node agents      Scheduled tasks        │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  NETWORKING                                                                 │
│  ──────────                                                                 │
│                                                                             │
│  Service Types:                                                             │
│  ClusterIP    → Internal only (default)                                    │
│  NodePort     → Exposed on node IPs                                        │
│  LoadBalancer → Cloud load balancer                                        │
│  Headless     → Direct pod DNS (StatefulSets)                              │
│                                                                             │
│  Ingress:                                                                   │
│  HTTP/HTTPS routing, TLS termination, path-based routing                   │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  CONFIGURATION                                                              │
│  ─────────────                                                              │
│                                                                             │
│  ConfigMap:  Non-sensitive config (env vars, files)                        │
│  Secret:     Sensitive data (passwords, tokens) - base64, not encrypted    │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  ORGANIZATION                                                               │
│  ────────────                                                               │
│                                                                             │
│  Namespaces:  Logical isolation (dev, staging, prod)                       │
│  Labels:      Key-value pairs for selection (app: myapp)                   │
│  Annotations: Non-identifying metadata (prometheus.io/scrape: "true")      │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  ESSENTIAL kubectl                                                          │
│  ─────────────────                                                          │
│                                                                             │
│  kubectl apply -f file.yaml       # Create/update resources                │
│  kubectl get pods -o wide         # List with details                      │
│  kubectl describe pod myapp       # Detailed info                          │
│  kubectl logs myapp -f            # Stream logs                            │
│  kubectl exec -it myapp -- bash   # Interactive shell                      │
│  kubectl port-forward pod 8080    # Local access                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Next Chapter: The Cloud Mental Model — Understanding cloud computing from first principles*
