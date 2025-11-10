---
title: mgk-mosquitto
description: 
published: true
date: 2025-11-08T00:00:00.000Z
tags: 
editor: markdown
dateCreated: 2025-11-08T00:00:00.000Z
---

# Mosquitto MQTT Broker on Kubernetes (Mini Guide)

This guide deploys Eclipse Mosquitto MQTT broker on Kubernetes with persistent storage, authentication, and both standard (1883) and WebSocket (9001) support.

## ✅ Step 1: Create Namespace and Setup

### 1. Create a dedicated namespace
```bash
kubectl create namespace mosquitto
kubectl config set-context --current --namespace=mosquitto
```

### 2. Verify cluster access
```bash
kubectl cluster-info
kubectl get nodes
```

## ✅ Step 2: Create Persistent Storage

### 3. Create PersistentVolumeClaims for data and config storage
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mosquitto-data-pvc
  namespace: mosquitto
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mosquitto-log-pvc
  namespace: mosquitto
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
EOF
```

### 4. Verify PVCs are created
```bash
kubectl get pvc -n mosquitto
```

## ✅ Step 3: Configuration Setup

### 5. Create Mosquitto configuration
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: mosquitto-config
  namespace: mosquitto
data:
  mosquitto.conf: |
    # Persistence configuration
    persistence true
    persistence_location /mosquitto/data/
    
    # Logging
    log_dest file /mosquitto/log/mosquitto.log
    log_dest stdout
    log_type all
    
    # Standard MQTT listener
    listener 1883
    protocol mqtt
    
    # WebSocket listener
    listener 9001
    protocol websockets
    
    # Authentication (optional - uncomment to enable)
    # allow_anonymous false
    # password_file /mosquitto/config/passwd
    
    # Allow anonymous by default for testing
    allow_anonymous true
EOF
```

### 6. Create password file (Optional - for authentication)
```bash
# Skip this step if allowing anonymous connections
# To enable authentication, create a password file:

# Set environment variables for MQTT credentials
export MQTT_USERNAME="mqttuser"
export MQTT_PASSWORD="sample password"

# Create temporary pod to generate password file with your credentials
kubectl run -n mosquitto mosquitto-passwd --image=eclipse-mosquitto:2 \
  --rm -i --tty --restart=Never -- sh -c \
  "mosquitto_passwd -c -b /tmp/passwd $MQTT_USERNAME $MQTT_PASSWORD && cat /tmp/passwd" > passwd.tmp

# Create secret from password file
kubectl create secret generic mosquitto-passwd -n mosquitto \
  --from-file=passwd=passwd.tmp

# Clean up temporary file
rm passwd.tmp

# Verify secret was created
kubectl describe secret mosquitto-passwd -n mosquitto

# Update ConfigMap to enable authentication (uncomment the auth lines in Step 5)
```

## ✅ Step 4: Deploy Mosquitto

### 7. Deploy Mosquitto broker
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mosquitto
  namespace: mosquitto
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mosquitto
  template:
    metadata:
      labels:
        app: mosquitto
    spec:
      containers:
      - name: mosquitto
        image: eclipse-mosquitto:2
        ports:
        - containerPort: 1883
          name: mqtt
          protocol: TCP
        - containerPort: 9001
          name: websocket
          protocol: TCP
        volumeMounts:
        - name: config
          mountPath: /mosquitto/config/mosquitto.conf
          subPath: mosquitto.conf
        - name: data
          mountPath: /mosquitto/data
        - name: log
          mountPath: /mosquitto/log
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
      volumes:
      - name: config
        configMap:
          name: mosquitto-config
      - name: data
        persistentVolumeClaim:
          claimName: mosquitto-data-pvc
      - name: log
        persistentVolumeClaim:
          claimName: mosquitto-log-pvc
EOF
```

### 8. Verify deployment
```bash
kubectl get pods -n mosquitto
kubectl logs -f deployment/mosquitto -n mosquitto
```

## ✅ Step 5: Expose Service

### 9. Create Service for MQTT access
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: mosquitto-service
  namespace: mosquitto
spec:
  type: LoadBalancer
  selector:
    app: mosquitto
  ports:
  - name: mqtt
    port: 1883
    targetPort: 1883
    protocol: TCP
  - name: websocket
    port: 9001
    targetPort: 9001
    protocol: TCP
EOF
```

### 10. Get service details and access information
```bash
kubectl get svc mosquitto-service -n mosquitto
```

**Note**: If using a local cluster (minikube, k3s, etc.) without a LoadBalancer, the EXTERNAL-IP will show `<pending>`. Use one of these alternatives:

#### Option A: NodePort (recommended for local clusters)
```bash
# Change service type to NodePort
kubectl patch svc mosquitto-service -n mosquitto -p '{"spec": {"type": "NodePort"}}'

# Get the NodePort assignments
kubectl get svc mosquitto-service -n mosquitto

# Access using node IP and NodePort
# Example output shows ports like: 1883:30123/TCP, 9001:30456/TCP
# Use: <NODE_IP>:<NodePort> to connect

# Get node IP
kubectl get nodes -o wide

# Example connection:
# mosquitto_pub -h <NODE_IP> -p <NodePort_for_1883> -t "test/topic" -m "Hello"
```

#### Option B: Port Forwarding (for testing/development)
```bash
# Forward ports to localhost
kubectl port-forward -n mosquitto svc/mosquitto-service 1883:1883 9001:9001

# In another terminal, connect to localhost
# mosquitto_pub -h localhost -p 1883 -t "test/topic" -m "Hello"
```

#### Option C: Get LoadBalancer IP (cloud environments)
```bash
# Wait for external IP assignment (may take a minute)
kubectl get svc mosquitto-service -n mosquitto -w

# Once assigned, get the IP
MQTT_HOST=$(kubectl get svc mosquitto-service -n mosquitto -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "MQTT Host: $MQTT_HOST"
```

## ✅ Step 6: Testing the MQTT Broker

### 11. Install MQTT client tools (on your local machine)
```bash
# macOS
brew install mosquitto

# Linux (Debian/Ubuntu)
sudo apt-get install mosquitto-clients

# Linux (RHEL/CentOS)
sudo yum install mosquitto
```

### 12. Test MQTT connection
```bash
# Get the service endpoint
MQTT_HOST=$(kubectl get svc mosquitto-service -n mosquitto -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# If using port-forward or NodePort, use localhost
MQTT_HOST="localhost"

# Subscribe to test topic (in one terminal)
mosquitto_sub -h $MQTT_HOST -p 1883 -t "test/topic" -v

# Publish to test topic (in another terminal)
mosquitto_pub -h $MQTT_HOST -p 1883 -t "test/topic" -m "Hello from Kubernetes MQTT!"
```

### 13. Test with authentication (if enabled)
```bash
# Subscribe with username/password
mosquitto_sub -h $MQTT_HOST -p 1883 -t "test/topic" -u mqttuser -P mqttpassword -v

# Publish with username/password
mosquitto_pub -h $MQTT_HOST -p 1883 -t "test/topic" -m "Authenticated message" -u mqttuser -P mqttpassword
```

## 📊 Monitoring and Maintenance

### View logs
```bash
kubectl logs -f deployment/mosquitto -n mosquitto
```

### Check persistent data
```bash
kubectl exec -n mosquitto deployment/mosquitto -- ls -la /mosquitto/data
kubectl exec -n mosquitto deployment/mosquitto -- tail -n 50 /mosquitto/log/mosquitto.log
```

### Restart broker
```bash
kubectl rollout restart deployment/mosquitto -n mosquitto
```

## 🔧 Advanced Configuration

### Enable TLS/SSL (Production Use)

For production deployments, enable TLS encryption to secure MQTT communications.

#### Prerequisites
You need the following certificate files:
- `mosquitto.domain.com.crt` - Server certificate (PEM format)
- `mosquitto.domain.com.key` - Server private key (PEM format)
- `root-ca.crt` - Root CA certificate (PEM format)

#### Step 1: Create Kubernetes Secret for TLS Certificates

Store the private key and certificates securely in Kubernetes secrets:

```bash
# Create secret containing the server certificate, private key, and CA certificate
kubectl create secret generic mosquitto-tls -n mosquitto \
  --from-file=tls.crt=mosquitto.domain.com.crt \
  --from-file=tls.key=mosquitto.domain.com.key \
  --from-file=ca.crt=root-ca.crt

# Verify secret was created
kubectl describe secret mosquitto-tls -n mosquitto
```

**Security Best Practice**: The private key (`mosquitto.domain.com.key`) is now securely stored in the Kubernetes secrets store and will not be exposed in ConfigMaps or pod specifications.

#### Step 2: Update Mosquitto Configuration for TLS

Update the ConfigMap to enable TLS on ports 8883 (MQTT) and 9002 (WebSocket):

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: mosquitto-config
  namespace: mosquitto
data:
  mosquitto.conf: |
    # Persistence configuration
    persistence true
    persistence_location /mosquitto/data/
    
    # Logging
    log_dest file /mosquitto/log/mosquitto.log
    log_dest stdout
    log_type all
    
    # Standard MQTT listener (non-encrypted - optional, can be removed for security)
    listener 1883
    protocol mqtt
    
    # Secure MQTT listener with TLS
    listener 8883
    protocol mqtt
    cafile /mosquitto/certs/ca.crt
    certfile /mosquitto/certs/tls.crt
    keyfile /mosquitto/certs/tls.key
    require_certificate false
    use_identity_as_username false
    
    # Standard WebSocket listener (non-encrypted - optional, can be removed)
    listener 9001
    protocol websockets
    
    # Secure WebSocket listener with TLS
    listener 9002
    protocol websockets
    cafile /mosquitto/certs/ca.crt
    certfile /mosquitto/certs/tls.crt
    keyfile /mosquitto/certs/tls.key
    require_certificate false
    use_identity_as_username false
    
    # Authentication - recommended with TLS
    allow_anonymous false
    password_file /mosquitto/config/passwd
EOF
```

#### Step 3: Update Deployment to Mount TLS Certificates

Modify the Mosquitto deployment to mount the TLS secret:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mosquitto
  namespace: mosquitto
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mosquitto
  template:
    metadata:
      labels:
        app: mosquitto
    spec:
      containers:
      - name: mosquitto
        image: eclipse-mosquitto:2
        ports:
        - containerPort: 1883
          name: mqtt
          protocol: TCP
        - containerPort: 8883
          name: mqtt-tls
          protocol: TCP
        - containerPort: 9001
          name: websocket
          protocol: TCP
        - containerPort: 9002
          name: websocket-tls
          protocol: TCP
        volumeMounts:
        - name: config
          mountPath: /mosquitto/config/mosquitto.conf
          subPath: mosquitto.conf
        - name: data
          mountPath: /mosquitto/data
        - name: log
          mountPath: /mosquitto/log
        - name: certs
          mountPath: /mosquitto/certs
          readOnly: true
        - name: passwd
          mountPath: /mosquitto/config/passwd
          subPath: passwd
          readOnly: true
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
      volumes:
      - name: config
        configMap:
          name: mosquitto-config
      - name: data
        persistentVolumeClaim:
          claimName: mosquitto-data-pvc
      - name: log
        persistentVolumeClaim:
          claimName: mosquitto-log-pvc
      - name: certs
        secret:
          secretName: mosquitto-tls
          defaultMode: 0440
      - name: passwd
        secret:
          secretName: mosquitto-passwd
          defaultMode: 0440
EOF
```

**Note**: The `defaultMode: 0440` ensures certificates are read-only and accessible only to the container user.

#### Step 4: Update Service to Expose TLS Ports

Update the service to expose the secure MQTT and WebSocket ports:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: mosquitto-service
  namespace: mosquitto
spec:
  type: LoadBalancer
  selector:
    app: mosquitto
  ports:
  - name: mqtt
    port: 1883
    targetPort: 1883
    protocol: TCP
  - name: mqtt-tls
    port: 8883
    targetPort: 8883
    protocol: TCP
  - name: websocket
    port: 9001
    targetPort: 9001
    protocol: TCP
  - name: websocket-tls
    port: 9002
    targetPort: 9002
    protocol: TCP
EOF
```

#### Step 5: Test TLS Connection

Test the secure MQTT connection using the CA certificate:

```bash
# Get the service endpoint
MQTT_HOST=$(kubectl get svc mosquitto-service -n mosquitto -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Subscribe using TLS (with CA verification)
mosquitto_sub -h $MQTT_HOST -p 8883 \
  --cafile root-ca.crt \
  -t "test/secure" \
  -u mqttuser -P mqttpassword \
  -v

# Publish using TLS (in another terminal)
mosquitto_pub -h $MQTT_HOST -p 8883 \
  --cafile root-ca.crt \
  -t "test/secure" \
  -m "Encrypted message!" \
  -u mqttuser -P mqttpassword

# Test with hostname verification (production recommended)
mosquitto_sub -h mosquitto.domain.com -p 8883 \
  --cafile root-ca.crt \
  -t "test/secure" \
  -u mqttuser -P mqttpassword \
  -v
```

#### Verify TLS Configuration

Check that TLS is properly configured:

```bash
# View certificate details in the secret
kubectl get secret mosquitto-tls -n mosquitto -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text

# Check broker logs for TLS initialization
kubectl logs -f deployment/mosquitto -n mosquitto | grep -i tls

# Verify certificate expiration
kubectl get secret mosquitto-tls -n mosquitto -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -enddate
```

#### Certificate Rotation

To rotate certificates before expiration:

```bash
# Delete existing secret
kubectl delete secret mosquitto-tls -n mosquitto

# Create new secret with updated certificates
kubectl create secret generic mosquitto-tls -n mosquitto \
  --from-file=tls.crt=mosquitto.domain.com.crt \
  --from-file=tls.key=mosquitto.domain.com.key \
  --from-file=ca.crt=root-ca.crt

# Restart deployment to pick up new certificates
kubectl rollout restart deployment/mosquitto -n mosquitto
```

### Access Control Lists (ACL)

Create an ACL file for fine-grained topic permissions:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: mosquitto-acl
  namespace: mosquitto
data:
  acl: |
    # User-specific ACLs
    user mqttuser
    topic readwrite test/#
    topic read sensors/#
    
    # Pattern-based ACLs
    pattern readwrite home/%u/#
EOF
```

## 🗑️ Cleanup

### Remove Mosquitto deployment
```bash
kubectl delete namespace mosquitto
```

Or remove individual resources:
```bash
kubectl delete deployment mosquitto -n mosquitto
kubectl delete service mosquitto-service -n mosquitto
kubectl delete configmap mosquitto-config -n mosquitto
kubectl delete pvc mosquitto-data-pvc mosquitto-log-pvc -n mosquitto
```

## 📚 Additional Resources

- [Eclipse Mosquitto Documentation](https://mosquitto.org/documentation/)
- [MQTT Protocol Specification](https://mqtt.org/)
- [Mosquitto Docker Hub](https://hub.docker.com/_/eclipse-mosquitto)
- [MQTT Security Best Practices](https://mosquitto.org/man/mosquitto-conf-5.html)

## 🎯 Quick Reference

| Component | Port | Protocol | Purpose | Security |
|-----------|------|----------|---------|----------|
| MQTT | 1883 | TCP | Standard MQTT broker | Unencrypted |
| WebSocket | 9001 | TCP | MQTT over WebSockets | Unencrypted |
| MQTT TLS | 8883 | TCP | Secure MQTT with TLS | Encrypted ⭐ |
| WebSocket TLS | 9002 | TCP | Secure WebSocket with TLS | Encrypted ⭐ |

**Note**: For production use, configure TLS (ports 8883/9002) and disable unencrypted ports (1883/9001).

### Common MQTT Client Commands
```bash
# Subscribe to all topics
mosquitto_sub -h <host> -p 1883 -t "#" -v

# Subscribe to specific topic pattern
mosquitto_sub -h <host> -p 1883 -t "sensors/+/temperature" -v

# Publish retained message
mosquitto_pub -h <host> -p 1883 -t "status/online" -m "true" -r

# Publish with QoS 2
mosquitto_pub -h <host> -p 1883 -t "critical/alert" -m "High priority" -q 2
```
