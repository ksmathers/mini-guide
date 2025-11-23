---
title: mgk-owlbear-rodeo
description: 
published: true
date: 2025-11-22T00:00:00.000Z
tags: 
editor: markdown
dateCreated: 2025-11-22T00:00:00.000Z
---

# Owlbear Rodeo Legacy on Kubernetes (Mini Guide)

This guide deploys Owlbear Rodeo Legacy, a virtual tabletop (VTT) application for running online tabletop RPG sessions. It consists of a React frontend and a Node.js backend with WebRTC support for peer-to-peer connections.

## 📋 About Owlbear Rodeo Legacy

Owlbear Rodeo Legacy (v1.0) is the original version of Owlbear Rodeo, released as open-source for personal, non-commercial use. It features:

- **Virtual Tabletop**: Share battle maps, tokens, and drawings in real-time
- **Peer-to-Peer Architecture**: Uses WebRTC for client-side image sharing
- **No Account Required**: Simple session-based gameplay
- **Client-Side Storage**: All user data stored in browser IndexedDB
- **3D Dice Roller**: Physics-driven dice rolling
- **Fog of War**: Dynamic fog and drawing tools

**Note**: This is a legacy application. Consider Owlbear Rodeo 2.0 for production use with cloud features.

## ⚙️ Configuration Variables

Set these environment variables to customize your deployment:

```bash
# Docker Hub configuration
export DOCKER_USERNAME="your-dockerhub-username"
export IMAGE_TAG="latest"  # or use a version tag like "v1.10.2"

# Application configuration
export APP_NAME="owlbear-rodeo"
export NAMESPACE="owlbear-rodeo"
export FRONTEND_PORT=3000
export BACKEND_PORT=9000

# Domain configuration (for ingress/loadbalancer)
export DOMAIN="owlbear.example.com"

# Backend configuration
export ALLOW_ORIGIN="http://localhost:${FRONTEND_PORT}|https://${DOMAIN}"

# Verify variables are set
echo "Docker Username: $DOCKER_USERNAME"
echo "App Name: $APP_NAME"
echo "Namespace: $NAMESPACE"
echo "Domain: $DOMAIN"
```

## ✅ Step 1: Build and Push Docker Images

### 1.1. Clone repository and build images
```bash
# Clone the Owlbear Rodeo Legacy repository
git clone https://github.com/owlbear-rodeo/owlbear-rodeo-legacy.git
cd owlbear-rodeo-legacy

# Login to Docker Hub
docker login -u $DOCKER_USERNAME

# Build the backend image using the provided Dockerfile
cd backend
docker build -t ${DOCKER_USERNAME}/owlbear-rodeo-backend:${IMAGE_TAG} .
docker push ${DOCKER_USERNAME}/owlbear-rodeo-backend:${IMAGE_TAG}

# Build the frontend image (we'll use the root Dockerfile)
cd ..
docker build -t ${DOCKER_USERNAME}/owlbear-rodeo-frontend:${IMAGE_TAG} .
docker push ${DOCKER_USERNAME}/owlbear-rodeo-frontend:${IMAGE_TAG}

# Verify images were pushed
echo "Backend image: ${DOCKER_USERNAME}/owlbear-rodeo-backend:${IMAGE_TAG}"
echo "Frontend image: ${DOCKER_USERNAME}/owlbear-rodeo-frontend:${IMAGE_TAG}"

# Return to your working directory
cd ..
```

**Note on Frontend Build**: The root Dockerfile is intended for development. For production, you may want to create a custom production Dockerfile that:
- Builds the React app with `yarn build`
- Serves the static files with nginx

**Alternative**: If you prefer to use my pre-built images, you can skip this step and use:
```bash
export DOCKER_USERNAME="ksmathers"  # or use public images if available
```

## ✅ Step 2: Setup Namespace and Configuration

### 2.1. Create namespace and configmap
```bash
# Create dedicated namespace
kubectl create namespace $NAMESPACE
kubectl config set-context --current --namespace=$NAMESPACE

# Create ConfigMap for backend configuration
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: ${APP_NAME}-backend-config
  namespace: $NAMESPACE
data:
  ALLOW_ORIGIN: "$ALLOW_ORIGIN"
  PORT: "$BACKEND_PORT"
EOF

# Verify namespace and configmap
kubectl get namespace $NAMESPACE
kubectl get configmap -n $NAMESPACE
```

## ✅ Step 3: Deploy Backend Service

### 3.1. Create backend deployment and service
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}-backend
  namespace: $NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${APP_NAME}-backend
  template:
    metadata:
      labels:
        app: ${APP_NAME}-backend
    spec:
      containers:
      - name: backend
        image: ${DOCKER_USERNAME}/owlbear-rodeo-backend:${IMAGE_TAG}
        envFrom:
        - configMapRef:
            name: ${APP_NAME}-backend-config
        ports:
        - containerPort: $BACKEND_PORT
          name: backend
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: $BACKEND_PORT
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: $BACKEND_PORT
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-backend
  namespace: $NAMESPACE
spec:
  selector:
    app: ${APP_NAME}-backend
  ports:
  - port: $BACKEND_PORT
    targetPort: $BACKEND_PORT
    name: backend
  type: ClusterIP
EOF
```

### 3.2. Verify backend is running
```bash
# Check deployment status
kubectl get deployments -n $NAMESPACE
kubectl get pods -n $NAMESPACE

# Check backend logs
kubectl logs -n $NAMESPACE -l app=${APP_NAME}-backend --tail=50

# Wait for backend to be ready
kubectl wait --for=condition=available --timeout=300s deployment/${APP_NAME}-backend -n $NAMESPACE
```

## ✅ Step 4: Deploy Frontend Service

### 4.1. Create frontend deployment and service
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}-frontend
  namespace: $NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${APP_NAME}-frontend
  template:
    metadata:
      labels:
        app: ${APP_NAME}-frontend
    spec:
      containers:
      - name: frontend
        image: ${DOCKER_USERNAME}/owlbear-rodeo-frontend:${IMAGE_TAG}
        env:
        - name: REACT_APP_BROKER_URL
          value: "http://${APP_NAME}-backend:${BACKEND_PORT}"
        - name: REACT_APP_VERSION
          value: "1.10.2-legacy"
        ports:
        - containerPort: $FRONTEND_PORT
          name: http
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /
            port: $FRONTEND_PORT
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: $FRONTEND_PORT
          initialDelaySeconds: 30
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-frontend
  namespace: $NAMESPACE
spec:
  selector:
    app: ${APP_NAME}-frontend
  ports:
  - port: 80
    targetPort: $FRONTEND_PORT
    name: http
  type: ClusterIP
EOF
```

### 4.2. Verify frontend is running
```bash
# Check deployment status
kubectl get deployments -n $NAMESPACE
kubectl get pods -n $NAMESPACE

# Check frontend logs
kubectl logs -n $NAMESPACE -l app=${APP_NAME}-frontend --tail=50

# Wait for frontend to be ready
kubectl wait --for=condition=available --timeout=600s deployment/${APP_NAME}-frontend -n $NAMESPACE
```

## ✅ Step 5: Expose Service Externally

You have two options for exposing the service: **LoadBalancer (MetalLB)** or **Ingress (nginx)**. Choose the appropriate method for your setup.

### Option A: Using LoadBalancer with MetalLB (Recommended if you have MetalLB)

### 5.1a. Update frontend service to use LoadBalancer
```bash
# Patch the frontend service to type LoadBalancer
kubectl patch service ${APP_NAME}-frontend -n $NAMESPACE -p '{"spec":{"type":"LoadBalancer"}}'

# Alternatively, delete and recreate the service
kubectl delete service ${APP_NAME}-frontend -n $NAMESPACE

kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-frontend
  namespace: $NAMESPACE
  annotations:
    metallb.universe.tf/allow-shared-ip: "owlbear-rodeo-shared"
spec:
  type: LoadBalancer
  selector:
    app: ${APP_NAME}-frontend
  ports:
  - port: 80
    targetPort: $FRONTEND_PORT
    name: http
EOF
```

### 5.2a. Get the LoadBalancer IP address
```bash
# Wait for MetalLB to assign an IP
kubectl get service ${APP_NAME}-frontend -n $NAMESPACE -w

# Get the assigned external IP
export EXTERNAL_IP=$(kubectl get service ${APP_NAME}-frontend -n $NAMESPACE -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Owlbear Rodeo is accessible at: http://${EXTERNAL_IP}"

# Verify the service is accessible
curl -I http://${EXTERNAL_IP}
```

**Note:** You'll need to update the `ALLOW_ORIGIN` environment variable in the backend to include the LoadBalancer IP:

```bash
# Update ALLOW_ORIGIN to include the external IP
kubectl patch configmap ${APP_NAME}-backend-config -n $NAMESPACE --type merge -p "{\"data\":{\"ALLOW_ORIGIN\":\"http://localhost:${FRONTEND_PORT}|http://${EXTERNAL_IP}\"}}"

# Restart backend to pick up the change
kubectl rollout restart deployment/${APP_NAME}-backend -n $NAMESPACE
```

### Option B: Using Ingress with nginx-ingress-controller

**Prerequisites:** Requires nginx-ingress-controller to be deployed in your cluster.

### 5.1b. Create ingress for external access
```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${APP_NAME}-ingress
  namespace: $NAMESPACE
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    # Add SSL redirect if using HTTPS
    # nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: $DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${APP_NAME}-frontend
            port:
              number: 80
  # Uncomment for HTTPS
  # tls:
  # - hosts:
  #   - $DOMAIN
  #   secretName: ${APP_NAME}-tls
EOF
```

### 5.2b. Verify ingress is configured
```bash
# Check ingress status
kubectl get ingress -n $NAMESPACE

# Get ingress IP/hostname
kubectl get ingress ${APP_NAME}-ingress -n $NAMESPACE -o jsonpath='{.status.loadBalancer.ingress[0]}'

# Update ALLOW_ORIGIN to include your domain
kubectl patch configmap ${APP_NAME}-backend-config -n $NAMESPACE --type merge -p "{\"data\":{\"ALLOW_ORIGIN\":\"http://localhost:${FRONTEND_PORT}|https://${DOMAIN}\"}}"

# Restart backend
kubectl rollout restart deployment/${APP_NAME}-backend -n $NAMESPACE
```

## ✅ Step 6: Configure STUN/TURN Servers (Optional)

For WebRTC to work across different networks, you may need to configure STUN/TURN servers.

### 6.1. Create ice.json ConfigMap
```bash
kubectl create configmap ${APP_NAME}-ice-config \
  -n $NAMESPACE \
  --from-literal=ice.json='[
    {
      "urls": "stun:stun.l.google.com:19302"
    }
  ]'

# Verify ConfigMap
kubectl get configmap ${APP_NAME}-ice-config -n $NAMESPACE -o yaml
```

### 6.2. Update backend deployment to use ice.json
```bash
# Patch the backend deployment to mount the ConfigMap
kubectl patch deployment ${APP_NAME}-backend -n $NAMESPACE --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/volumeMounts",
    "value": [{
      "name": "ice-config",
      "mountPath": "/app/backend/ice.json",
      "subPath": "ice.json"
    }]
  },
  {
    "op": "add",
    "path": "/spec/template/spec/volumes",
    "value": [{
      "name": "ice-config",
      "configMap": {
        "name": "'${APP_NAME}'-ice-config"
      }
    }]
  }
]'
```

## 📊 Monitoring and Management

### Check application status
```bash
# View all resources
kubectl get all -n $NAMESPACE

# Check pod health
kubectl get pods -n $NAMESPACE -w

# View logs for frontend
kubectl logs -n $NAMESPACE -l app=${APP_NAME}-frontend --tail=100 -f

# View logs for backend
kubectl logs -n $NAMESPACE -l app=${APP_NAME}-backend --tail=100 -f
```

### Access the application
```bash
# Port-forward for local testing (if no ingress)
kubectl port-forward -n $NAMESPACE svc/${APP_NAME}-frontend 8080:80

# Then access at http://localhost:8080
```

## 🔧 Troubleshooting

### Frontend not connecting to backend
```bash
# Check backend service DNS
kubectl run -it --rm debug --image=busybox --restart=Never -n $NAMESPACE -- \
  nslookup ${APP_NAME}-backend

# Verify backend is responding
kubectl exec -it -n $NAMESPACE deployment/${APP_NAME}-backend -- \
  wget -O- http://localhost:${BACKEND_PORT}/health
```

### Custom images not showing on other computers
This is usually due to WebRTC connection issues. You need to configure STUN/TURN servers (see Step 5).

Common free STUN servers:
- `stun:stun.l.google.com:19302`
- `stun:stun1.l.google.com:19302`
- `stun:stun2.l.google.com:19302`

For TURN servers (required for restrictive networks), consider:
- [Twilio TURN](https://www.twilio.com/stun-turn)
- [Xirsys](https://xirsys.com/)
- Self-hosted [coturn](https://github.com/coturn/coturn)

### Pod memory issues
```bash
# The build process requires significant memory
# Increase limits if pods are being OOMKilled

kubectl patch deployment ${APP_NAME}-frontend -n $NAMESPACE --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/resources/limits/memory",
    "value": "3Gi"
  }
]'
```

## 🗑️ Cleanup

### Remove all resources
```bash
# Delete namespace and all resources
kubectl delete namespace $NAMESPACE

# Or delete individual resources
kubectl delete ingress ${APP_NAME}-ingress -n $NAMESPACE
kubectl delete service ${APP_NAME}-frontend ${APP_NAME}-backend -n $NAMESPACE
kubectl delete deployment ${APP_NAME}-frontend ${APP_NAME}-backend -n $NAMESPACE
kubectl delete configmap ${APP_NAME}-backend-config ${APP_NAME}-ice-config -n $NAMESPACE
```

## 📚 Additional Notes

### Memory Requirements
- **Frontend (Development Image)**: Requires 256-512MB at runtime (runs `yarn start`)
- **Frontend (Production Image with nginx)**: Requires ~50-100MB at runtime
- **Backend**: Requires ~128-256MB at runtime
- **Docker Build**: At least 5GB memory allocated to Docker when building images locally

### Performance Considerations
- Consider using a production-ready build with multi-stage Docker builds
- Use persistent volume for build cache to speed up restarts
- Enable horizontal pod autoscaling for high-traffic scenarios

### Security Notes
- **Personal Use Only**: This is licensed for non-commercial use
- **No Authentication**: Sessions are not password protected by default
- **Client-Side Data**: All maps/tokens stored in user's browser
- **WebRTC Security**: Peer-to-peer connections bypass server for image transfer

### Alternative Deployment Options

#### Using Docker Compose (Simpler for Development)
```bash
git clone https://github.com/owlbear-rodeo/owlbear-rodeo-legacy.git
cd owlbear-rodeo-legacy
docker-compose up
```

### Building Production Images

The repository includes Dockerfiles for both backend and frontend, but the frontend Dockerfile is designed for development. For production, consider creating an optimized frontend image:

#### Production Frontend Dockerfile
Create a `Dockerfile.production` in the repository root:

```dockerfile
# Build stage
FROM node:16.20.0-alpine3.18 AS builder

WORKDIR /app

# Copy package files
COPY package.json yarn.lock ./
RUN yarn install --non-interactive --frozen-lockfile

# Copy source code
COPY public ./public
COPY src ./src
COPY tsconfig.json ./

# Build the application
RUN yarn build

# Production stage - serve with nginx
FROM nginx:alpine

# Copy built assets from builder
COPY --from=builder /app/build /usr/share/nginx/html

# Copy nginx configuration
RUN echo 'server { \
    listen 3000; \
    location / { \
        root /usr/share/nginx/html; \
        index index.html; \
        try_files $uri $uri/ /index.html; \
    } \
}' > /etc/nginx/conf.d/default.conf

EXPOSE 3000

CMD ["nginx", "-g", "daemon off;"]
```

Build and push the production image:
```bash
docker build -f Dockerfile.production -t ${DOCKER_USERNAME}/owlbear-rodeo-frontend:${IMAGE_TAG} .
docker push ${DOCKER_USERNAME}/owlbear-rodeo-frontend:${IMAGE_TAG}
```

#### Backend Dockerfile Details
The backend Dockerfile in the repository already follows best practices:
- Multi-stage build (builder, install, runtime)
- Production dependencies only
- Non-root user
- Minimal alpine base image

#### Updating Images
To update your deployment with new images:
```bash
# Rebuild and push images
cd owlbear-rodeo-legacy
docker build -t ${DOCKER_USERNAME}/owlbear-rodeo-backend:${IMAGE_TAG} ./backend
docker push ${DOCKER_USERNAME}/owlbear-rodeo-backend:${IMAGE_TAG}

# Restart deployments to pull new images
kubectl rollout restart deployment/${APP_NAME}-backend -n $NAMESPACE
kubectl rollout restart deployment/${APP_NAME}-frontend -n $NAMESPACE

# Monitor rollout
kubectl rollout status deployment/${APP_NAME}-backend -n $NAMESPACE
kubectl rollout status deployment/${APP_NAME}-frontend -n $NAMESPACE
```

## 🔗 References

- [Owlbear Rodeo Legacy GitHub](https://github.com/owlbear-rodeo/owlbear-rodeo-legacy)
- [Owlbear Rodeo 2.0](https://owlbear.rodeo/) - Current production version
- [WebRTC Documentation](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API)
- [STUN/TURN Server Setup](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols)

---

**License**: This project is for personal use only. No commercial use is permitted.
**Created**: November 22, 2025
