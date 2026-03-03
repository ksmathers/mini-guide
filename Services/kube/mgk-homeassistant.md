---
title: mgk-homeassistant
description: 
published: true
date: 2025-10-26T18:00:00.000Z
tags: 
editor: markdown
dateCreated: 2025-10-26T18:00:00.000Z
---

# HomeAssistant on Kubernetes with Bluetooth (Mini Guide)

This guide deploys HomeAssistant on Kubernetes with privileged access to the host Bluetooth stack. Designed for minikube with driver=none.

## ✅ Step 1: Prerequisites and Setup

### 1. Verify minikube is running with driver=none
```bash
minikube status
minikube profile list
```

> **Note**: Driver=none runs Kubernetes directly on the host, allowing direct hardware access necessary for Bluetooth.

### 1a. Verify MetalLB is installed and configured
```bash
# Check MetalLB is running
kubectl get pods -n metallb-system

# Verify MetalLB has an IP pool configured
kubectl get ipaddresspool -n metallb-system
```

> **Note**: MetalLB provides LoadBalancer IPs, avoiding port conflicts with NodePort services. See the MetalLB setup guide if not yet installed.

### 2. Create a dedicated namespace
```bash
kubectl create namespace homeassistant
kubectl config set-context --current --namespace=homeassistant
```

### 3. Verify cluster access
```bash
kubectl cluster-info
kubectl get nodes
```

## ✅ Step 2: Create Persistent Storage

### 4. Create PersistentVolumeClaim for HomeAssistant configuration
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: homeassistant-config-pvc
  namespace: homeassistant
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
EOF
```

### 5. Verify PVC is created
```bash
kubectl get pvc -n homeassistant
```

## ✅ Step 3: Deploy HomeAssistant with Bluetooth Access

### 6. Deploy HomeAssistant with privileged security context
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: homeassistant
  namespace: homeassistant
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: homeassistant
  template:
    metadata:
      labels:
        app: homeassistant
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: homeassistant
        image: ghcr.io/home-assistant/home-assistant:stable
        securityContext:
          privileged: true
        ports:
        - containerPort: 8123
          protocol: TCP
        env:
        - name: TZ
          value: "America/New_York"  # Adjust to your timezone
        volumeMounts:
        - name: config
          mountPath: /config
        - name: dbus
          mountPath: /run/dbus
          readOnly: true
        - name: bluetooth
          mountPath: /sys/class/bluetooth
          readOnly: true
        - name: dev-bus-usb
          mountPath: /dev/bus/usb
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /
            port: 8123
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /
            port: 8123
            scheme: HTTP
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
      volumes:
      - name: config
        persistentVolumeClaim:
          claimName: homeassistant-config-pvc
      - name: dbus
        hostPath:
          path: /run/dbus
          type: Directory
      - name: bluetooth
        hostPath:
          path: /sys/class/bluetooth
          type: Directory
      - name: dev-bus-usb
        hostPath:
          path: /dev/bus/usb
          type: Directory
EOF
```

### 7. Verify HomeAssistant deployment
```bash
kubectl get pods -n homeassistant
kubectl logs -n homeassistant -l app=homeassistant --tail=50
```

## ✅ Step 4: Expose HomeAssistant Service

### 8. Create Service to access HomeAssistant
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: homeassistant-service
  namespace: homeassistant
  annotations:
    avahi.local/name: "homeassistant"
spec:
  selector:
    app: homeassistant
  ports:
  - name: http
    port: 8123
    targetPort: 8123
    protocol: TCP
  type: LoadBalancer
EOF
```

### 9. Wait for MetalLB to assign an IP address
```bash
kubectl get svc -n homeassistant -w
# Wait until EXTERNAL-IP shows an IP address (not <pending>)
```

### 10. Get service endpoint
```bash
# Get the assigned LoadBalancer IP
LB_IP=$(kubectl get svc homeassistant-service -n homeassistant -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "HomeAssistant URL: http://${LB_IP}:8123"
```

> Access HomeAssistant at http://<loadbalancer-ip>:8123 or http://homeassistant.local:8123 (if using Avahi advertiser)

## ✅ Step 5: Verify Bluetooth Access

### 11. Check Bluetooth devices are accessible
```bash
# Shell into the HomeAssistant pod
kubectl exec -it -n homeassistant deployment/homeassistant -- /bin/bash

# Inside the pod, check Bluetooth
hciconfig -a
bluetoothctl list

# Exit the pod
exit
```

### 12. Verify D-Bus socket access
```bash
kubectl exec -it -n homeassistant deployment/homeassistant -- ls -la /run/dbus
```

## ✅ Step 6: Configure HomeAssistant for Bluetooth

### 13. Add Bluetooth integration in configuration.yaml

Once HomeAssistant is accessible via web UI, you can enable Bluetooth integrations through:
- Navigate to **Settings** → **Devices & Services** → **Add Integration**
- Search for "Bluetooth" and configure

Alternatively, add to `/config/configuration.yaml`:
```yaml
bluetooth:
  adapters:
    - hci0  # Default Bluetooth adapter
```

### 14. Restart HomeAssistant to apply changes
```bash
kubectl rollout restart deployment/homeassistant -n homeassistant
kubectl rollout status deployment/homeassistant -n homeassistant
```

## 🔧 Troubleshooting

### Check Bluetooth adapter on host
```bash
# On the host system
hciconfig -a
bluetoothctl list
systemctl status bluetooth
```

### Verify pod has proper permissions
```bash
kubectl exec -n homeassistant deployment/homeassistant -- ls -la /sys/class/bluetooth
kubectl exec -n homeassistant deployment/homeassistant -- ls -la /dev/bus/usb
```

### View HomeAssistant logs
```bash
kubectl logs -n homeassistant -l app=homeassistant -f
```

### Check for D-Bus issues
```bash
kubectl exec -n homeassistant deployment/homeassistant -- bash -c "ls -la /run/dbus/system_bus_socket"
```

### Ensure Bluetooth service is running on host
```bash
# On the host system
sudo systemctl enable bluetooth
sudo systemctl start bluetooth
```

## 📋 Important Notes

1. **Privileged Mode**: The `securityContext.privileged: true` setting grants the container full access to the host system. This is necessary for Bluetooth but reduces security isolation.

2. **Host Network**: Using `hostNetwork: true` allows HomeAssistant to discover devices on the local network via mDNS/Zeroconf.

3. **Driver=none**: This deployment requires minikube with `--driver=none` because it needs direct hardware access to Bluetooth adapters.

4. **D-Bus Access**: The `/run/dbus` mount provides access to the system D-Bus, required for BlueZ communication.

5. **Single Replica**: HomeAssistant should only run as a single replica due to hardware access constraints.

6. **MetalLB LoadBalancer**: Using LoadBalancer type with MetalLB provides a dedicated IP address for HomeAssistant, avoiding port number conflicts that can occur with NodePort services. If you have the Avahi advertiser installed, the service will be accessible at `homeassistant.local`.

## 🗑️ Cleanup

To remove the deployment:
```bash
kubectl delete namespace homeassistant
```

## 📚 References

- [HomeAssistant Official Documentation](https://www.home-assistant.io/docs/)
- [HomeAssistant Bluetooth Integration](https://www.home-assistant.io/integrations/bluetooth/)
- [Kubernetes Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Minikube None Driver](https://minikube.sigs.k8s.io/docs/drivers/none/)
