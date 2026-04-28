# Kubernetes Complete Learning Guide (EKS + Networking)
## Step-by-Step Hands-On Tutorial

---

## Table of Contents
1. [Prerequisites & Setup](#prerequisites--setup)
2. [Module 1: Kubernetes Fundamentals](#module-1-kubernetes-fundamentals)
3. [Module 2: Pods](#module-2-pods)
4. [Module 3: Deployments](#module-3-deployments)
5. [Module 4: Services & Networking](#module-4-services--networking)
6. [Module 5: ConfigMaps & Secrets](#module-5-configmaps--secrets)
7. [Module 6: Persistent Storage](#module-6-persistent-storage)
8. [Module 7: Advanced Controllers](#module-7-advanced-controllers)
9. [Module 8: Networking Deep Dive](#module-8-networking-deep-dive)
10. [Module 9: Troubleshooting](#module-9-troubleshooting)

---

# Prerequisites & Setup

## 1. Verify Your EKS Cluster

```powershell
# Check if kubectl is installed
kubectl version --client

# Check cluster info
kubectl cluster-info

# Check nodes
kubectl get nodes

# Check node details (should show 2 nodes)
kubectl get nodes -o wide
```

**Expected Output:**
```
NAME                          STATUS   ROLES    AGE   VERSION
ip-10-0-1-100.ec2.internal   Ready    <none>   5d    v1.28.0
ip-10-0-1-101.ec2.internal   Ready    <none>   5d    v1.28.0
```

## 2. Set Up Namespaces for Learning

```powershell
# Create namespaces for different modules
kubectl create namespace learning-dev
kubectl create namespace learning-prod
kubectl create namespace learning-networking

# List namespaces
kubectl get namespaces
```

## 3. Useful Commands to Know

```powershell
# Check API version
kubectl api-versions

# Check all resources available
kubectl api-resources

# Get context info
kubectl config current-context

# Get cluster info
kubectl get nodes
kubectl get pods --all-namespaces
kubectl get services --all-namespaces
```

---

# Module 1: Kubernetes Fundamentals

## Understanding Kubernetes Architecture

### Components in Your EKS Cluster:

```
┌─────────────────────────────────────────┐
│         AWS EKS Control Plane           │
│  (AWS manages this - you don't manage)  │
│  - API Server                           │
│  - etcd (database)                      │
│  - Scheduler                            │
│  - Controller Manager                   │
└─────────────────────────────────────────┘
           ↓              ↓
    ┌──────────┐    ┌──────────┐
    │  Node 1  │    │  Node 2  │
    │(kubelet) │    │(kubelet) │
    │(docker)  │    │(docker)  │
    └──────────┘    └──────────┘
```

### Key Concepts

| Concept | What it does |
|---------|-------------|
| **Pod** | Smallest unit, contains one or more containers |
| **Deployment** | Manages Pods, ensures desired replicas are running |
| **Service** | Exposes Pods to network (internal or external) |
| **Namespace** | Virtual cluster to organize resources |
| **Label** | Sticker/tag on resources for selection |
| **Selector** | Way to find resources by labels |

---

# Module 2: Pods

## What is a Pod?

A Pod is the **smallest deployable unit** in Kubernetes. It's like a container wrapper.

- Can contain **1 or more containers** (usually 1)
- Containers share network namespace (same IP, different ports)
- Containers share storage
- **Ephemeral** (temporary, gets deleted)

### Exercise 2.1: Create Your First Pod

**File: `pod-nginx.yaml`**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: learning-dev
  labels:
    app: nginx
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
```

**Apply and test:**

```powershell
# Create the Pod
kubectl apply -f pod-nginx.yaml

# Check if Pod is running
kubectl get pods -n learning-dev

# Get detailed info
kubectl describe pod nginx-pod -n learning-dev

# Check logs
kubectl logs nginx-pod -n learning-dev

# Access the Pod (port-forward)
kubectl port-forward nginx-pod 8080:80 -n learning-dev

# Test in another terminal
curl http://localhost:8080

# Delete the Pod
kubectl delete pod nginx-pod -n learning-dev
```

### Exercise 2.2: Multi-Container Pod

**File: `pod-multi-container.yaml`**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app-pod
  namespace: learning-dev
spec:
  containers:
  # Container 1: Web server
  - name: web
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  
  # Container 2: Content generator
  - name: sidecar
    image: busybox:1.35
    command: ["/bin/sh"]
    args:
      - -c
      - while true; do echo "<h1>Pod Name: $(hostname)</h1>" > /shared/index.html; sleep 10; done
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  
  volumes:
  - name: shared-data
    emptyDir: {}
```

**Apply and test:**

```powershell
# Create Pod
kubectl apply -f pod-multi-container.yaml

# Check both containers
kubectl get pods -n learning-dev
kubectl describe pod web-app-pod -n learning-dev

# Execute command in specific container
kubectl exec web-app-pod -c web -it -- /bin/bash

# Port forward and test
kubectl port-forward web-app-pod 8080:80 -n learning-dev
curl http://localhost:8080
```

### Key Takeaway
- Pods are **ephemeral** (temporary)
- **Don't create Pods directly in production** - use Deployments
- Pods can have **multiple containers** but usually just one

---

# Module 3: Deployments

## What is a Deployment?

A Deployment **manages Pods** and ensures:
- Desired number of replicas are always running
- Automatic restart if Pod fails
- Easy scaling up/down
- Rolling updates (update image without downtime)

### Exercise 3.1: Create a Deployment

**File: `deployment-nginx.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: learning-dev
  labels:
    app: nginx
spec:
  replicas: 3  # Creates 3 Pods
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max 1 extra Pod during update
      maxUnavailable: 1  # Min 1 Pod always available
  
  selector:
    matchLabels:
      app: nginx         # Manages Pods with this label
  
  template:              # Pod template
    metadata:
      labels:
        app: nginx       # Pod label
        version: v1
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        # Health checks
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Apply and test:**

```powershell
# Create Deployment
kubectl apply -f deployment-nginx.yaml

# Check Deployment status
kubectl get deployment -n learning-dev
kubectl get pods -n learning-dev

# See details
kubectl describe deployment nginx-deployment -n learning-dev

# Get detailed Pod info
kubectl get pods -n learning-dev -o wide

# See all replicas
kubectl get pods -n learning-dev -l app=nginx

# Scale up to 5 replicas
kubectl scale deployment nginx-deployment --replicas=5 -n learning-dev

# Watch scaling in real-time
kubectl get pods -n learning-dev -w

# Scale down to 2
kubectl scale deployment nginx-deployment --replicas=2 -n learning-dev
```

### Exercise 3.2: Rolling Update

```powershell
# Update image to newer version
kubectl set image deployment/nginx-deployment nginx=nginx:1.26 -n learning-dev

# Watch the update happen
kubectl get pods -n learning-dev -w

# Check rollout status
kubectl rollout status deployment/nginx-deployment -n learning-dev

# See rollout history
kubectl rollout history deployment/nginx-deployment -n learning-dev

# Rollback to previous version
kubectl rollout undo deployment/nginx-deployment -n learning-dev

# Check status after rollback
kubectl get pods -n learning-dev
```

### Key Takeaway
- Deployments **manage Pods automatically**
- Always use Deployments, not standalone Pods
- Easy to scale, update, and rollback
- Health checks keep Pods healthy

---

# Module 4: Services & Networking

## Service Types Explained

| Type | Use Case | Access |
|------|----------|--------|
| **ClusterIP** | Internal communication only | Only within cluster |
| **NodePort** | External access via node IP | `<NodeIP>:<NodePort>` |
| **LoadBalancer** | External access (cloud LB) | AWS ALB/NLB |
| **ExternalName** | Route to external service | External DNS |

### Exercise 4.1: ClusterIP Service (Internal)

**File: `service-clusterip.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: learning-dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: app
        image: httpbin:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  namespace: learning-dev
spec:
  type: ClusterIP        # Default - internal only
  selector:
    app: web             # Select Pods with this label
  ports:
  - protocol: TCP
    port: 8080           # Service port
    targetPort: 80       # Container port
```

**Test ClusterIP:**

```powershell
# Apply
kubectl apply -f service-clusterip.yaml

# Get Service
kubectl get svc -n learning-dev

# Get Service details (check Endpoints)
kubectl describe svc web-app-service -n learning-dev

# Test from within cluster (create a debug Pod)
kubectl run -it --rm debug --image=busybox:1.35 --restart=Never -- sh

# Inside the Pod shell:
wget -O- http://web-app-service:8080/get
exit

# Test with port-forward
kubectl port-forward svc/web-app-service 8080:8080 -n learning-dev
curl http://localhost:8080/get
```

### Exercise 4.2: NodePort Service (External Access)

**File: `service-nodeport.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-nodeport
  namespace: learning-dev
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - protocol: TCP
    port: 8080        # Cluster internal port
    targetPort: 80    # Pod container port
    nodePort: 30080   # Node external port (30000-32767)
```

**Test NodePort:**

```powershell
# Apply
kubectl apply -f service-nodeport.yaml

# Get NodePort service
kubectl get svc -n learning-dev

# Get Node IPs
kubectl get nodes -o wide

# Test from your machine (using Node IP)
curl http://<NODE-IP>:30080/get

# Get the actual command to test
kubectl get svc web-app-nodeport -n learning-dev -o jsonpath='{.spec.clusterIP}'
```

### Exercise 4.3: LoadBalancer Service (AWS ALB)

**File: `service-loadbalancer.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-lb
  namespace: learning-dev
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"  # Network LB
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

**Test LoadBalancer:**

```powershell
# Apply
kubectl apply -f service-loadbalancer.yaml

# Wait for AWS to provision the LB (takes 1-2 minutes)
kubectl get svc -n learning-dev -w

# Get the External IP (AWS LB endpoint)
kubectl get svc web-app-lb -n learning-dev

# Test using the External IP
curl http://<EXTERNAL-IP>/get
```

### Service Selector Deep Dive

```powershell
# All services in a namespace
kubectl get svc -n learning-dev

# See what Pods a service targets
kubectl get endpoints -n learning-dev

# Check specific endpoints
kubectl describe endpoints web-app-service -n learning-dev

# Get Pods with specific label
kubectl get pods -n learning-dev -l app=web

# Get Pods with multiple labels
kubectl get pods -n learning-dev -l app=web,version=v1
```

---

# Module 5: ConfigMaps & Secrets

## ConfigMaps: Non-Sensitive Configuration

**File: `configmap-example.yaml`**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: learning-dev
data:
  app.properties: |
    app.name=MyApp
    app.version=1.0.0
    log.level=INFO
    database.host=db.example.com
  app.env: production
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-config
  namespace: learning-dev
spec:
  replicas: 2
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
        image: nginx:1.25
        env:
        # Method 1: Single value from ConfigMap
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: app.env
        volumeMounts:
        # Method 2: Mount entire ConfigMap as file
        - name: config-volume
          mountPath: /etc/config
      volumes:
      - name: config-volume
        configMap:
          name: app-config
```

**Test ConfigMap:**

```powershell
# Create ConfigMap
kubectl apply -f configmap-example.yaml

# View ConfigMap
kubectl get configmap -n learning-dev
kubectl describe configmap app-config -n learning-dev

# Edit ConfigMap
kubectl edit configmap app-config -n learning-dev

# Check mounted files in Pod
kubectl exec <POD-NAME> -n learning-dev -- cat /etc/config/app.properties

# Create ConfigMap from file
echo "key1=value1" > config.txt
kubectl create configmap file-config --from-file=config.txt -n learning-dev
```

## Secrets: Sensitive Data

**File: `secret-example.yaml`**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: learning-dev
type: Opaque
stringData:  # Use stringData for plain text (easier to read)
  username: admin
  password: super-secret-password
  connection-string: "mongodb://admin:super-secret-password@mongo-db:27017/mydb"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-secrets
  namespace: learning-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-secret
  template:
    metadata:
      labels:
        app: app-secret
    spec:
      containers:
      - name: app
        image: nginx:1.25
        env:
        # Method 1: Single value from Secret
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        volumeMounts:
        # Method 2: Mount as files
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
      volumes:
      - name: secret-volume
        secret:
          secretName: db-credentials
```

**Test Secrets:**

```powershell
# Create Secret
kubectl apply -f secret-example.yaml

# View Secrets (values are base64 encoded)
kubectl get secret -n learning-dev
kubectl describe secret db-credentials -n learning-dev

# View secret value (base64 encoded)
kubectl get secret db-credentials -n learning-dev -o jsonpath='{.data.password}'

# Decode the value
kubectl get secret db-credentials -n learning-dev -o jsonpath='{.data.password}' | [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((Read-Host)))

# Create Secret from file
kubectl create secret generic my-secret --from-file=secret.txt -n learning-dev
```

### Important Security Note

⚠️ **Secrets are NOT encrypted by default in EKS!**

To enable encryption:
- Use AWS KMS for encryption
- Or use tools like Sealed Secrets, HashiCorp Vault

---

# Module 6: Persistent Storage

## PersistentVolume (PV) & PersistentVolumeClaim (PVC)

In EKS, use AWS EBS volumes.

**File: `persistent-storage.yaml`**

```yaml
# Step 1: Create PersistentVolumeClaim (request for storage)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  namespace: learning-dev
spec:
  accessModes:
    - ReadWriteOnce    # Can be mounted by one Pod at a time
  storageClassName: gp2  # AWS GP2 EBS volume
  resources:
    requests:
      storage: 5Gi     # Request 5GB
---
# Step 2: Use PVC in Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-storage
  namespace: learning-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: storage-app
  template:
    metadata:
      labels:
        app: storage-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        volumeMounts:
        - name: data-volume
          mountPath: /data        # Mount path in container
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: data-pvc     # Use the PVC we created
```

**Test Persistent Storage:**

```powershell
# Create PVC and Deployment
kubectl apply -f persistent-storage.yaml

# Check PVC status
kubectl get pvc -n learning-dev
kubectl describe pvc data-pvc -n learning-dev

# Check if PersistentVolume was created
kubectl get pv

# Write data to the mounted volume
kubectl exec <POD-NAME> -n learning-dev -- bash -c "echo 'Hello from EBS' > /data/test.txt"

# Verify data is written
kubectl exec <POD-NAME> -n learning-dev -- cat /data/test.txt

# Even if Pod is deleted and recreated, data persists
kubectl delete pod <POD-NAME> -n learning-dev

# New Pod will be created by Deployment
kubectl get pods -n learning-dev

# Data should still be there
kubectl exec <NEW-POD-NAME> -n learning-dev -- cat /data/test.txt
```

## StatefulSet for Databases

Use StatefulSet when:
- Pods need stable identities
- Data needs to persist
- Pods need to communicate with each other (e.g., databases)

**File: `statefulset-mongodb.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-headless
  namespace: learning-dev
spec:
  clusterIP: None              # Headless Service (no load balancing)
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: learning-dev
spec:
  serviceName: mongodb-headless  # Must use headless service
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:6.0
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: data-volume
          mountPath: /data/db
  # Persistent storage for each replica
  volumeClaimTemplates:
  - metadata:
      name: data-volume
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: gp2
      resources:
        requests:
          storage: 10Gi
```

**Test StatefulSet:**

```powershell
# Create StatefulSet
kubectl apply -f statefulset-mongodb.yaml

# Check Pods - they have stable names
kubectl get pods -n learning-dev

# Output: mongodb-0, mongodb-1, mongodb-2

# Scale (scales down in reverse order)
kubectl scale statefulset mongodb --replicas=5 -n learning-dev

# Check PVCs for each replica
kubectl get pvc -n learning-dev

# Connect to specific replica
kubectl exec -it mongodb-0 -n learning-dev -- mongosh
```

---

# Module 7: Advanced Controllers

## DaemonSet: Run Pod on Every Node

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitoring
  namespace: learning-dev
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      containers:
      - name: monitor
        image: busybox:1.35
        command: ['sh', '-c', 'while true; do echo "Running on $(hostname)"; sleep 30; done']
```

**Test:**

```powershell
# Create DaemonSet
kubectl apply -f daemonset-example.yaml

# Check Pods - one on each node
kubectl get pods -n learning-dev -o wide

# Should see 2 Pods (one per node)
```

## CronJob: Scheduled Tasks

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
  namespace: learning-dev
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: busybox:1.35
            command: ['sh', '-c', 'echo "Running backup at $(date)"']
          restartPolicy: OnFailure
```

**Test:**

```powershell
# Create CronJob
kubectl apply -f cronjob-example.yaml

# See CronJob
kubectl get cronjob -n learning-dev

# Check generated Jobs
kubectl get jobs -n learning-dev

# Check Job Pods
kubectl get pods -n learning-dev
```

---

# Module 8: Networking Deep Dive

## Kubernetes Network Model

```
Pod to Pod Communication:
┌──────────────────────────────┐
│         Node 1               │
│  ┌─────────┐   ┌─────────┐  │
│  │ Pod A   │   │ Pod B   │  │
│  │10.0.1.2 │   │10.0.1.3 │  │
│  └─────────┘   └─────────┘  │
│        │            │         │
│        └────────────┘         │
│      (Direct via CNI)         │
└──────────────────────────────┘
         │
         │ (Routed between nodes)
         │
┌──────────────────────────────┐
│         Node 2               │
│  ┌─────────┐   ┌─────────┐  │
│  │ Pod C   │   │ Pod D   │  │
│  │10.0.2.2 │   │10.0.2.3 │  │
│  └─────────┘   └─────────┘  │
└──────────────────────────────┘
```

### Exercise 8.1: Pod-to-Pod Communication

**File: `networking-pods.yaml`**

```yaml
# Backend Pod
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
  namespace: learning-networking
  labels:
    app: backend
spec:
  containers:
  - name: backend
    image: httpbin:latest
    ports:
    - containerPort: 80
---
# Frontend Pod
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  namespace: learning-networking
  labels:
    app: frontend
spec:
  containers:
  - name: frontend
    image: busybox:1.35
    command: ['sh', '-c', 'sleep 3600']
```

**Test communication:**

```powershell
# Create Pods
kubectl apply -f networking-pods.yaml

# Get Pod IPs
kubectl get pods -n learning-networking -o wide

# Execute command in frontend Pod to reach backend
kubectl exec -it frontend-pod -n learning-networking -- wget -O- http://<BACKEND-POD-IP>/get

# Pod-to-Pod communication works!
```

### Exercise 8.2: Service-Based Communication

**File: `networking-service.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-service
  namespace: learning-networking
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
    spec:
      containers:
      - name: api
        image: httpbin:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-api
  namespace: learning-networking
spec:
  type: ClusterIP
  selector:
    app: backend-api
  ports:
  - port: 8080
    targetPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: client-pod
  namespace: learning-networking
spec:
  containers:
  - name: client
    image: busybox:1.35
    command: ['sh', '-c', 'sleep 3600']
```

**Test service communication:**

```powershell
# Create resources
kubectl apply -f networking-service.yaml

# Get Service IP
kubectl get svc -n learning-networking

# From client Pod, access service by name
kubectl exec -it client-pod -n learning-networking -- wget -O- http://backend-api:8080/get

# Service resolves to multiple Pod IPs (load balanced)
# Try multiple times - different Pods respond
for ($i=1; $i -le 5; $i++) {
  kubectl exec client-pod -n learning-networking -- wget -O- http://backend-api:8080/uuid
}

# See how responses differ (different Pods)
```

### Exercise 8.3: Network Policies (Firewall Rules)

```yaml
# Allow traffic only from specific pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-only
  namespace: learning-networking
spec:
  podSelector:
    matchLabels:
      app: backend-api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
```

**Test NetworkPolicy:**

```powershell
# Create policy
kubectl apply -f networkpolicy-example.yaml

# Traffic from allowed Pod works
kubectl exec -it frontend-pod -- wget -O- http://backend-api:8080/get

# Traffic from other Pods is blocked
# (This depends on your CNI plugin supporting NetworkPolicy)
```

### DNS in Kubernetes

```powershell
# Pod DNS name format:
# <pod-name>.<namespace>.pod.cluster.local

# Service DNS name format:
# <service-name>.<namespace>.svc.cluster.local

# Test DNS resolution
kubectl exec client-pod -n learning-networking -- nslookup backend-api.learning-networking.svc.cluster.local

# Short form (same namespace):
kubectl exec client-pod -n learning-networking -- nslookup backend-api

# Check CoreDNS Pods (manages DNS)
kubectl get pods -n kube-system | grep coredns

# Check DNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns
```

---

# Module 9: Troubleshooting

## Essential Troubleshooting Commands

```powershell
# ===== Cluster Health =====
# Check cluster status
kubectl cluster-info

# Check nodes
kubectl get nodes
kubectl describe node <NODE-NAME>

# Check system Pods
kubectl get pods -n kube-system
kubectl logs -n kube-system -l component=kubelet

# ===== Pod Debugging =====
# Get Pod status
kubectl get pods -n learning-dev
kubectl describe pod <POD-NAME> -n learning-dev

# View Pod logs
kubectl logs <POD-NAME> -n learning-dev

# Logs for multi-container Pod
kubectl logs <POD-NAME> -c <CONTAINER-NAME> -n learning-dev

# Stream logs (follow)
kubectl logs <POD-NAME> -f -n learning-dev

# Previous logs (if Pod restarted)
kubectl logs <POD-NAME> --previous -n learning-dev

# ===== Pod Execution =====
# Execute command in Pod
kubectl exec <POD-NAME> -n learning-dev -- ls -la

# Interactive shell
kubectl exec -it <POD-NAME> -n learning-dev -- bash

# Multi-container Pod
kubectl exec -it <POD-NAME> -c <CONTAINER-NAME> -n learning-dev -- bash

# ===== Port Forwarding =====
# Forward local port to Pod
kubectl port-forward <POD-NAME> 8080:80 -n learning-dev

# Forward to Service
kubectl port-forward svc/<SERVICE-NAME> 8080:80 -n learning-dev

# ===== Event Debugging =====
# View events (shows errors)
kubectl get events -n learning-dev

# Watch events in real-time
kubectl get events -n learning-dev -w

# ===== Resource Usage =====
# Check Pod resource usage
kubectl top pods -n learning-dev

# Check node resource usage
kubectl top nodes

# ===== YAML Debugging =====
# Get resource YAML
kubectl get deployment nginx-deployment -n learning-dev -o yaml

# Get specific field
kubectl get pod <POD-NAME> -n learning-dev -o jsonpath='{.metadata.labels}'

# Edit resource directly
kubectl edit deployment nginx-deployment -n learning-dev
```

## Common Issues & Solutions

### Issue 1: Pod is Pending

```powershell
# Check why Pod is not scheduled
kubectl describe pod <POD-NAME> -n learning-dev

# Likely causes:
# - Node doesn't have enough resources
# - NodeSelector constraint not met
# - PVC not bound

# Solution:
kubectl get nodes
kubectl top nodes
```

### Issue 2: Pod is CrashLoopBackOff

```powershell
# Container is crashing repeatedly
kubectl describe pod <POD-NAME> -n learning-dev
kubectl logs <POD-NAME> -n learning-dev
kubectl logs <POD-NAME> --previous -n learning-dev

# Check application logs for errors
```

### Issue 3: Service has No Endpoints

```powershell
# Service created but no Pods connected
kubectl get endpoints -n learning-dev
kubectl describe svc <SERVICE-NAME> -n learning-dev

# Check if labels match
kubectl get pods -n learning-dev -l app=<LABEL-VALUE>

# Selector in service should match Pod labels
```

### Issue 4: Cannot Reach Service

```powershell
# Test from another Pod
kubectl run -it --rm debug --image=busybox:1.35 --restart=Never -- sh

# Inside Pod shell:
wget -O- http://<SERVICE-NAME>:8080

# If fails, check:
# - Service exists: kubectl get svc
# - Endpoints exist: kubectl get endpoints
# - Pod is running: kubectl get pods
# - Labels match: kubectl get pods -l <LABEL>
```

### Issue 5: Node Not Ready

```powershell
# Check node status
kubectl describe node <NODE-NAME>

# Check kubelet logs (in AWS SSM or CloudWatch)
# Usually issues:
# - Disk space full
# - Memory issues
# - Network plugin not working

# Drain and restart node (if using auto-scaling group):
kubectl drain <NODE-NAME> --ignore-daemonsets
# Terminate the node in AWS console (new one will be created)
```

---

# Complete Practice Project: Deploy Your ToDo App

Now let's apply everything you learned to deploy your actual ToDo API:

**File: `todo-app-complete.yaml`**

```yaml
---
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: todo-app
---
# ConfigMap for app configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: todo-config
  namespace: todo-app
data:
  LOG_LEVEL: INFO
  JAVA_OPTS: "-Xmx256m -Xms256m"
---
# Secret for MongoDB credentials
apiVersion: v1
kind: Secret
metadata:
  name: mongo-credentials
  namespace: todo-app
type: Opaque
stringData:
  username: admin
  password: securepassword123
---
# MongoDB StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: todo-app
spec:
  serviceName: mongodb
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:6.0
        ports:
        - containerPort: 27017
          name: mongo
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongo-credentials
              key: username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongo-credentials
              key: password
        volumeMounts:
        - name: mongo-data
          mountPath: /data/db
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          exec:
            command:
            - mongosh
            - --eval
            - "db.adminCommand('ping')"
          initialDelaySeconds: 30
          periodSeconds: 10
      securityContext:
        fsGroup: 999
  volumeClaimTemplates:
  - metadata:
      name: mongo-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: gp2
      resources:
        requests:
          storage: 10Gi
---
# MongoDB Service (Headless for StatefulSet)
apiVersion: v1
kind: Service
metadata:
  name: mongodb
  namespace: todo-app
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
---
# Todo API Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-api
  namespace: todo-app
  labels:
    app: todo-api
    version: v1
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: todo-api
  template:
    metadata:
      labels:
        app: todo-api
        version: v1
    spec:
      containers:
      - name: todo-api
        image: lishwar/springboot-api:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
          name: http
        env:
        # From ConfigMap
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: todo-config
              key: LOG_LEVEL
        # MongoDB connection
        - name: SPRING_DATA_MONGODB_HOST
          value: mongodb.todo-app.svc.cluster.local
        - name: SPRING_DATA_MONGODB_PORT
          value: "27017"
        - name: SPRING_DATA_MONGODB_DATABASE
          value: todo_db
        - name: SPRING_DATA_MONGODB_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongo-credentials
              key: username
        - name: SPRING_DATA_MONGODB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongo-credentials
              key: password
        - name: SPRING_DATA_MONGODB_AUTHENTICATION_DATABASE
          value: admin
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 5000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 5000
          initialDelaySeconds: 15
          periodSeconds: 5
          timeoutSeconds: 3
        
        volumeMounts:
        - name: config-volume
          mountPath: /config
          readOnly: true
      
      volumes:
      - name: config-volume
        configMap:
          name: todo-config
      
      affinity:
        # Spread Pods across nodes
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - todo-api
              topologyKey: kubernetes.io/hostname
---
# ClusterIP Service (Internal)
apiVersion: v1
kind: Service
metadata:
  name: todo-api-internal
  namespace: todo-app
  labels:
    app: todo-api
spec:
  type: ClusterIP
  selector:
    app: todo-api
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
    name: http
---
# LoadBalancer Service (External - AWS ALB)
apiVersion: v1
kind: Service
metadata:
  name: todo-api-lb
  namespace: todo-app
  labels:
    app: todo-api
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: todo-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000
    name: http
```

**Deploy and test:**

```powershell
# Apply all resources
kubectl apply -f todo-app-complete.yaml

# Check everything is running
kubectl get all -n todo-app

# Watch deployment
kubectl get pods -n todo-app -w

# Check MongoDB is running
kubectl get statefulset -n todo-app
kubectl describe statefulset mongodb -n todo-app

# Wait for LoadBalancer to get external IP (1-2 minutes)
kubectl get svc -n todo-app -w

# Get the external IP
kubectl get svc todo-api-lb -n todo-app

# Test the API
curl http://<EXTERNAL-IP>/todos

# View logs
kubectl logs -n todo-app -l app=todo-api -f

# Check MongoDB connection
kubectl exec -it mongodb-0 -n todo-app -- mongosh -u admin -p securepassword123 --authenticationDatabase admin
> show databases
> use todo_db
> show collections
> exit
```

---

# Learning Path Summary

## Week 1: Fundamentals
- [ ] Module 1: Kubernetes Architecture
- [ ] Module 2: Pods
- [ ] Module 3: Deployments
- [ ] Small project: Deploy 2-3 replicas of nginx

## Week 2: Services & Networking
- [ ] Module 4: Services (ClusterIP, NodePort, LoadBalancer)
- [ ] Module 8: Networking Deep Dive
- [ ] Project: Deploy app with internal and external access

## Week 3: Configuration & Storage
- [ ] Module 5: ConfigMaps & Secrets
- [ ] Module 6: Persistent Storage
- [ ] Project: Deploy app with environment variables and data persistence

## Week 4: Advanced Concepts
- [ ] Module 7: Advanced Controllers (DaemonSet, CronJob, StatefulSet)
- [ ] Module 9: Troubleshooting
- [ ] Complete project: Deploy full ToDo app with DB

## Ongoing Practice
- [ ] Deploy different applications
- [ ] Practice troubleshooting
- [ ] Explore helm charts
- [ ] Study Ingress resources
- [ ] Learn about RBAC (Role-Based Access Control)

---

# Additional Resources

## Useful Commands Reference

```powershell
# General
kubectl get <resource> [-n <namespace>] [-o <format>] [-w]
kubectl describe <resource> <name> [-n <namespace>]
kubectl apply -f <file.yaml>
kubectl delete <resource> <name> [-n <namespace>]
kubectl edit <resource> <name> [-n <namespace>]

# Debugging
kubectl logs <pod> [-c <container>] [--previous] [-f]
kubectl exec -it <pod> [-c <container>] -- <command>
kubectl port-forward <pod|svc/name> <local>:<remote> [-n <namespace>]

# Scaling
kubectl scale <resource> <name> --replicas=<count> [-n <namespace>]

# Updates
kubectl set image <resource>/<name> <container>=<image> [-n <namespace>]
kubectl rollout status <resource>/<name> [-n <namespace>]
kubectl rollout history <resource>/<name> [-n <namespace>]
kubectl rollout undo <resource>/<name> [-n <namespace>]

# Labels & Selectors
kubectl get <resource> -l <key>=<value> [-n <namespace>]
kubectl label <resource> <name> <key>=<value> [-n <namespace>]
```

---


