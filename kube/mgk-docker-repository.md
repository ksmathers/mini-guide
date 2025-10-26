# Minimal Docker Registry with Read-Through Cache on Kubernetes

This guide sets up a private Docker registry on Kubernetes with read-through caching of Docker Hub images.

## ✅ Step 1: Create a Namespace

This is required if this is the first time you are creating a ConfigMap.  ConfigMap objects are used for passing 
insecure configuration information to deployed services.  They are not suitable for storing secrets. 

```
kubectl create namespace registry
```

## ✅ Step 2: Create a ConfigMap for Registry Configuration

Create a standard registry that supports pushes and pulls:

```
kubectl apply -n registry -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: registry-config
data:
  config.yml: |
    version: 0.1
    storage:
      filesystem:
        rootdirectory: /var/lib/registry
      cache:
        blobdescriptor: inmemory
    http:
      addr: :5000
EOF
```

**Note:** This configuration supports push/pull of private images. For Docker Hub caching, see Step 2b for a separate registry approach.

## ✅ Step 2b: Docker Hub Pull-Through Cache (Alternative)

**Important:** Docker registry proxy mode is read-only and conflicts with pushes. For both capabilities, you need separate registries or use Docker Hub directly.

For pull-through caching only, replace the ConfigMap with:

```
kubectl apply -n registry -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: registry-config
data:
  config.yml: |
    version: 0.1
    storage:
      filesystem:
        rootdirectory: /var/lib/registry
      cache:
        blobdescriptor: inmemory
    proxy:
      remoteurl: https://registry-1.docker.io
    http:
      addr: :5000
EOF
```

**Limitation:** This mode only supports pulls from Docker Hub (`library/*` namespace). No pushes allowed.

## ✅ Step 3: Deploy the Registry (Deployment + PVC + Service)

This creates a persistent volume with 10GB of available space together with the service
deployment.  A deployment creates one or more pods that host your service application.
Last it exposes a service port, mapping it to port 30000 to avoid conflicts.

```
kubectl apply -n registry -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry-cache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry-cache
  template:
    metadata:
      labels:
        app: registry-cache
    spec:
      containers:
        - name: registry
          image: registry:2
          volumeMounts:
            - name: config
              mountPath: /etc/docker/registry
            - name: storage
              mountPath: /var/lib/registry
          ports:
            - containerPort: 5000
      volumes:
        - name: config
          configMap:
            name: registry-config
        - name: storage
          persistentVolumeClaim:
            claimName: registry-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: registry-cache
spec:
  selector:
    app: registry-cache
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
      nodePort: 30000
  type: NodePort
EOF
```

✅ This exposes the registry at `http://<node-ip>:30000`.

## ✅ Step 4: Allow Insecure Registry on Client Machine (Development Only)

Edit `/etc/docker/daemon.json` on your development machine:

```json
{
  "insecure-registries": ["<NODE_IP>:30000"]
}
```

Restart Docker:

```
sudo systemctl restart docker
```

## ✅ Step 4b: Configure Colima for Insecure Registry (macOS)

If using Colima on macOS, edit the Colima config:

```
colima stop
```

Edit `~/.colima/default/colima.yaml` and add:

```yaml
docker:
  insecure-registries:
    - <NODE_IP>:30000
```

Restart Colima:

```
colima start
```

## ✅ Step 5: Test Private Image Push/Pull

```
NODE_IP=<NODE_IP>
echo -e "FROM alpine:latest\nRUN echo 'k8s test' > /message.txt" > Dockerfile
docker build -t $NODE_IP:30000/myproject/mytest:latest .
docker push $NODE_IP:30000/myproject/mytest:latest
docker rmi $NODE_IP:30000/myproject/mytest:latest
docker pull $NODE_IP:30000/myproject/mytest:latest
```

## ✅ Step 6: Test with Docker Hub Images (Optional)

If using pull-through cache config (Step 2b):

```
NODE_IP=<NODE_IP>
docker pull $NODE_IP:30000/library/alpine:latest
docker pull $NODE_IP:30000/library/alpine:latest  # cached now, faster
```

## 🎉 You're Done!

✅ Kubernetes-hosted registry  
✅ Private image storage and pushes  
✅ Optional: Pull-through cache (Step 2b, read-only)

## 📌 Next Steps (Optional)

| Enhancement | Method |
|------------|--------|
| TLS | Use Ingress with certs |
| Authentication | Enable htpasswd in config |
| External exposure | Use LoadBalancer or Ingress |