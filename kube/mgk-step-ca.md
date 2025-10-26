# Step-CA Certificate Authority on Kubernetes (Mini Guide)

This guide deploys a step-ca server on Kubernetes for managing certificates in a homelab environment, including ACME and manual certificate management.

## ⚙️ Configuration Variables

Set these environment variables to customize your CA deployment:

```bash
# Basic CA configuration
export CA_NAME="Homelab CA"
export CA_HOSTNAME="ca.example.com"
export CA_PASSWORD="your-secure-ca-password"
export CA_NAMESPACE="step-ca"

# Verify variables are set
echo "CA Name: $CA_NAME"
echo "CA Hostname: $CA_HOSTNAME"
echo "CA Namespace: $CA_NAMESPACE"
```

## ✅ Step 1: Setup Namespace and CA Storage

**Note on DNS**: This guide uses `example.com` for external DNS names and Kubernetes internal DNS for cluster services. For production use, replace with your actual domain. The step-ca server will be accessible via NodePort, so no mDNS/Avahi is required.

### 1. Create namespace and persistent storage
```bash
# Create dedicated namespace
kubectl create namespace $CA_NAMESPACE
kubectl config set-context --current --namespace=$CA_NAMESPACE

# Create PVC for CA data and configuration
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: step-ca-data-pvc
  namespace: $CA_NAMESPACE
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: step-ca-config-pvc
  namespace: $CA_NAMESPACE
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Verify PVCs are created
kubectl get pvc -n $CA_NAMESPACE
```

## ✅ Step 2: Initialize Certificate Authority

### 2. Initialize CA using temporary pod
```bash
# Create initialization pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: step-ca-init
  namespace: $CA_NAMESPACE
spec:
  containers:
  - name: step-ca-init
    image: smallstep/step-ca:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - name: ca-data
      mountPath: /home/step
    - name: ca-config
      mountPath: /etc/step-ca
    env:
    - name: STEPPATH
      value: "/home/step"
  volumes:
  - name: ca-data
    persistentVolumeClaim:
      claimName: step-ca-data-pvc
  - name: ca-config
    persistentVolumeClaim:
      claimName: step-ca-config-pvc
EOF

# Wait for pod to be ready
kubectl wait --for=condition=ready pod step-ca-init -n $CA_NAMESPACE --timeout=120s

# Initialize the CA using environment variables
kubectl exec -n $CA_NAMESPACE -it step-ca-init -- step ca init \
  --name="$CA_NAME" \
  --dns="step-ca-service.$CA_NAMESPACE.svc.cluster.local,$CA_HOSTNAME" \
  --address=":9000" \
  --provisioner="admin" \
  --password-file=/dev/stdin <<< "$CA_PASSWORD"

# Copy configuration to config volume
kubectl exec -n $CA_NAMESPACE step-ca-init -- cp -r /home/step/.step /etc/step-ca/

# Clean up init pod
kubectl delete pod step-ca-init -n $CA_NAMESPACE
```

## ✅ Step 3: Deploy Step-CA Server

### 3. Create CA configuration and deploy server
```bash
# Create ConfigMap for CA settings
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: step-ca-config
  namespace: $CA_NAMESPACE
data:
  DOCKER_STEPCA_INIT_NAME: "$CA_NAME"
  DOCKER_STEPCA_INIT_DNS_NAMES: "step-ca-service.$CA_NAMESPACE.svc.cluster.local,step-ca-service,$CA_HOSTNAME,localhost"
---
apiVersion: v1
kind: Secret
metadata:
  name: step-ca-password
  namespace: $CA_NAMESPACE
type: Opaque
stringData:
  password: "$CA_PASSWORD"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: step-ca
  namespace: $CA_NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: step-ca
  template:
    metadata:
      labels:
        app: step-ca
    spec:
      containers:
      - name: step-ca
        image: smallstep/step-ca:latest
        ports:
        - containerPort: 9000
        envFrom:
        - configMapRef:
            name: step-ca-config
        env:
        - name: DOCKER_STEPCA_INIT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: step-ca-password
              key: password
        volumeMounts:
        - name: ca-data
          mountPath: /home/step
        - name: ca-config
          mountPath: /etc/step-ca
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health
            port: 9000
          initialDelaySeconds: 10
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 9000
          initialDelaySeconds: 30
          periodSeconds: 30
      volumes:
      - name: ca-data
        persistentVolumeClaim:
          claimName: step-ca-data-pvc
      - name: ca-config
        persistentVolumeClaim:
          claimName: step-ca-config-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: step-ca-service
  namespace: $CA_NAMESPACE
spec:
  selector:
    app: step-ca
  ports:
  - port: 9000
    targetPort: 9000
    nodePort: 30900
  type: NodePort
EOF

# Verify deployment
kubectl get pods -n $CA_NAMESPACE -l app=step-ca
kubectl logs -n $CA_NAMESPACE -l app=step-ca --tail=20
```

## ✅ Step 4: Setup ACME Provisioner

### 4. Configure ACME provisioner for automatic certificates
```bash
# Get CA service IP for configuration
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
if [ -z "$NODE_IP" ]; then
  NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
fi
echo "CA Server available at: https://$NODE_IP:30900"

# Add ACME provisioner (run this inside step-ca pod)
kubectl exec -n $CA_NAMESPACE deployment/step-ca -- step ca provisioner add acme --type ACME

# Restart step-ca to load new provisioner
kubectl rollout restart deployment/step-ca -n $CA_NAMESPACE
kubectl wait --for=condition=ready pod -l app=step-ca -n $CA_NAMESPACE --timeout=120s

# Verify ACME provisioner is active
kubectl exec -n $CA_NAMESPACE deployment/step-ca -- step ca provisioner list
```

## ✅ Step 5: Install Cert-Manager for ACME Integration

### 5. Deploy cert-manager for automated certificate management
```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.yaml

# Wait for cert-manager to be ready
kubectl wait --for=condition=ready pod -l app=cert-manager -n cert-manager --timeout=300s
kubectl wait --for=condition=ready pod -l app=cainjector -n cert-manager --timeout=300s
kubectl wait --for=condition=ready pod -l app=webhook -n cert-manager --timeout=300s

# Create ClusterIssuer for step-ca ACME
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: step-ca-acme
spec:
  acme:
    server: https://step-ca-service.$CA_NAMESPACE.svc.cluster.local:9000/acme/acme/directory
    skipTLSVerify: true  # Only for homelab - use proper TLS in production
    privateKeySecretRef:
      name: step-ca-acme-key
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

# Verify ClusterIssuer is ready
kubectl get clusterissuer step-ca-acme -o wide
```

## ✅ Step 6: Usage Examples

Set these additional variables for the usage examples:

```bash
# Example application configuration
export ADMIN_EMAIL="admin@example.com"
export APP_HOSTNAME="hello.example.com"
export API_HOSTNAME="api.example.com"
export WEBAPP_NAMESPACE="webapp"

# Verify example variables
echo "Admin Email: $ADMIN_EMAIL"
echo "App Hostname: $APP_HOSTNAME"
echo "API Hostname: $API_HOSTNAME"
echo "WebApp Namespace: $WEBAPP_NAMESPACE"
```

### 6a. ACME Certificate Example - Web Server with Automatic TLS

```bash
# Create namespace for web app
kubectl create namespace $WEBAPP_NAMESPACE

# Deploy simple web server with ACME certificate
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: $WEBAPP_NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: hello-world-html
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-world-html
  namespace: $WEBAPP_NAMESPACE
data:
  index.html: |
    <html>
    <head><title>Homelab HTTPS</title></head>
    <body>
      <h1>Hello from Kubernetes!</h1>
      <p>This page is served with a certificate from step-ca via ACME!</p>
    </body>
    </html>
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world-service
  namespace: $WEBAPP_NAMESPACE
spec:
  selector:
    app: hello-world
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: hello-world-tls
  namespace: $WEBAPP_NAMESPACE
spec:
  secretName: hello-world-tls-secret
  issuerRef:
    name: step-ca-acme
    kind: ClusterIssuer
  dnsNames:
  - $APP_HOSTNAME
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
  namespace: $WEBAPP_NAMESPACE
  annotations:
    cert-manager.io/cluster-issuer: step-ca-acme
spec:
  tls:
  - hosts:
    - $APP_HOSTNAME
    secretName: hello-world-tls-secret
  rules:
  - host: $APP_HOSTNAME
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world-service
            port:
              number: 80
EOF

# Check certificate status
kubectl get certificate -n $WEBAPP_NAMESPACE
kubectl describe certificate hello-world-tls -n $WEBAPP_NAMESPACE
```

### 6b. Manual Certificate Management via CLI

```bash
# Create step-cli client pod for manual certificate operations
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: step-cli
  namespace: $CA_NAMESPACE
spec:
  containers:
  - name: step-cli
    image: smallstep/step-cli:latest
    command: ["sleep", "3600"]
    env:
    - name: STEPPATH
      value: "/home/step"
    volumeMounts:
    - name: step-config
      mountPath: /home/step
  volumes:
  - name: step-config
    emptyDir: {}
EOF

# Bootstrap step-cli with CA
kubectl exec -n $CA_NAMESPACE -it step-cli -- step ca bootstrap \
  --ca-url https://step-ca-service:9000 \
  --fingerprint $(kubectl exec -n $CA_NAMESPACE deployment/step-ca -- step certificate fingerprint /etc/step-ca/.step/certs/root_ca.crt)

# Generate server certificate manually
kubectl exec -n $CA_NAMESPACE -it step-cli -- step ca certificate \
  "$API_HOSTNAME" \
  api-cert.pem api-key.pem \
  --provisioner admin \
  --san "api.$WEBAPP_NAMESPACE.svc.cluster.local" \
  --san "api.$WEBAPP_NAMESPACE" \
  --san api

# Generate client certificate
kubectl exec -n $CA_NAMESPACE -it step-cli -- step ca certificate \
  "$ADMIN_EMAIL" \
  client-cert.pem client-key.pem \
  --provisioner admin

# Create Kubernetes secrets from certificates
kubectl exec -n $CA_NAMESPACE step-cli -- cat api-cert.pem | kubectl create secret tls api-server-cert --cert=/dev/stdin --key=<(kubectl exec -n $CA_NAMESPACE step-cli -- cat api-key.pem) -n $WEBAPP_NAMESPACE

# Verify certificates
kubectl exec -n $CA_NAMESPACE -it step-cli -- step certificate inspect api-cert.pem
kubectl exec -n $CA_NAMESPACE -it step-cli -- step certificate verify api-cert.pem --roots /home/step/.step/certs/root_ca.crt
```

### 6c. Database TLS Certificate Example

```bash
# Create namespace for database
kubectl create namespace database

# Generate database server certificate
kubectl exec -n $CA_NAMESPACE -it step-cli -- step ca certificate \
  "postgres.database.svc.cluster.local" \
  postgres-server.pem postgres-server-key.pem \
  --provisioner admin \
  --san postgres \
  --san postgres.database \
  --san postgres.database.svc.cluster.local \
  --san localhost \
  --san 127.0.0.1

# Create secret for PostgreSQL TLS
kubectl create secret tls postgres-tls-secret \
  --cert=<(kubectl exec -n $CA_NAMESPACE step-cli -- cat postgres-server.pem) \
  --key=<(kubectl exec -n $CA_NAMESPACE step-cli -- cat postgres-server-key.pem) \
  -n database

# Deploy PostgreSQL with TLS
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: database
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
        env:
        - name: POSTGRES_DB
          value: homelab
        - name: POSTGRES_USER
          value: admin
        - name: POSTGRES_PASSWORD
          value: secure-password
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: tls-certs
          mountPath: /var/lib/postgresql/certs
          readOnly: true
        - name: postgres-config
          mountPath: /etc/postgresql
        command:
        - postgres
        - -c
        - ssl=on
        - -c
        - ssl_cert_file=/var/lib/postgresql/certs/tls.crt
        - -c
        - ssl_key_file=/var/lib/postgresql/certs/tls.key
      volumes:
      - name: tls-certs
        secret:
          secretName: postgres-tls-secret
      - name: postgres-config
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: database
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
EOF
```

### 6d. SSH Certificate Authentication Example

```bash
# Add SSH provisioner to step-ca
kubectl exec -n $CA_NAMESPACE deployment/step-ca -- step ca provisioner add sshpop --type SSHPOP

# Restart step-ca to load SSH provisioner
kubectl rollout restart deployment/step-ca -n $CA_NAMESPACE
kubectl wait --for=condition=ready pod -l app=step-ca -n $CA_NAMESPACE --timeout=120s

# Configure SSH host certificates for a target server
# Generate SSH host certificate (run this for each server you want to manage)
export SSH_HOSTNAME="server1.example.com"
kubectl exec -n $CA_NAMESPACE -it step-cli -- step ssh certificate \
  "$SSH_HOSTNAME" \
  ssh-host-cert.pub \
  --host \
  --provisioner sshpop \
  --principal "$SSH_HOSTNAME" \
  --principal "server1" \
  --principal "$(echo $SSH_HOSTNAME | cut -d. -f1)"

# Generate SSH user certificate for admin access
kubectl exec -n $CA_NAMESPACE -it step-cli -- step ssh certificate \
  "$ADMIN_EMAIL" \
  ssh-user-cert.pub \
  --provisioner sshpop \
  --principal admin \
  --principal root \
  --principal "$(echo $ADMIN_EMAIL | cut -d@ -f1)"

# Create ConfigMap with SSH CA public key for servers
SSH_CA_PUB=$(kubectl exec -n $CA_NAMESPACE deployment/step-ca -- step ssh config --roots)
kubectl create configmap ssh-ca-config \
  --from-literal=ssh-ca.pub="$SSH_CA_PUB" \
  -n $CA_NAMESPACE

# Example: Deploy a bastion host with SSH certificate authentication
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ssh-bastion
  namespace: $CA_NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ssh-bastion
  template:
    metadata:
      labels:
        app: ssh-bastion
    spec:
      containers:
      - name: openssh-server
        image: lscr.io/linuxserver/openssh-server:latest
        ports:
        - containerPort: 2222
        env:
        - name: PUID
          value: "1000"
        - name: PGID
          value: "1000"
        - name: TZ
          value: "UTC"
        - name: PUBLIC_KEY_DIR
          value: "/config/ssh-ca"
        - name: USER_NAME
          value: "admin"
        - name: SUDO_ACCESS
          value: "true"
        volumeMounts:
        - name: ssh-config
          mountPath: /config/ssh
        - name: ssh-ca
          mountPath: /config/ssh-ca
          readOnly: true
        - name: sshd-config
          mountPath: /etc/ssh/sshd_config.d
          readOnly: true
      volumes:
      - name: ssh-config
        emptyDir: {}
      - name: ssh-ca
        configMap:
          name: ssh-ca-config
      - name: sshd-config
        configMap:
          name: sshd-ca-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: sshd-ca-config
  namespace: $CA_NAMESPACE
data:
  99-step-ca.conf: |
    # Enable SSH certificate authentication
    TrustedUserCAKeys /config/ssh-ca/ssh-ca.pub
    PubkeyAuthentication yes
    AuthorizedKeysFile none
---
apiVersion: v1
kind: Service
metadata:
  name: ssh-bastion-service
  namespace: $CA_NAMESPACE
spec:
  selector:
    app: ssh-bastion
  ports:
  - port: 2222
    targetPort: 2222
    nodePort: 30222
  type: NodePort
EOF

# Get certificates for local use
kubectl cp $CA_NAMESPACE/$(kubectl get pod -n $CA_NAMESPACE -l app=step-cli -o jsonpath='{.items[0].metadata.name}'):ssh-user-cert.pub ./ssh-user-cert.pub
kubectl cp $CA_NAMESPACE/$(kubectl get pod -n $CA_NAMESPACE -l app=step-cli -o jsonpath='{.items[0].metadata.name}'):ssh-host-cert.pub ./ssh-host-cert.pub

# Configure local SSH client to use certificates
mkdir -p ~/.ssh/step-ca
cp ssh-user-cert.pub ~/.ssh/step-ca/
echo "Host *.example.com" >> ~/.ssh/config
echo "  CertificateFile ~/.ssh/step-ca/ssh-user-cert.pub" >> ~/.ssh/config
echo "  IdentitiesOnly yes" >> ~/.ssh/config

# Test SSH certificate authentication
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
if [ -z "$NODE_IP" ]; then
  NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
fi
echo "SSH bastion available at: ssh admin@$NODE_IP -p 30222"
echo "Certificates valid for principals: admin, root, $(echo $ADMIN_EMAIL | cut -d@ -f1)"
```

### 6e. Service Mesh mTLS Example

```bash
# Generate client certificates for service-to-service communication
kubectl exec -n $CA_NAMESPACE -it step-cli -- step ca certificate \
  "frontend.$WEBAPP_NAMESPACE.svc.cluster.local" \
  frontend-client.pem frontend-client-key.pem \
  --provisioner admin

kubectl exec -n $CA_NAMESPACE -it step-cli -- step ca certificate \
  "backend.$WEBAPP_NAMESPACE.svc.cluster.local" \
  backend-server.pem backend-server-key.pem \
  --provisioner admin

# Create secrets for mTLS
kubectl create secret tls frontend-client-cert \
  --cert=<(kubectl exec -n $CA_NAMESPACE step-cli -- cat frontend-client.pem) \
  --key=<(kubectl exec -n $CA_NAMESPACE step-cli -- cat frontend-client-key.pem) \
  -n $WEBAPP_NAMESPACE

kubectl create secret tls backend-server-cert \
  --cert=<(kubectl exec -n $CA_NAMESPACE step-cli -- cat backend-server.pem) \
  --key=<(kubectl exec -n $CA_NAMESPACE step-cli -- cat backend-server-key.pem) \
  -n $WEBAPP_NAMESPACE

# Example backend service requiring client certificates
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-backend
  namespace: $WEBAPP_NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secure-backend
  template:
    metadata:
      labels:
        app: secure-backend
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 443
        volumeMounts:
        - name: server-certs
          mountPath: /etc/nginx/certs
          readOnly: true
        - name: ca-cert
          mountPath: /etc/nginx/ca
          readOnly: true
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
      volumes:
      - name: server-certs
        secret:
          secretName: backend-server-cert
      - name: ca-cert
        configMap:
          name: ca-certificates
      - name: nginx-config
        configMap:
          name: nginx-mtls-config
EOF
```

## ✅ Step 7: Monitoring and Management

### 7. Monitor CA and certificate health
```bash
# Check CA health
kubectl exec -n $CA_NAMESPACE deployment/step-ca -- step ca health

# List all provisioners
kubectl exec -n $CA_NAMESPACE deployment/step-ca -- step ca provisioner list

# Check certificate expiration
kubectl get certificates -A
kubectl describe certificate hello-world-tls -n $WEBAPP_NAMESPACE

# View CA logs
kubectl logs -n $CA_NAMESPACE -l app=step-ca --tail=50

# Backup CA data (important!)
kubectl cp $CA_NAMESPACE/$(kubectl get pod -n $CA_NAMESPACE -l app=step-ca -o jsonpath='{.items[0].metadata.name}'):/home/step ~/.step-ca-backup

# Certificate renewal check
kubectl exec -n $CA_NAMESPACE -it step-cli -- step ca renew api-cert.pem api-key.pem
```

## 📋 Quick Reference

### Connection Information
- **CA Server**: `https://<NODE_IP>:30900`
- **ACME Directory**: `https://step-ca-service.$CA_NAMESPACE.svc.cluster.local:9000/acme/acme/directory`
- **Health Endpoint**: `https://<NODE_IP>:30900/health`

### Common Commands
```bash
# Get CA root certificate
kubectl exec -n $CA_NAMESPACE deployment/step-ca -- step ca root

# List active certificates
kubectl get certificates -A

# Force certificate renewal
kubectl annotate certificate hello-world-tls -n $WEBAPP_NAMESPACE cert-manager.io/force-renewal=$(date +%s)

# Check ACME account
kubectl get certificaterequests -A

# SSH certificate operations
kubectl exec -n $CA_NAMESPACE deployment/step-ca -- step ssh list --type host
kubectl exec -n $CA_NAMESPACE deployment/step-ca -- step ssh list --type user
kubectl exec -n $CA_NAMESPACE -it step-cli -- step ssh inspect ssh-user-cert.pub

# View step-ca configuration
kubectl exec -n $CA_NAMESPACE deployment/step-ca -- cat /etc/step-ca/.step/config/ca.json
```

## 🔧 Troubleshooting

- **ACME challenges fail**: Check ingress controller is installed and DNS resolves correctly
- **Certificate not issued**: Verify ClusterIssuer status with `kubectl describe clusterissuer step-ca-acme`
- **CA not accessible**: Check service ports and firewall rules
- **TLS verification fails**: Ensure CA root certificate is trusted by clients
- **SSH certificate rejected**: Verify TrustedUserCAKeys is configured on target servers
- **SSH authentication fails**: Check certificate principals match target user/host
- **Pod startup fails**: Check persistent volume permissions and available storage

## ⚙️ Configuration Options

### CA Settings
- **Certificate lifetime**: Default 24h, configurable in provisioner settings
- **ACME challenge types**: HTTP-01 (default), DNS-01 for wildcards
- **Key types**: RSA-2048 (default), ECDSA-256, RSA-4096
- **Renewal threshold**: 2/3 of certificate lifetime (cert-manager default)

### Security Considerations
- Change default passwords before production use
- Enable audit logging for certificate operations  
- Set appropriate RBAC for step-ca namespace
- Use network policies to restrict CA access
- Regular backup of CA private keys and configuration
- Monitor certificate expiration and renewal

## 🔒 Production Considerations

- Use proper DNS names instead of IP addresses
- Implement proper TLS verification (remove skipTLSVerify)
- Set up high availability with multiple CA replicas
- Configure proper backup and disaster recovery
- Implement certificate transparency logging
- Use external secrets management (Vault, etc.)
- Set up monitoring and alerting for certificate expiry
- Configure appropriate resource limits and requests

---

*Guide tested with step-ca 0.25+ and cert-manager 1.13+*