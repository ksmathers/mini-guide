---
title: mgk-avahi-advertiser
description: 
published: true
date: 2025-11-22T00:00:00.000Z
tags: 
editor: markdown
dateCreated: 2025-11-22T00:00:00.000Z
---

# Avahi Advertiser for Kubernetes (Mini Guide)

This guide deploys an automatic Avahi/mDNS advertiser that watches Kubernetes services and makes them accessible via `.local` domain names (e.g., `jellyfin.local`, `wikijs.local`).

## 📋 Overview

When you deploy services in Kubernetes, they need DNS names for easy access. This service watches Kubernetes services and automatically creates appropriate Avahi advertisements:

- **LoadBalancer services**: Creates A records in `/etc/avahi/hosts` (hostname → IP mapping)
- **NodePort services**: Creates service records in `/etc/avahi/services` (mDNS-SD for service discovery)

This enables mDNS discovery on your local network without manual DNS configuration.

**Perfect for homelab environments using:**
- Minikube with `--driver=none`
- MetalLB for LoadBalancer IPs
- Avahi daemon on the host

## ⚙️ Prerequisites

Verify your environment meets these requirements:

```bash
# 1. Minikube running with driver=none
minikube status

# 2. Avahi daemon installed and running on host
systemctl status avahi-daemon

# 3. Avahi hosts file and services directory exist
ls -la /etc/avahi/hosts /etc/avahi/services/

# 4. MetalLB installed (for LoadBalancer services)
kubectl get pods -n metallb-system
```

## ✅ Step 1: Setup Git Repository

### 1. Create GitHub repository for the advertiser code

**Via GitHub Web Interface:**
1. Go to https://github.com/new
2. Repository name: `k8s-avahi-advertiser`
3. Description: "Automatic Avahi/mDNS advertiser for Kubernetes services"
4. Choose Public or Private
5. **Do not** initialize with README, .gitignore, or license
6. Click "Create repository"

**Save the repository URL** (you'll need it in the next step):
```bash
export REPO_URL="https://github.com/YOUR_USERNAME/k8s-avahi-advertiser.git"
echo "Repository URL: $REPO_URL"
```

### 2. Initialize and push the advertiser code

The code is located in the `avahi-advertiser/` directory of this guide. Let's create a git repository and push it:

```bash
# Navigate to the advertiser code directory
cd Services/kube/avahi-advertiser/

# Initialize git repository
git init

# Add all files
git add .

# Create initial commit
git commit -m "Initial commit: Kubernetes Avahi service advertiser"

# Add remote repository
git remote add origin $REPO_URL

# Push to GitHub
git branch -M main
git push -u origin main
```

### 3. Verify the repository
```bash
# View repository status
git status

# View remote URL
git remote -v

# You should also see the files on GitHub at:
# https://github.com/YOUR_USERNAME/k8s-avahi-advertiser
```

## ✅ Step 2: Build Container Image

### 4. Build the Docker image locally

Since we're using minikube with `driver=none`, we can build directly on the host:

```bash
# Ensure you're in the advertiser directory
cd Services/kube/avahi-advertiser/

# Build the Docker image
docker build -t localhost/avahi-advertiser:latest .

# Verify the image was created
docker images | grep avahi-advertiser
```

**Note**: We use `localhost/` prefix because minikube with `driver=none` uses the host's Docker daemon directly.

### 5. (Optional) Test the image locally

Before deploying to Kubernetes, you can test the image:

```bash
# Run a test container (will fail without kubeconfig, but validates the image)
docker run --rm localhost/avahi-advertiser:latest python -c "import avahi_k8s_advertiser; print('Import successful')"
```

## ✅ Step 3: Deploy to Kubernetes

### 6. Review the manifest file

The `k8s-manifests.yaml` file contains:
- **ServiceAccount**: Identity for the advertiser pods
- **ClusterRole**: Permissions to watch services
- **ClusterRoleBinding**: Links ServiceAccount to ClusterRole
- **DaemonSet**: Runs advertiser on nodes with Avahi

```bash
# View the manifest
cat k8s-manifests.yaml
```

### 7. Deploy the advertiser

```bash
# Apply the Kubernetes manifests
kubectl apply -f k8s-manifests.yaml

# Verify the DaemonSet was created
kubectl get daemonset -n kube-system avahi-advertiser
```

### 8. Verify deployment

```bash
# Check if pod is running
kubectl get pods -n kube-system -l app=avahi-advertiser

# View pod logs
kubectl logs -n kube-system -l app=avahi-advertiser --tail=20

# Expected output should show:
# - "Kubernetes Avahi Advertiser starting..."
# - "Syncing existing services..."
# - "Starting to watch Kubernetes services..."
```

## ✅ Step 4: Test Advertisement

### 9. Deploy a test service (if you don't have one already)

If you don't have a LoadBalancer service yet, create one:

```bash
kubectl create namespace test-avahi
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
  namespace: test-avahi
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test
  namespace: test-avahi
  annotations:
    avahi.local/name: "nginx-test"
spec:
  type: LoadBalancer
  selector:
    app: nginx-test
  ports:
  - port: 80
    targetPort: 80
EOF
```

### 10. Verify service got LoadBalancer IP

```bash
# Wait for LoadBalancer IP assignment
kubectl wait --for=jsonpath='{.status.loadBalancer.ingress}' \
  service/nginx-test -n test-avahi --timeout=60s

# Get the assigned IP
kubectl get service nginx-test -n test-avahi
```

### 11. Check Avahi A record was created

```bash
# Check Avahi hosts file
cat /etc/avahi/hosts

# You should see an entry like:
# 192.168.x.x nginx-test.local # Managed by k8s-avahi-advertiser (test-avahi/nginx-test)

# List managed entries
grep "Managed by k8s-avahi-advertiser" /etc/avahi/hosts
```

**Note**: LoadBalancer services create A records in `/etc/avahi/hosts`, not service files. NodePort services would create files in `/etc/avahi/services/`.

### 12. Test mDNS resolution

```bash
# Test DNS resolution
avahi-resolve -n nginx-test.local

# Expected output: nginx-test.local    192.168.x.x

# Test HTTP access
curl http://nginx-test.local/

# Or open in browser
xdg-open http://nginx-test.local/
```

## ✅ Step 5: Configure Existing Services

### 13. Add annotations to existing services

For services like Jellyfin, add annotations to customize advertisement:

```bash
# Example: Annotate Jellyfin service
kubectl annotate service jellyfin-service -n jellyfin \
  avahi.local/name="jellyfin"

# Check advertiser logs to see it pick up the change
kubectl logs -n kube-system -l app=avahi-advertiser --tail=10
```

### 14. Verify Avahi A record was created

```bash
# Check for Jellyfin A record
grep jellyfin /etc/avahi/hosts

# Expected: 192.168.x.x jellyfin.local # Managed by k8s-avahi-advertiser (jellyfin/jellyfin-service)

# Test resolution
avahi-resolve -n jellyfin.local

# Test access
curl http://jellyfin.local:8096/
```

## ✅ Step 6: Advanced Configuration

### 15. Customize service advertisement with annotations

Available annotations (all prefixed with `avahi.local/`):

| Annotation | Purpose | Applies To | Example |
|------------|---------|------------|---------|
| `enabled` | Enable/disable advertisement | Both | `"false"` to disable |
| `name` | Custom mDNS name | Both | `"myservice"` |
| `service-type` | Avahi service type (mDNS-SD) | NodePort only | `"_https._tcp"` |
| `txt-*` | Add TXT records | NodePort only | `txt-path: "/api"` |

**Example LoadBalancer service (creates A record):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-lb-service
  namespace: default
  annotations:
    avahi.local/name: "myapp"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 443
    targetPort: 8443
EOF
```

**Example NodePort service (creates service record with mDNS-SD):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
  namespace: default
  annotations:
    avahi.local/name: "myapi"
    avahi.local/service-type: "_https._tcp"
    avahi.local/txt-path: "/api/v1"
    avahi.local/txt-version: "2.0"
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 443
    targetPort: 8443
    nodePort: 30443
EOF
```

### 16. Disable advertisement for specific services

```bash
# Disable advertisement for a service
kubectl annotate service my-service -n default \
  avahi.local/enabled="false"

# The advertiser will remove the Avahi A record or service file automatically
```

## ✅ Step 7: Maintenance and Updates

### 17. Update the advertiser code

```bash
# Make changes to the Python code
cd Services/kube/avahi-advertiser/

# Edit avahi_k8s_advertiser.py as needed
nano avahi_k8s_advertiser.py

# Commit changes
git add .
git commit -m "Update: description of changes"
git push

# Rebuild the Docker image
docker build -t localhost/avahi-advertiser:latest .

# Restart the DaemonSet pods to use new image
kubectl rollout restart daemonset/avahi-advertiser -n kube-system

# Watch the rollout
kubectl rollout status daemonset/avahi-advertiser -n kube-system
```

### 18. View advertiser logs

```bash
# View current logs
kubectl logs -n kube-system -l app=avahi-advertiser --tail=50

# Follow logs in real-time
kubectl logs -n kube-system -l app=avahi-advertiser -f

# View logs with timestamps
kubectl logs -n kube-system -l app=avahi-advertiser --timestamps=true
```

### 19. Adjust resource limits if needed

```bash
# Edit the DaemonSet
kubectl edit daemonset avahi-advertiser -n kube-system

# Or patch with new resource limits
kubectl patch daemonset avahi-advertiser -n kube-system --type=json -p='[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/resources/limits/memory",
    "value": "256Mi"
  }
]'
```

### 20. Cleanup test resources (optional)

```bash
# Remove test service
kubectl delete namespace test-avahi

# Verify Avahi file was removed
ls -la /etc/avahi/services/ | grep nginx-test
# (Should return nothing)
```

### 21. Remove advertiser (optional)

```bash
# Remove the DaemonSet and RBAC resources
kubectl delete -f k8s-manifests.yaml

# Verify removal
kubectl get daemonset -n kube-system avahi-advertiser
# (Should show: Error from server (NotFound))

# Manually clean up any remaining Avahi files
sudo rm /etc/avahi/services/k8s-*.service
```

## 📋 Quick Reference

### Git Repository
- **Location**: `Services/kube/avahi-advertiser/`
- **Files**:
  - `avahi_k8s_advertiser.py` - Main Python application
  - `Dockerfile` - Container build instructions
  - `requirements.txt` - Python dependencies
  - `k8s-manifests.yaml` - Kubernetes deployment
  - `README.md` - Repository documentation

### Kubernetes Resources
- **Namespace**: `kube-system`
- **DaemonSet**: `avahi-advertiser`
- **ServiceAccount**: `avahi-advertiser`
- **ClusterRole**: `avahi-advertiser`

### Host Integration
- **Avahi Hosts File**: `/etc/avahi/hosts` (A records for LoadBalancer services)
- **Avahi Services Directory**: `/etc/avahi/services/` (service records for NodePort)
- **Service Files**: `/etc/avahi/services/k8s-*.service` (NodePort services only)
- **Volume Mounts**: `hostPath` to both `/etc/avahi/hosts` and `/etc/avahi/services`

### Useful Commands

```bash
# View all advertiser resources
kubectl get all -n kube-system -l app=avahi-advertiser

# Check advertiser logs
kubectl logs -n kube-system -l app=avahi-advertiser --tail=50

# List all LoadBalancer services
kubectl get services --all-namespaces --field-selector spec.type=LoadBalancer

# List all NodePort services
kubectl get services --all-namespaces --field-selector spec.type=NodePort

# Check Avahi A records (LoadBalancer services)
cat /etc/avahi/hosts
grep "Managed by k8s-avahi-advertiser" /etc/avahi/hosts

# List Avahi service files (NodePort services)
ls -la /etc/avahi/services/k8s-*.service

# Test mDNS resolution
avahi-resolve -n <service-name>.local

# Browse all advertised services
avahi-browse -a -t

# Test service connectivity
curl http://<service-name>.local:<port>/

# Rebuild and redeploy
docker build -t localhost/avahi-advertiser:latest .
kubectl rollout restart daemonset/avahi-advertiser -n kube-system

# Update repository
cd Services/kube/avahi-advertiser/
git add .
git commit -m "Description"
git push
```

## 🔧 Troubleshooting

### Pods not starting

```bash
# Check pod status and events
kubectl describe daemonset avahi-advertiser -n kube-system
kubectl describe pods -n kube-system -l app=avahi-advertiser

# Check pod logs for errors
kubectl logs -n kube-system -l app=avahi-advertiser --previous
```

**Common issues:**
- Image not found: Rebuild with `docker build -t localhost/avahi-advertiser:latest .`
- RBAC permissions: Verify ServiceAccount and ClusterRoleBinding are created

### Avahi records not being created

```bash
# Check advertiser logs
kubectl logs -n kube-system -l app=avahi-advertiser --tail=100

# Verify hosts file mount
kubectl exec -n kube-system -it $(kubectl get pod -n kube-system -l app=avahi-advertiser -o jsonpath='{.items[0].metadata.name}') -- cat /etc/avahi/hosts

# Verify services directory mount (for NodePort)
kubectl exec -n kube-system -it $(kubectl get pod -n kube-system -l app=avahi-advertiser -o jsonpath='{.items[0].metadata.name}') -- ls -la /etc/avahi/services/

# Check file/directory permissions on host
ls -la /etc/avahi/hosts /etc/avahi/services/
# Should be writable by root
```

**Common issues:**
- Permission denied: Ensure `/etc/avahi/hosts` and `/etc/avahi/services/` are writable
- Services not LoadBalancer or NodePort: Only these types are advertised
- No LoadBalancer IP: LoadBalancer service must have an assigned IP from MetalLB
- No NodePort assigned: NodePort service must have nodePort defined

### mDNS resolution not working

```bash
# Check if Avahi daemon is running
systemctl status avahi-daemon

# Restart Avahi daemon if needed
sudo systemctl restart avahi-daemon

# Verify Avahi can see the hosts
avahi-browse -a -t

# Check A records are being read
avahi-resolve -n <hostname>.local

# For NodePort services, check if service file is valid XML
xmllint --noout /etc/avahi/services/k8s-*.service 2>/dev/null || echo "No service files"
```

**Common issues:**
- Firewall blocking mDNS: Ensure UDP port 5353 is open
- Invalid XML (NodePort): Check service file syntax
- Invalid hosts format: Ensure A records follow format: `IP hostname.local # comment`
- Network not configured for mDNS: Ensure Avahi is configured properly

### Services not being advertised

```bash
# Check if service has LoadBalancer or NodePort type
kubectl get service <service-name> -n <namespace>

# For LoadBalancer: Verify service has external IP
kubectl get service <service-name> -n <namespace> -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# For NodePort: Verify service has nodePort
kubectl get service <service-name> -n <namespace> -o jsonpath='{.spec.ports[0].nodePort}'

# Check if advertisement is disabled
kubectl get service <service-name> -n <namespace> -o jsonpath='{.metadata.annotations.avahi\.local/enabled}'

# Watch advertiser logs
kubectl logs -n kube-system -l app=avahi-advertiser -f
```

### Git push fails

```bash
# Ensure you're authenticated
git config --global user.name "Your Name"
git config --global user.email "your@email.com"

# For HTTPS, you may need a personal access token
# Generate one at: https://github.com/settings/tokens

# Update remote URL with token (if needed)
git remote set-url origin https://YOUR_TOKEN@github.com/YOUR_USERNAME/k8s-avahi-advertiser.git
```

## 🔒 Production Considerations

### Security
- **Limited Permissions**: ClusterRole only allows read access to services
- **hostPath Mount**: Required but limited to `/etc/avahi/services/`
- **Network Exposure**: mDNS broadcasts service availability on local network
- **No Secrets**: Application doesn't access or store any secrets

### Performance
- **Resource Usage**: Very light (64Mi RAM, 50m CPU typical)
- **Watch Efficiency**: Uses Kubernetes watch API for real-time updates
- **File I/O**: Minimal - only writes when services change

### Reliability
- **DaemonSet**: Runs on all nodes (typically just one with driver=none)
- **Automatic Recovery**: Kubernetes restarts pod if it crashes
- **State Management**: Stateless - rebuilds from current cluster state on restart
- **Cleanup**: Automatically removes Avahi files when services deleted

### Monitoring
- **Logs**: Application logs all advertisement operations
- **Health**: Pod status indicates if watcher is running
- **Metrics**: Consider adding Prometheus metrics for production

### Limitations
- **driver=none only**: Designed for minikube without VM isolation
- **Single host Avahi**: Assumes Avahi runs on Kubernetes host
- **Local network**: mDNS only works on local network segments
- **No TLS**: Basic HTTP advertisement (add HTTPS via annotations)

### Scaling Considerations
- **Multiple nodes**: DaemonSet runs on each node, but all access same `/etc/avahi/services/`
- **Service churn**: High service creation/deletion rate is handled efficiently
- **Large clusters**: Watches all services; consider namespace filtering for very large clusters

### Best Practices
- **Use annotations**: Explicitly configure important services
- **Namespace strategy**: Consider advertising only specific namespaces
- **Name uniqueness**: Ensure service names don't conflict across namespaces
- **Documentation**: Document which services are advertised in your infrastructure
- **Testing**: Test mDNS resolution after deploying new services

### Future Enhancements
- **Namespace filtering**: Only watch specific namespaces
- **Custom annotations**: More flexible service type detection
- **Metrics endpoint**: Prometheus metrics for monitoring
- **Health checks**: Readiness/liveness probes
- **Multiple ports**: Advertise all service ports, not just first one

---

*Guide tested with Kubernetes 1.28+, Minikube 1.32+, and Avahi 0.8+*
