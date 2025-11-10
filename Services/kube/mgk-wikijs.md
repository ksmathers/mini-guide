---
title: mgk-wikijs
description: 
published: true
date: 2025-10-26T17:52:20.671Z
tags: 
editor: markdown
dateCreated: 2025-10-26T17:52:18.004Z
---

# WikiJS on Kubernetes (Mini Guide)

This guide deploys WikiJS wiki platform on Kubernetes with PostgreSQL database and persistent storage.

## ✅ Step 1: Create Namespace and Setup

### 1. Create a dedicated namespace
```bash
kubectl create namespace wikijs
kubectl config set-context --current --namespace=wikijs
```

### 2. Verify cluster access
```bash
kubectl cluster-info
kubectl get nodes
```

## ✅ Step 2: Create Persistent Storage

### 3. Create PersistentVolumeClaims for data storage
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: wikijs
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wikijs-config-pvc
  namespace: wikijs
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

### 4. Verify PVCs are created
```bash
kubectl get pvc -n wikijs
```

## ✅ Step 3: Database Configuration

### 5. Create Secret for database credentials
```bash
kubectl create secret generic postgres-secret -n wikijs \
  --from-literal=POSTGRES_DB=wiki \
  --from-literal=POSTGRES_USER=wikijs \
  --from-literal=POSTGRES_PASSWORD=wikijspass
```

### 6. Deploy PostgreSQL database
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: wikijs
spec:
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
        envFrom:
        - secretRef:
            name: postgres-secret
        ports:
        - containerPort: 5432
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
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: wikijs
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
EOF
```

### 7. Verify PostgreSQL deployment
```bash
kubectl get pods -n wikijs -l app=postgres
kubectl logs -n wikijs -l app=postgres
```

## ✅ Step 4: WikiJS Deployment

### 8. Create WikiJS ConfigMap for environment variables
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: wikijs-config
  namespace: wikijs
data:
  DB_TYPE: "postgres"
  DB_HOST: "postgres-service"
  DB_PORT: "5432"
  DB_USER: "wikijs"
  DB_NAME: "wiki"
EOF
```

### 9. Deploy WikiJS application
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wikijs
  namespace: wikijs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wikijs
  template:
    metadata:
      labels:
        app: wikijs
    spec:
      containers:
      - name: wikijs
        image: requarks/wiki:2
        ports:
        - containerPort: 3000
        envFrom:
        - configMapRef:
            name: wikijs-config
        env:
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_PASSWORD
        volumeMounts:
        - name: wikijs-config-storage
          mountPath: /wiki/config
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /healthz
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /healthz
            port: 3000
          initialDelaySeconds: 60
          periodSeconds: 30
      volumes:
      - name: wikijs-config-storage
        persistentVolumeClaim:
          claimName: wikijs-config-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: wikijs-service
  namespace: wikijs
spec:
  selector:
    app: wikijs
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30300
  type: NodePort
EOF
```

### 10. Verify WikiJS deployment
```bash
kubectl get pods -n wikijs -l app=wikijs
kubectl logs -n wikijs -l app=wikijs
```

## ✅ Step 5: Access and Testing

### 11. Get WikiJS service access information
```bash
# Check service details
kubectl get service wikijs-service -n wikijs

# Get node IP for NodePort access
kubectl get nodes -o wide
```

### 12. Test database connectivity
```bash
# Check if WikiJS can connect to PostgreSQL
kubectl exec -n wikijs -it deployment/wikijs -- sh -c "nc -zv postgres-service 5432"

# Check PostgreSQL logs for connections
kubectl logs -n wikijs -l app=postgres --tail=10
```

### 13. Access WikiJS web interface

**Method 1: NodePort (Direct cluster access):**
```bash
# Get any node IP
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
if [ -z "$NODE_IP" ]; then
  NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
fi
echo "Access WikiJS at: http://$NODE_IP:30300"
```

**Method 2: Port-Forward (Local development):**
```bash
# Run port-forward in background
kubectl port-forward -n wikijs service/wikijs-service 3000:3000 &
echo "Access WikiJS at: http://localhost:3000"
```

**Method 3: LoadBalancer (Cloud environments - optional):**
```bash
# Convert service to LoadBalancer if needed
kubectl patch service wikijs-service -n wikijs -p '{"spec":{"type":"LoadBalancer"}}'
# Wait for external IP assignment (may take several minutes)
kubectl get service wikijs-service -n wikijs --watch
```

### 14. Complete WikiJS installation via web browser
1. **Administrator Account Setup**
   - Enter administrator email address
   - Set administrator password  
   - Click "Install"

2. **General Settings**
   - Set site title (e.g., "Kubernetes Wiki")
   - Set site description
   - Choose site URL format

3. **Storage Configuration**
   - Select "Local File System" for content storage
   - Keep default path: `/wiki/data`

4. **Authentication**
   - Configure local authentication or connect external providers
   - Complete the setup wizard

### 15. Verify installation and persistence
```bash
# Create a test page in the web interface, then restart pods
kubectl rollout restart deployment/wikijs -n wikijs
kubectl rollout restart deployment/postgres -n wikijs

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app=wikijs -n wikijs --timeout=120s
kubectl wait --for=condition=ready pod -l app=postgres -n wikijs --timeout=120s

# Refresh browser - data should persist
```

## ✅ Step 6: Management and Scaling

### 16. Scale WikiJS deployment (optional)
```bash
kubectl scale deployment wikijs -n wikijs --replicas=2
kubectl get pods -n wikijs -l app=wikijs
```

### 17. Update WikiJS (rolling update)
```bash
kubectl set image deployment/wikijs -n wikijs wikijs=requarks/wiki:latest
kubectl rollout status deployment/wikijs -n wikijs
```

### 18. Cleanup (optional)
```bash
# Remove all WikiJS resources
kubectl delete namespace wikijs

# Or remove individual components
kubectl delete deployment,service,pvc,secret,configmap --all -n wikijs
```

## 📋 Quick Reference

### Namespace Information
- **Namespace**: `wikijs`
- **PostgreSQL Service**: `postgres-service:5432`
- **WikiJS Service**: `wikijs-service:3000`

### Persistent Storage
- **PostgreSQL Data**: PVC `postgres-pvc` (5Gi)
- **WikiJS Config**: PVC `wikijs-config-pvc` (1Gi)

### Useful Commands
```bash
# View all resources
kubectl get all -n wikijs

# Check pod logs
kubectl logs -n wikijs -l app=wikijs --tail=20
kubectl logs -n wikijs -l app=postgres --tail=20

# Execute commands in pods
kubectl exec -n wikijs -it deployment/wikijs -- sh
kubectl exec -n wikijs -it deployment/postgres -- psql -U wikijs -d wiki

# Port forward for local access
kubectl port-forward -n wikijs service/wikijs-service 3000:3000

# Backup database
kubectl exec -n wikijs deployment/postgres -- pg_dump -U wikijs wiki > wiki-backup.sql
```

## 🔧 Troubleshooting

- **Pods not starting**: Check resource limits and node capacity with `kubectl describe pod`
- **Database connection failed**: Verify service names and secret values match
- **PVC pending**: Check if cluster has dynamic provisioning or create PVs manually  
- **NodePort not accessible**: Verify firewall rules allow port 30300 and check node IP addresses
- **WikiJS not accessible**: Ensure pods are ready and check service endpoints with `kubectl get endpoints`
- **Port-forward fails**: Check if port 3000 is already in use locally

## 🔒 Production Considerations

- Use proper resource limits and requests
- Configure backup strategy for PostgreSQL PVC
- Set up monitoring and alerting
- Use TLS/HTTPS with Ingress controller
- Consider using managed database service
- Implement network policies for security

---

*Guide tested with Kubernetes 1.28+ and WikiJS 2.x*