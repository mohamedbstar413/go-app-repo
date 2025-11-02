# 🚀 Kubernetes CI/CD Pipeline with Jenkins

A complete CI/CD solution for deploying Go applications to Kubernetes using Jenkins, Helm, and Kaniko.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Infrastructure Components](#infrastructure-components)
- [Pipeline Stages](#pipeline-stages)
- [Setup Instructions](#setup-instructions)
- [Configuration](#configuration)
- [Security](#security)
- [Troubleshooting](#troubleshooting)

---

## 🎯 Overview

This project implements a production-ready CI/CD pipeline that automates the build, test, and deployment of a Go application to Kubernetes. The pipeline leverages Jenkins for orchestration, Kaniko for Docker image building, and Helm for Kubernetes deployments.

### Key Features

✅ **Automated Docker Builds** - Kaniko-based containerless builds  
✅ **Helm Chart Deployment** - Automated chart creation and deployment  
✅ **Smoke Testing** - Post-deployment health checks  
✅ **Email Notifications** - Build status notifications via Gmail  
✅ **Kubernetes Native** - Jenkins agents run as Kubernetes pods  
✅ **Secure Secrets Management** - Kubernetes Secrets for sensitive data  

---

## 🏗️ Architecture

```
┌─────────────────┐
│   GitHub Repo   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Jenkins Agent  │◄──── Kubernetes Pod
│   (Dynamic)     │
└────────┬────────┘
         │
    ┌────┴────┬─────────┬──────────┐
    ▼         ▼         ▼          ▼
┌───────┐ ┌───────┐ ┌───────┐ ┌────────┐
│  Git  │ │Kaniko │ │ Helm  │ │Tester  │
└───┬───┘ └───┬───┘ └───┬───┘ └───┬────┘
    │         │         │         │
    └─────────┴─────────┴─────────┘
                 │
                 ▼
         ┌───────────────┐
         │  Kubernetes   │
         │   Cluster     │
         └───────────────┘
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
┌────────┐ ┌──────┐ ┌────────┐
│ Nginx  │ │MySQL │ │Backend │
│ Proxy  │ │  DB  │ │  App   │
└────────┘ └──────┘ └────────┘
```

---

## 📦 Prerequisites

### Required Tools

- **Kubernetes Cluster** (v1.20+)
- **Jenkins** (v2.300+) with Kubernetes plugin
- **Helm** (v3.0+)
- **Docker Hub Account**
- **Gmail Account** (for notifications)

### Required Credentials

Configure the following credentials in Jenkins:

| Credential ID | Type | Usage |
|--------------|------|-------|
| `dockerhub` | Username/Password | Docker Hub authentication |
| `gmail` | Username/Password | Email notifications |

---

## 📁 Project Structure

```
.
├── k8s/
│   ├── jenkins-pv.yaml           # Persistent Volume
│   ├── jenkins-pvc.yaml          # Persistent Volume Claim
│   ├── nginx-proxy.yaml          # Nginx reverse proxy
│   ├── nginx-svc.yaml            # Nginx service
│   ├── nginx-tls-secret.yaml     # TLS certificates
│   ├── back-deploy.yaml          # Backend deployment
│   ├── back-svc.yaml             # Backend service
│   ├── db-ss.yaml                # MySQL StatefulSet
│   ├── db-svc.yaml               # MySQL service
│   ├── db-config.yaml            # Database ConfigMap
│   ├── db-secret.yaml            # Database Secret
│   ├── jenkins-rbac.yaml         # Jenkins RBAC
│   └── jenkins-clusterrole.yaml  # Jenkins ClusterRole
│
├── helm_test/                     # Helm chart template
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       └── serviceaccount.yaml
│
└── Jenkinsfile                    # Pipeline definition
```

---

## 🔧 Infrastructure Components

### 1. **Jenkins Persistent Storage**

```yaml
Storage: 10Gi
Access Mode: ReadWriteOnce
Path: /mnt/jenkins-data
```

### 2. **Nginx Reverse Proxy**

- **Image**: `nginx:latest`
- **Port**: 8080 → 443 (HTTPS)
- **TLS**: Configured with self-signed certificates
- **Purpose**: SSL termination and routing

### 3. **Backend Application**

- **Image**: `bstar999/senior-go`
- **Replicas**: 3
- **Port**: 8000
- **Secrets**: Database password mounted at `/run/secrets/db-password`

### 4. **MySQL Database**

- **Image**: `mysql:8.0`
- **Type**: StatefulSet
- **Port**: 3306
- **Storage**: HostPath volume at `/mysql-data`
- **Credentials**: Base64 encoded (root password: `admin123`)

---

## 🔄 Pipeline Stages

### Stage 1: **Checkout**
```groovy
Container: alpine/git
Action: Clone repository from GitHub
```

### Stage 2: **Build & Push Docker Image**
```groovy
Container: kaniko
Action: Build Docker image and push to Docker Hub
Tag: Build number (e.g., bstar999/senior-go:42)
```

### Stage 3: **Create Helm Chart**
```groovy
Container: helm-kubectl
Action: Generate Helm chart and copy K8s manifests
```

### Stage 4: **Deploy Helm Chart**
```groovy
Container: helm-kubectl
Action: Deploy/upgrade application to Kubernetes
Namespace: default
```

### Stage 5: **Smoke Test**
```groovy
Container: ubuntu
Action: Verify HTTPS endpoint availability
URL: https://nginx-svc.default.svc.cluster.local
```

### Stage 6: **Send Email Notification**
```groovy
Container: alpine
Action: Send build status email via Gmail SMTP
```

---

## 🚀 Setup Instructions

### 1. **Deploy Infrastructure**

```bash
# Create namespace
kubectl create namespace jenkins

# Deploy Jenkins RBAC
kubectl apply -f k8s/jenkins-clusterrole.yaml
kubectl apply -f k8s/jenkins-rbac.yaml

# Deploy storage
kubectl apply -f k8s/jenkins-pv.yaml
kubectl apply -f k8s/jenkins-pvc.yaml

# Deploy application components
kubectl apply -f k8s/db-secret.yaml
kubectl apply -f k8s/db-config.yaml
kubectl apply -f k8s/db-ss.yaml
kubectl apply -f k8s/db-svc.yaml
kubectl apply -f k8s/back-deploy.yaml
kubectl apply -f k8s/back-svc.yaml

# Deploy Nginx proxy
kubectl apply -f k8s/nginx-tls-secret.yaml
kubectl apply -f k8s/nginx-proxy.yaml
kubectl apply -f k8s/nginx-svc.yaml
```

### 2. **Configure Jenkins**

1. Install required plugins:
   - Kubernetes Plugin
   - Pipeline Plugin
   - Git Plugin
   - Credentials Binding Plugin

2. Add credentials:
   ```
   Manage Jenkins → Credentials → System → Global credentials
   ```

3. Create pipeline job:
   ```
   New Item → Pipeline → Pipeline script from SCM
   ```

### 3. **Run Pipeline**

```bash
# Trigger build
Build Now

# Monitor logs
Jenkins → Job → Console Output
```

---

## ⚙️ Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `GIT_REPO` | GitHub repository URL | `https://github.com/mohamedbstar413/go-app-repo.git` |
| `GIT_BRANCH` | Branch to build | `main` |
| `DOCKER_IMAGE` | Docker image name | `bstar999/senior-go` |
| `DOCKER_TAG` | Image tag | `${BUILD_NUMBER}` |
| `HELM_RELEASE_NAME` | Helm release name | `senior-go-app` |
| `K8S_NAMESPACE` | Target namespace | `default` |
| `SMOKE_TEST_URL` | Health check URL | `https://nginx-svc.default.svc.cluster.local` |

### Customizing Helm Values

Edit `helm_test/values.yaml`:

```yaml
replicaCount: 3
image:
  repository: bstar999/senior-go
  tag: "latest"
service:
  type: ClusterIP
  port: 80
```

---

## 🔐 Security

### Secrets Management

**Database Secret** (Base64 encoded):
```bash
echo -n "admin123" | base64
# Output: YWRtaW4xMjM=
```

**TLS Certificates**:
- Self-signed certificates for development
- For production, use cert-manager or valid CA certificates

### RBAC Configuration

Jenkins service account has cluster-wide permissions:
```yaml
apiGroups: ["*"]
resources: ["*"]
verbs: ["*"]
```

⚠️ **Production Recommendation**: Restrict permissions to specific namespaces and resources.

---

## 🐛 Troubleshooting

### Common Issues

**1. Kaniko Build Fails**
```bash
# Check Docker Hub credentials
kubectl get secret dockerhub -n jenkins -o yaml

# Verify Kaniko logs
kubectl logs -n jenkins <kaniko-pod>
```

**2. Helm Deployment Fails**
```bash
# Check Helm release status
helm list -n default

# Debug deployment
kubectl describe deployment senior-go-app -n default
```

**3. Smoke Test Fails**
```bash
# Verify Nginx service
kubectl get svc nginx-svc -n default

# Check certificate
kubectl get secret nginx-tls -n default

# Test connectivity
kubectl run test --rm -it --image=busybox -- wget -O- https://nginx-svc.default.svc.cluster.local
```

**4. Email Notification Fails**
```bash
# Verify Gmail app password (not regular password)
# Enable "Less secure app access" or use App Password
```

### Logs and Debugging

```bash
# Jenkins agent pod logs
kubectl logs -n jenkins -l app=jenkins-agent

# Application logs
kubectl logs -n default -l app=back

# Database logs
kubectl logs -n default -l app=db
```

---

## 📊 Monitoring

### Check Deployment Status

```bash
# Pods
kubectl get pods -n default

# Services
kubectl get svc -n default

# Helm releases
helm list -n default

# Persistent volumes
kubectl get pv,pvc
```

### Access Application

```bash
# Port forward to Nginx
kubectl port-forward svc/nginx-svc 8443:443 -n default

# Access in browser
https://localhost:8443
```

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## 📝 License

This project is licensed under the MIT License.

---

## 👤 Author

**Mohamed Abdelsattar**
- GitHub: [@mohamedbstar413](https://github.com/mohamedbstar413)
- Email: mabdelsattar413@gmail.com

---



---

**Built with ❤️ using Kubernetes, Jenkins, and Helm**
