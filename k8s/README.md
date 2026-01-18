# Kubernetes Deployment for Blue-Green Application

This directory contains Kubernetes manifests and deployment scripts for the Blue-Green deployment application.

## 📁 Files Overview

### Kubernetes Manifests

- **configmap.yaml** - Application configuration (MongoDB URI, Backend URL)
- **mongodb-deployment.yaml** - MongoDB deployment with PersistentVolumeClaim
- **mongodb-service.yaml** - MongoDB ClusterIP service
- **backend-deployment.yaml** - Backend API deployment with health checks
- **backend-service.yaml** - Backend ClusterIP service
- **frontend-blue-deployment.yaml** - Blue frontend deployment
- **frontend-green-deployment.yaml** - Green frontend deployment
- **frontend-service.yaml** - Frontend NodePort service (for blue-green switching)

### Deployment Scripts

- **deploy.ps1** / **deploy.sh** - Automated deployment script
- **switch-to-green.ps1** / **switch-to-green.sh** - Switch to Green version
- **switch-to-blue.ps1** / **switch-to-blue.sh** - Switch to Blue version

## 🚀 Quick Start

### Prerequisites

1. **Minikube** installed and configured
2. **kubectl** installed
3. **Docker images** built locally:
   - `ramiz-bgd/backend:v1`
   - `ramiz-bgd-blue/frontend-blue:v1`
   - `ramiz-bgd-green/frontend-green:v1`

### Deployment Steps

#### Option 1: Automated Deployment (Recommended)

**Windows (PowerShell):**
```powershell
.\k8s\deploy.ps1
```

**Linux/Mac (Bash):**
```bash
chmod +x k8s/deploy.sh
./k8s/deploy.sh
```

#### Option 2: Manual Deployment

1. **Start Minikube:**
   ```bash
   minikube start
   ```

2. **Load Docker images into Minikube:**
   ```bash
   minikube image load ramiz-bgd/backend:v1
   minikube image load ramiz-bgd-blue/frontend-blue:v1
   minikube image load ramiz-bgd-green/frontend-green:v1
   ```

3. **Apply Kubernetes manifests:**
   ```bash
   # Apply ConfigMap
   kubectl apply -f k8s/configmap.yaml
   
   # Deploy MongoDB
   kubectl apply -f k8s/mongodb-deployment.yaml
   kubectl apply -f k8s/mongodb-service.yaml
   
   # Wait for MongoDB
   kubectl wait --for=condition=ready pod -l app=mongodb --timeout=120s
   
   # Deploy Backend
   kubectl apply -f k8s/backend-deployment.yaml
   kubectl apply -f k8s/backend-service.yaml
   
   # Wait for Backend
   kubectl wait --for=condition=ready pod -l app=backend --timeout=120s
   
   # Deploy Frontends
   kubectl apply -f k8s/frontend-blue-deployment.yaml
   kubectl apply -f k8s/frontend-green-deployment.yaml
   kubectl apply -f k8s/frontend-service.yaml
   ```

4. **Verify deployment:**
   ```bash
   kubectl get deployments
   kubectl get pods
   kubectl get services
   ```

## 🔄 Blue-Green Switching

### Switch to Green Version

**Windows:**
```powershell
.\k8s\switch-to-green.ps1
```

**Linux/Mac:**
```bash
./k8s/switch-to-green.sh
```

**Manual Command:**
```bash
kubectl patch service frontend-service -p '{
  "spec": {
    "selector": {
      "app": "frontend",
      "version": "green"
    },
    "ports": [{
      "port": 80,
      "targetPort": 3200,
      "nodePort": 30080,
      "protocol": "TCP",
      "name": "http"
    }]
  }
}'
```

### Switch to Blue Version

**Windows:**
```powershell
.\k8s\switch-to-blue.ps1
```

**Linux/Mac:**
```bash
./k8s/switch-to-blue.sh
```

**Manual Command:**
```bash
kubectl patch service frontend-service -p '{
  "spec": {
    "selector": {
      "app": "frontend",
      "version": "blue"
    },
    "ports": [{
      "port": 80,
      "targetPort": 3100,
      "nodePort": 30080,
      "protocol": "TCP",
      "name": "http"
    }]
  }
}'
```

## 🌐 Accessing the Application

### Frontend

Get the frontend URL:
```bash
minikube service frontend-service --url
```

Or access directly:
```bash
# Get Minikube IP
minikube ip

# Access frontend at http://<minikube-ip>:30080
```

### Backend

Use port-forwarding to access the backend:
```bash
kubectl port-forward service/backend 5000:5000
```

Then access at: http://localhost:5000

### MongoDB

Use port-forwarding to access MongoDB:
```bash
kubectl port-forward service/mongodb 27017:27017
```

Then connect at: mongodb://localhost:27017/registration

## 📊 Monitoring and Debugging

### Check Pod Status
```bash
kubectl get pods
kubectl get pods -w  # Watch mode
```

### View Logs
```bash
# Backend logs
kubectl logs -l app=backend -f

# Frontend Blue logs
kubectl logs -l app=frontend,version=blue -f

# Frontend Green logs
kubectl logs -l app=frontend,version=green -f

# MongoDB logs
kubectl logs -l app=mongodb -f
```

### Describe Resources
```bash
kubectl describe deployment backend
kubectl describe service frontend-service
kubectl describe pod <pod-name>
```

### Check Health
```bash
# Port-forward and test backend health
kubectl port-forward service/backend 5000:5000
curl http://localhost:5000/health

# Test frontend health (via Minikube service)
curl $(minikube service frontend-service --url)/health
```

### View Current Service Configuration
```bash
kubectl describe service frontend-service
kubectl get service frontend-service -o yaml
```

## 🔧 Resource Configuration

### Health Checks

All services have configured:
- **Liveness Probes**: Restart unhealthy containers
- **Readiness Probes**: Remove unready pods from service

### Resource Limits

| Service | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---------|-------------|-----------|----------------|--------------|
| MongoDB | 250m | 500m | 512Mi | 1Gi |
| Backend | 100m | 200m | 128Mi | 256Mi |
| Frontend Blue | 50m | 100m | 64Mi | 128Mi |
| Frontend Green | 50m | 100m | 64Mi | 128Mi |

### Replicas

- Backend: 2 replicas
- Frontend Blue: 2 replicas
- Frontend Green: 2 replicas
- MongoDB: 1 replica (with persistent storage)

## 🧹 Cleanup

### Delete All Resources
```bash
kubectl delete -f k8s/
```

### Delete Persistent Volume Claims
```bash
kubectl delete pvc mongodb-pvc
```

### Stop Minikube
```bash
minikube stop
```

### Delete Minikube Cluster
```bash
minikube delete
```

## 🐛 Troubleshooting

### Images Not Found

If you see `ImagePullBackOff` errors:
```bash
# Verify images are loaded in Minikube
minikube image ls | grep ramiz-bgd

# If not, load them:
minikube image load ramiz-bgd/backend:v1
minikube image load ramiz-bgd-blue/frontend-blue:v1
minikube image load ramiz-bgd-green/frontend-green:v1
```

### Pods Not Starting

Check pod events:
```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Service Not Accessible

Check service endpoints:
```bash
kubectl get endpoints
kubectl describe service frontend-service
```

### MongoDB Connection Issues

Check MongoDB logs:
```bash
kubectl logs -l app=mongodb
```

Verify backend can connect:
```bash
kubectl logs -l app=backend | grep -i mongo
```

## 📝 Notes

- The frontend service initially points to the **Blue** version
- Both Blue and Green versions run simultaneously
- Switching is instant via service selector update
- No downtime during version switching
- Easy rollback by switching back to the previous version
