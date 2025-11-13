# ArgoCD Deployment Tutorial - Complete Guide

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Part 1: Application Development](#part-1-application-development)
4. [Part 2: Containerization](#part-2-containerization)
5. [Part 3: Kubernetes Manifests](#part-3-kubernetes-manifests)
6. [Part 4: ArgoCD Installation](#part-4-argocd-installation)
7. [Part 5: GitOps Deployment](#part-5-gitops-deployment)
8. [Part 6: Verification & Testing](#part-6-verification--testing)
9. [Part 7: Making Changes (GitOps Workflow)](#part-7-making-changes-gitops-workflow)
10. [Troubleshooting](#troubleshooting)

---

## Overview

This tutorial covers deploying a complete application stack using ArgoCD and GitOps principles:

- **Application**: Python Flask REST API with visitor counter
- **Database**: PostgreSQL 15
- **Container Registry**: Docker Hub
- **Orchestration**: Kubernetes (Civo)
- **CD Tool**: ArgoCD
- **Scaling**: Horizontal Pod Autoscaler

**What You'll Learn:**
- Creating a containerized application with database integration
- Building and pushing Docker images
- Writing Kubernetes manifests
- Installing and configuring ArgoCD
- Implementing GitOps workflows
- Auto-scaling with HPA

---

## Prerequisites

**Required Tools:**
- `kubectl` - Kubernetes CLI
- `docker` - Container runtime
- `git` - Version control
- Access to a Kubernetes cluster (Civo)
- Docker Hub account
- GitHub/GitLab account

**Verify Prerequisites:**
```bash
# Check kubectl connection
kubectl cluster-info
kubectl get nodes

# Check docker
docker --version
docker login

# Check git
git --version
```

---

## Part 1: Application Development

### 1.1 Flask Application (`app.py`)

Create a Python Flask application with PostgreSQL integration:

```python
from flask import Flask, jsonify
import psycopg2
import os
from datetime import datetime

app = Flask(__name__)

# Database configuration from environment variables
DB_HOST = os.getenv('DB_HOST', 'localhost')
DB_PORT = os.getenv('DB_PORT', '5432')
DB_NAME = os.getenv('DB_NAME', 'visitordb')
DB_USER = os.getenv('DB_USER', 'postgres')
DB_PASSWORD = os.getenv('DB_PASSWORD', 'postgres')

def get_db_connection():
    """Create database connection"""
    conn = psycopg2.connect(
        host=DB_HOST,
        port=DB_PORT,
        database=DB_NAME,
        user=DB_USER,
        password=DB_PASSWORD
    )
    return conn

def init_db():
    """Initialize database table"""
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute('''
            CREATE TABLE IF NOT EXISTS visitors (
                id SERIAL PRIMARY KEY,
                visit_time TIMESTAMP NOT NULL,
                ip_address VARCHAR(50)
            )
        ''')
        conn.commit()
        cur.close()
        conn.close()
        print("Database initialized successfully")
    except Exception as e:
        print(f"Error initializing database: {e}")

@app.route('/')
def home():
    """Home endpoint"""
    return jsonify({
        "message": "Welcome to the Visitor Counter API",
        "endpoints": {
            "/": "This help message",
            "/visit": "Record a visit",
            "/count": "Get total visit count",
            "/health": "Health check"
        }
    })

@app.route('/visit', methods=['POST'])
def record_visit():
    """Record a new visit"""
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute(
            'INSERT INTO visitors (visit_time, ip_address) VALUES (%s, %s)',
            (datetime.now(), 'unknown')
        )
        conn.commit()
        cur.close()
        conn.close()
        return jsonify({"status": "success", "message": "Visit recorded"}), 201
    except Exception as e:
        return jsonify({"status": "error", "message": str(e)}), 500

@app.route('/count')
def get_count():
    """Get total visitor count"""
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute('SELECT COUNT(*) FROM visitors')
        count = cur.fetchone()[0]
        cur.close()
        conn.close()
        return jsonify({"total_visits": count})
    except Exception as e:
        return jsonify({"status": "error", "message": str(e)}), 500

@app.route('/health')
def health():
    """Health check endpoint"""
    try:
        conn = get_db_connection()
        conn.close()
        return jsonify({"status": "healthy", "database": "connected"}), 200
    except Exception as e:
        return jsonify({"status": "unhealthy", "database": str(e)}), 500

if __name__ == '__main__':
    init_db()
    app.run(host='0.0.0.0', port=5000, debug=True)
```

### 1.2 Python Dependencies (`requirements.txt`)

```txt
Flask==3.0.0
psycopg2-binary==2.9.9
gunicorn==21.2.0
```

---

## Part 2: Containerization

### 2.1 Dockerfile

```dockerfile
# Use Python slim image
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app.py .

# Expose port
EXPOSE 5000

# Run with gunicorn for production
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "app:app"]
```

### 2.2 Docker Compose (Local Testing)

```yaml
version: '3.8'

services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: visitordb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    build: .
    ports:
      - "5000:5000"
    environment:
      DB_HOST: db
      DB_PORT: 5432
      DB_NAME: visitordb
      DB_USER: postgres
      DB_PASSWORD: postgres
    depends_on:
      db:
        condition: service_healthy

volumes:
  postgres_data:
```

### 2.3 Build and Push Commands

```bash
# Build the Docker image
docker build -t your-dockerhub-username/visitor-counter:v1.0.0 .

# Test locally (optional)
docker-compose up

# Login to Docker Hub
docker login

# Push to Docker Hub
docker push your-dockerhub-username/visitor-counter:v1.0.0
```

---

## Part 3: Kubernetes Manifests

### 3.1 Project Structure

Create this directory structure:

```
visitor-counter-gitops/
├── k8s/
│   ├── 01-namespace.yaml
│   ├── 02-secret.yaml
│   ├── 03-postgres.yaml
│   ├── 04-deployment.yaml
│   ├── 05-service.yaml
│   └── 06-hpa.yaml
└── argocd-application.yaml
```

### 3.2 Namespace (`k8s/01-namespace.yaml`)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: visitor-app
```

### 3.3 Database Secret (`k8s/02-secret.yaml`)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: visitor-app
type: Opaque
stringData:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: MySecurePassword123
  POSTGRES_DB: visitordb
```

### 3.4 PostgreSQL StatefulSet (`k8s/03-postgres.yaml`)

```yaml
---
# PostgreSQL Service
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: visitor-app
spec:
  ports:
  - port: 5432
    targetPort: 5432
  clusterIP: None
  selector:
    app: postgres
---
# PostgreSQL StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: visitor-app
spec:
  serviceName: postgres
  replicas: 1
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
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_PASSWORD
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_DB
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 5
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

### 3.5 Application Deployment (`k8s/04-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: visitor-counter
  namespace: visitor-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: visitor-counter
  template:
    metadata:
      labels:
        app: visitor-counter
    spec:
      # Wait for database to be ready before starting app
      initContainers:
      - name: wait-for-db
        image: postgres:15-alpine
        command:
        - sh
        - -c
        - |
          until pg_isready -h postgres -U postgres; do
            echo "Waiting for PostgreSQL to be ready..."
            sleep 2
          done
          echo "PostgreSQL is ready!"
      
      containers:
      - name: app
        image: curtis2025/visitor-counter:v1.0.0
        ports:
        - containerPort: 5000
        env:
        - name: DB_HOST
          value: "postgres"
        - name: DB_PORT
          value: "5432"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_PASSWORD
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_DB
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 30
          periodSeconds: 5
          timeoutSeconds: 3
```

### 3.6 Application Service (`k8s/05-service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: visitor-counter
  namespace: visitor-app
spec:
  type: LoadBalancer
  selector:
    app: visitor-counter
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000
```

### 3.7 Horizontal Pod Autoscaler (`k8s/06-hpa.yaml`)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: visitor-counter-hpa
  namespace: visitor-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: visitor-counter
  minReplicas: 3
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

### 3.8 ArgoCD Application (`argocd-application.yaml`)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: visitor-counter
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/your-username/visitor-counter-gitops.git
    targetRevision: main
    path: k8s
  
  destination:
    server: https://kubernetes.default.svc
    namespace: visitor-app
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

---

## Part 4: ArgoCD Installation

### 4.1 Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl get pods -n argocd -w
```

### 4.2 Access ArgoCD UI

**Option A: Port Forward (Quick)**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Access at: https://localhost:8080
```

**Option B: LoadBalancer (For Civo)**
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc argocd-server -n argocd
# Use the EXTERNAL-IP
```

### 4.3 Get Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

**Login Credentials:**
- Username: `admin`
- Password: (output from above command)

### 4.4 Install ArgoCD CLI (Optional)

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

### 4.5 Login via CLI

```bash
# If using port-forward
argocd login localhost:8080

# If using LoadBalancer
argocd login <EXTERNAL-IP>
```

### 4.6 Change Admin Password

```bash
argocd account update-password
# Password must be 8-32 characters
```

---

## Part 5: GitOps Deployment

### 5.1 Create Git Repository

```bash
# Create directory
mkdir visitor-counter-gitops
cd visitor-counter-gitops

# Create k8s directory
mkdir k8s

# Add all YAML files to k8s/ directory
# (01-namespace.yaml, 02-secret.yaml, etc.)

# Initialize Git
git init
git add .
git commit -m "Initial commit: visitor counter app"

# Create GitHub repository and push
git branch -M main
git remote add origin https://github.com/your-username/visitor-counter-gitops.git
git push -u origin main
```

### 5.2 Deploy Application with ArgoCD

**Method 1: Using ArgoCD UI**

1. Open ArgoCD UI
2. Click **+ NEW APP**
3. Fill in:
   - **Application Name**: `visitor-counter`
   - **Project**: `default`
   - **Sync Policy**: `Automatic` (enable auto-sync, prune, self-heal)
   - **Repository URL**: `https://github.com/your-username/visitor-counter-gitops.git`
   - **Revision**: `main`
   - **Path**: `k8s`
   - **Cluster URL**: `https://kubernetes.default.svc`
   - **Namespace**: `visitor-app`
4. Click **CREATE**

**Method 2: Using ArgoCD CLI**

```bash
argocd app create visitor-counter \
  --repo https://github.com/your-username/visitor-counter-gitops.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace visitor-app \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

**Method 3: Using kubectl**

```bash
# Edit argocd-application.yaml with your repo URL
kubectl apply -f argocd-application.yaml
```

---

## Part 6: Verification & Testing

### 6.1 Check ArgoCD Application

```bash
# Get application status
argocd app get visitor-counter

# Watch sync progress
argocd app sync visitor-counter --watch
```

### 6.2 Verify Kubernetes Resources

```bash
# Check all resources
kubectl get all -n visitor-app

# Check pods
kubectl get pods -n visitor-app

# Check services
kubectl get svc -n visitor-app

# Check HPA
kubectl get hpa -n visitor-app

# Check PVC
kubectl get pvc -n visitor-app
```

### 6.3 Check Pod Logs

```bash
# PostgreSQL logs
kubectl logs -n visitor-app postgres-0

# Application logs
kubectl logs -n visitor-app -l app=visitor-counter

# Follow logs
kubectl logs -n visitor-app -l app=visitor-counter -f
```

### 6.4 Get LoadBalancer IP

```bash
# Get external IP
kubectl get svc visitor-counter -n visitor-app

# Wait for EXTERNAL-IP to be assigned
# Store in variable
EXTERNAL_IP=$(kubectl get svc visitor-counter -n visitor-app -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $EXTERNAL_IP
```

### 6.5 Test Application Endpoints

```bash
# Home endpoint
curl http://$EXTERNAL_IP/

# Health check
curl http://$EXTERNAL_IP/health

# Record a visit
curl -X POST http://$EXTERNAL_IP/visit

# Get visit count
curl http://$EXTERNAL_IP/count
```

---

## Part 7: Making Changes (GitOps Workflow)

### 7.1 Scale Application

Edit `k8s/04-deployment.yaml`:

```yaml
spec:
  replicas: 5  # Changed from 3 to 5
```

Commit and push:
```bash
git add k8s/04-deployment.yaml
git commit -m "Scale app to 5 replicas"
git push
```

### 7.2 Watch Auto-Sync

```bash
# ArgoCD will detect changes automatically
argocd app watch visitor-counter

# Check sync status
argocd app get visitor-counter

# Verify new pods
kubectl get pods -n visitor-app -l app=visitor-counter
```

### 7.3 Update Application Image

Edit `k8s/04-deployment.yaml`:

```yaml
containers:
- name: app
  image: curtis2025/visitor-counter:v2.0.0  # New version
```

Commit and push:
```bash
git add k8s/04-deployment.yaml
git commit -m "Update to v2.0.0"
git push
```

### 7.4 Rollback (Revert in Git)

```bash
# Revert last commit
git revert HEAD

# Or go back to specific commit
git revert <commit-hash>

# Push revert
git push

# ArgoCD will automatically sync the rollback
```

### 7.5 Test Autoscaling

Generate load to test HPA:

```bash
# Install Apache Bench
# macOS: brew install httpd
# Ubuntu: sudo apt-get install apache2-utils

# Generate load
ab -n 10000 -c 100 http://$EXTERNAL_IP/count

# Watch HPA scale
kubectl get hpa -n visitor-app -w

# Watch pods scale up
kubectl get pods -n visitor-app -w
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. PostgreSQL CrashLoopBackOff

**Symptoms:**
```
postgres-0   0/1   CrashLoopBackOff
```

**Solution:**
```bash
# Delete StatefulSet and PVC to start fresh
kubectl delete statefulset postgres -n visitor-app
kubectl delete pvc postgres-storage-postgres-0 -n visitor-app

# ArgoCD will recreate them
```

#### 2. Application Health Check Failing

**Symptoms:**
```
Liveness probe failed: HTTP probe failed with statuscode: 500
```

**Check:**
```bash
# Check app logs
kubectl logs -n visitor-app -l app=visitor-counter

# Check if database is ready
kubectl get pods -n visitor-app postgres-0

# Verify init container ran
kubectl describe pod -n visitor-app <app-pod-name>
```

**Solution:**
- Ensure PostgreSQL is running
- Check database credentials in secret
- Verify init container completed successfully

#### 3. ArgoCD Out of Sync

**Check sync status:**
```bash
argocd app get visitor-counter
```

**Manual sync:**
```bash
argocd app sync visitor-counter
```

**Force sync:**
```bash
argocd app sync visitor-counter --force
```

#### 4. LoadBalancer Pending

**Check:**
```bash
kubectl get svc visitor-counter -n visitor-app
```

**If stuck in Pending:**
- Verify Civo has available load balancers
- Check cluster quota
- Try NodePort instead:
  ```yaml
  spec:
    type: NodePort
  ```

#### 5. Secret Not Found

**Verify secret exists:**
```bash
kubectl get secret postgres-secret -n visitor-app

# View secret (base64 encoded)
kubectl get secret postgres-secret -n visitor-app -o yaml

# Decode values
kubectl get secret postgres-secret -n visitor-app -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d
```

### Useful Debug Commands

```bash
# Get pod details
kubectl describe pod <pod-name> -n visitor-app

# Get events
kubectl get events -n visitor-app --sort-by='.lastTimestamp'

# Exec into pod
kubectl exec -it <pod-name> -n visitor-app -- /bin/sh

# Test database connection from app pod
kubectl exec -it <app-pod> -n visitor-app -- psql -h postgres -U postgres -d visitordb

# Port forward to database
kubectl port-forward -n visitor-app svc/postgres 5432:5432

# Delete and recreate everything
kubectl delete namespace visitor-app
argocd app sync visitor-counter
```

### ArgoCD Specific Commands

```bash
# List all applications
argocd app list

# Get application details
argocd app get visitor-counter

# View application logs
argocd app logs visitor-counter

# Delete application
argocd app delete visitor-counter

# Refresh application (check for changes)
argocd app refresh visitor-counter

# View diff
argocd app diff visitor-counter

# View history
argocd app history visitor-counter

# Rollback to previous version
argocd app rollback visitor-counter <revision-id>
```

---

## Key Concepts Summary

### GitOps Principles

1. **Declarative**: Entire system described declaratively in Git
2. **Versioned**: Git as single source of truth with version history
3. **Immutable**: No manual changes to cluster
4. **Automated**: Changes applied automatically via ArgoCD
5. **Auditable**: Complete audit trail in Git commits

### ArgoCD Features Used

- **Automated Sync**: Automatically deploys changes from Git
- **Self-Healing**: Reverts manual changes to cluster
- **Prune**: Removes resources deleted from Git
- **Health Assessment**: Monitors application health
- **Rollback**: Easy rollback via Git revert

### Kubernetes Resources

- **Namespace**: Isolation boundary for resources
- **Secret**: Sensitive data (database credentials)
- **StatefulSet**: Stateful applications (PostgreSQL)
- **Deployment**: Stateless applications (Flask app)
- **Service**: Network access to pods
- **HPA**: Automatic horizontal scaling
- **PVC**: Persistent storage for database

---

## Next Steps

### Enhancements to Explore

1. **Add Monitoring**
   - Prometheus for metrics
   - Grafana for dashboards
   - Alert rules

2. **Implement CI/CD**
   - GitHub Actions for building images
   - Automated image tag updates
   - Image scanning

3. **Add Ingress**
   - Nginx Ingress Controller
   - TLS certificates (cert-manager)
   - Custom domain

4. **Multiple Environments**
   - Separate folders: `k8s/dev`, `k8s/staging`, `k8s/prod`
   - Environment-specific values
   - Promotion workflows

5. **Advanced Deployments**
   - Blue-Green deployments
   - Canary releases
   - Progressive delivery with Argo Rollouts

6. **Security**
   - Sealed Secrets for encrypted secrets in Git
   - Pod Security Standards
   - Network Policies
   - RBAC policies

7. **Observability**
   - Distributed tracing (Jaeger)
   - Centralized logging (ELK/Loki)
   - Application metrics

---

## Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [GitOps Principles](https://opengitops.dev/)
- [Flask Documentation](https://flask.palletsprojects.com/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

---

## Summary

**What We Built:**
- ✅ Flask REST API with PostgreSQL database
- ✅ Docker containerization
- ✅ Complete Kubernetes manifests
- ✅ ArgoCD GitOps deployment
- ✅ Automatic scaling with HPA
- ✅ Self-healing infrastructure

**Key Achievements:**
- Deployed a production-ready application stack
- Implemented GitOps workflow
- Configured auto-sync and self-healing
- Set up database with persistent storage
- Configured health checks and probes
- Enabled horizontal autoscaling

**GitOps Benefits:**
- All changes tracked in Git
- Automated deployments
- Easy rollbacks
- Audit trail
- Consistent environments
- No manual kubectl commands needed

---

*Tutorial completed! You now have a fully functional GitOps deployment pipeline using ArgoCD.*
