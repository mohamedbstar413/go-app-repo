# 🚀 Kubernetes Full-Stack Application

A production-ready full-stack application deployed on Kubernetes, featuring a Go backend, MySQL database, and Nginx reverse proxy with TLS termination.

## 📋 Table of Contents

- [Architecture](#architecture)
- [Components](#components)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Security](#security)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)

## 🏗️ Architecture

This application follows a microservices architecture with the following components:

```
┌─────────────────┐
│  Nginx Proxy    │ (TLS Termination, Port 8080)
│  (nginx-proxy)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Go Backend    │ (3 Replicas, Port 8000)
│  (back-deploy)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  MySQL Database │ (StatefulSet, Port 3306)
│    (db-ss)      │
└─────────────────┘
```

## 🧩 Components

### Backend Service
- **Image**: `bstar999/senior-go`
- **Replicas**: 3 (for high availability)
- **Port**: 8000
- **Features**: 
  - Horizontal scaling
  - Secret-based database authentication
  - Health checks ready

### Database
- **Engine**: MySQL 8.0
- **Type**: StatefulSet (persistent storage)
- **Port**: 3306
- **Storage**: HostPath volume at `/mysql-data`
- **Features**:
  - Persistent data storage
  - ConfigMap-based configuration
  - Secret-based authentication

### Nginx Reverse Proxy
- **Image**: nginx
- **Port**: 8080
- **Features**:
  - TLS/SSL termination
  - Custom configuration support
  - Self-signed certificates included

## 📦 Prerequisites

- Kubernetes cluster (v1.19+)
- kubectl configured and connected to your cluster
- Sufficient cluster resources:
  - 3+ CPUs
  - 4GB+ RAM
  - Storage for MySQL data

## 🚀 Installation

### 1. Clone the Repository

```bash
git clone <your-repo-url>
cd <your-repo-name>
```

### 2. Create Required Directories on Nodes

```bash
# On each Kubernetes node, create these directories:
sudo mkdir -p /mysql-data
sudo mkdir -p /nginx-conf
sudo mkdir -p /nginx-index
sudo mkdir -p /root/secret
```

### 3. Deploy Database Layer

```bash
# Apply database secret
kubectl apply -f db-secret.yaml

# Apply database configuration
kubectl apply -f db-config.yaml

# Deploy StatefulSet and Service
kubectl apply -f db-statefulset.yaml
kubectl apply -f db-service.yaml
```

### 4. Deploy Backend Layer

```bash
# Ensure database password secret exists on host
echo -n "admin123" > /root/secret/db-password

# Deploy backend
kubectl apply -f back-deployment.yaml
kubectl apply -f back-service.yaml
```

### 5. Deploy Nginx Proxy

```bash
# Apply TLS certificates
kubectl apply -f nginx-tls-secret.yaml

# Deploy Nginx proxy
kubectl apply -f nginx-proxy.yaml
```

## ⚙️ Configuration

### Database Configuration

The database is configured via ConfigMap with the following settings:

- **Root Host**: `%` (allows connections from any host)
- **Database Name**: `example`
- **Root Password**: Stored in Secret (base64 encoded: `admin123`)

### Backend Configuration

The backend connects to the database using:
- Database host: `db` (internal DNS)
- Port: `3306`
- Credentials: Mounted from `/root/secret/db-password`

### Nginx Configuration

Custom Nginx configuration can be placed at:
- **Config**: `/nginx-conf/nginx.conf`
- **Index**: `/nginx-index/index.html`
- **Certificates**: Auto-mounted from Secret

## 🔐 Security

### Secrets Management

This project uses Kubernetes Secrets for sensitive data:

1. **Database Root Password** (`db-secret`)
   ```yaml
   MYSQL_ROOT_PASSWORD: YWRtaW4xMjM= # base64 encoded
   ```

2. **TLS Certificates** (`nginx-tls`)
   - Self-signed certificates included
   - Valid for: `localhost`
   - Replace with proper certificates for production

### Security Recommendations

- [ ] Replace default MySQL root password
- [ ] Use proper TLS certificates from a CA
- [ ] Implement network policies
- [ ] Enable Pod Security Standards
- [ ] Use Secrets store CSI driver for production
- [ ] Implement RBAC policies
- [ ] Regular security scanning of container images

## 🎯 Usage

### Accessing the Application

```bash
# Get Nginx proxy pod name
kubectl get pods | grep nginx-proxy

# Port forward to access locally
kubectl port-forward pod/nginx-proxy 8080:8080

# Access via browser (accept self-signed certificate warning)
https://localhost:8080
```

### Scaling the Backend

```bash
# Scale backend replicas
kubectl scale deployment back-deploy --replicas=5

# Verify scaling
kubectl get pods -l app=back
```

### Viewing Logs

```bash
# Backend logs
kubectl logs -l app=back --tail=100 -f

# Database logs
kubectl logs -l app=db --tail=100 -f

# Nginx logs
kubectl logs nginx-proxy --tail=100 -f
```

### Database Access

```bash
# Connect to MySQL
kubectl exec -it db-ss-0 -- mysql -u root -padmin123 example

# Create a backup
kubectl exec -it db-ss-0 -- mysqldump -u root -padmin123 example > backup.sql
```

## 🐛 Troubleshooting

### Backend Can't Connect to Database

```bash
# Check if database is ready
kubectl get pods -l app=db

# Check database service
kubectl get svc db

# Verify secret mount
kubectl exec -it <backend-pod> -- cat /run/secrets/db-password
```

### Nginx Not Serving Traffic

```bash
# Check Nginx configuration
kubectl exec -it nginx-proxy -- nginx -t

# Verify certificate mount
kubectl exec -it nginx-proxy -- ls -la /etc/nginx/ssl/
```

### Persistent Data Issues

```bash
# Check volume mounts
kubectl describe pod db-ss-0

# Verify host path exists
# On the node running the pod:
ls -la /mysql-data
```

## 📊 Monitoring

### Health Checks

```bash
# Check all pods status
kubectl get pods

# Check services
kubectl get svc

# View events
kubectl get events --sort-by='.lastTimestamp'
```

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request


## 🙏 Acknowledgments

- Built with Kubernetes
- Backend powered by Go
- Database: MySQL 8.0
- Reverse proxy: Nginx

---

