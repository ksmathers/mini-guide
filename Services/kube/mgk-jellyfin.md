---
title: mgk-jellyfin
description: 
published: true
date: 2025-11-22T00:00:00.000Z
tags: 
editor: markdown
dateCreated: 2025-11-22T00:00:00.000Z
---

# Jellyfin Media Server on Kubernetes (Mini Guide)

This guide deploys Jellyfin media server on Kubernetes with NFS-mounted media storage and user permission controls to restrict private content access.

## ⚙️ Configuration Variables

Set these environment variables to customize your Jellyfin deployment:

```bash
# NFS Configuration
export NFS_SERVER="192.168.1.100"             # Your NFS server IP
export NFS_MEDIA_PATH="/mnt/media"            # Path to media on NFS server
export NFS_PRIVATE_PATH="/mnt/media/private"  # Path to sensitive content (optional)

# Jellyfin Configuration
export JELLYFIN_NAMESPACE="jellyfin"
export JELLYFIN_PORT="30865"                # NodePort for access

# Verify variables are set
echo "NFS Server: $NFS_SERVER"
echo "NFS Media Path: $NFS_MEDIA_PATH"
echo "Jellyfin Namespace: $JELLYFIN_NAMESPACE"
```

## ✅ Step 1: Setup Namespace and Verify Cluster

### 1. Create namespace and verify cluster access
```bash
# Create dedicated namespace
kubectl create namespace $JELLYFIN_NAMESPACE
kubectl config set-context --current --namespace=$JELLYFIN_NAMESPACE

# Verify cluster is accessible
kubectl cluster-info
kubectl get nodes
```

## ✅ Step 2: Configure Persistent Storage

### 2. Create PersistentVolumeClaim for Jellyfin config and cache
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jellyfin-config-pvc
  namespace: $JELLYFIN_NAMESPACE
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jellyfin-cache-pvc
  namespace: $JELLYFIN_NAMESPACE
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
EOF

# Verify PVCs are created
kubectl get pvc -n $JELLYFIN_NAMESPACE
```

## ✅ Step 3: Setup NFS Volumes for Media

### 3. Create PersistentVolume and PVC for general media (NFS)
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jellyfin-media-pv
spec:
  capacity:
    storage: 500Gi
  accessModes:
    - ReadOnlyMany
  nfs:
    server: $NFS_SERVER
    path: $NFS_MEDIA_PATH
  mountOptions:
    - nfsvers=4.1
    - ro
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jellyfin-media-pvc
  namespace: $JELLYFIN_NAMESPACE
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 500Gi
  volumeName: jellyfin-media-pv
EOF

# Verify NFS volume is bound
kubectl get pv jellyfin-media-pv
kubectl get pvc jellyfin-media-pvc -n $JELLYFIN_NAMESPACE
```

### 4. (Optional) Create separate PV/PVC for private content
This allows you to mount private content to a separate library that can be restricted:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jellyfin-private-pv
spec:
  capacity:
    storage: 200Gi
  accessModes:
    - ReadOnlyMany
  nfs:
    server: $NFS_SERVER
    path: $NFS_PRIVATE_PATH
  mountOptions:
    - nfsvers=4.1
    - ro
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jellyfin-private-pvc
  namespace: $JELLYFIN_NAMESPACE
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 200Gi
  volumeName: jellyfin-private-pv
EOF

# Verify private content volume is bound
kubectl get pv jellyfin-private-pv
kubectl get pvc jellyfin-private-pvc -n $JELLYFIN_NAMESPACE
```

**Note**: Mounting private content separately makes it easier to control access through Jellyfin's library permissions.

## ✅ Step 4: Deploy Jellyfin Server

### 5. Deploy Jellyfin application
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jellyfin
  namespace: $JELLYFIN_NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jellyfin
  template:
    metadata:
      labels:
        app: jellyfin
    spec:
      containers:
      - name: jellyfin
        image: jellyfin/jellyfin:latest
        ports:
        - containerPort: 8096
          name: http
        - containerPort: 8920
          name: https
        - containerPort: 1900
          name: dlna-udp
          protocol: UDP
        - containerPort: 7359
          name: discovery-udp
          protocol: UDP
        env:
        - name: TZ
          value: "America/Los_Angeles"
        volumeMounts:
        - name: config
          mountPath: /config
        - name: cache
          mountPath: /cache
        - name: media
          mountPath: /media
          readOnly: true
        - name: private
          mountPath: /media-private
          readOnly: true
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        readinessProbe:
          httpGet:
            path: /health
            port: 8096
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8096
          initialDelaySeconds: 60
          periodSeconds: 30
      volumes:
      - name: config
        persistentVolumeClaim:
          claimName: jellyfin-config-pvc
      - name: cache
        persistentVolumeClaim:
          claimName: jellyfin-cache-pvc
      - name: media
        persistentVolumeClaim:
          claimName: jellyfin-media-pvc
      - name: private
        persistentVolumeClaim:
          claimName: jellyfin-private-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: jellyfin-service
  namespace: $JELLYFIN_NAMESPACE
spec:
  selector:
    app: jellyfin
  ports:
  - name: http
    port: 8096
    targetPort: 8096
    nodePort: $JELLYFIN_PORT
  - name: https
    port: 8920
    targetPort: 8920
  - name: dlna-udp
    port: 1900
    targetPort: 1900
    protocol: UDP
  - name: discovery-udp
    port: 7359
    targetPort: 7359
    protocol: UDP
  type: NodePort
EOF
```

**Note**: If you're not using the private content volume, remove the private volume mount and volume definition from the deployment.

### 6. Verify Jellyfin deployment
```bash
kubectl get pods -n $JELLYFIN_NAMESPACE -l app=jellyfin
kubectl logs -n $JELLYFIN_NAMESPACE -l app=jellyfin --tail=30
```

## ✅ Step 5: Access and Initial Setup

### 7. Get Jellyfin access information
```bash
# Get node IP for NodePort access
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
if [ -z "$NODE_IP" ]; then
  NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
fi
echo "Access Jellyfin at: http://$NODE_IP:$JELLYFIN_PORT"
```

**Alternative: Port-Forward for local access:**
```bash
kubectl port-forward -n $JELLYFIN_NAMESPACE service/jellyfin-service 8096:8096 &
echo "Access Jellyfin at: http://localhost:8096"
```

### 8. Complete initial Jellyfin setup via web browser

1. **Welcome Screen**
   - Click "Next" to begin setup

2. **Language Selection**
   - Select your preferred language
   - Click "Next"

3. **Create Administrator Account**
   - Enter username (e.g., "admin")
   - Enter and confirm password
   - Click "Next"

4. **Setup Media Libraries**
   - Click "Add Media Library"
   - **For General Media:**
     - Content type: Movies, TV Shows, Music, etc.
     - Display name: "Movies" (or appropriate name)
     - Folders: Click + and add `/media/movies` (or subdirectory)
     - Click "OK"
   
   - **For Private Content (if configured):**
     - Content type: Movies or appropriate type
     - Display name: "Private Content"
     - Folders: Click + and add `/media-private`
     - Click "OK"

5. **Remote Access**
   - Configure remote access settings
   - Enable/disable automatic port mapping
   - Click "Next"

6. **Finish Setup**
   - Review settings
   - Click "Finish"

## ✅ Step 6: Configure User Permissions and Parental Controls

### 9. Create restricted user accounts (via web interface)

1. **Navigate to Dashboard → Users**
2. **Add New User:**
   - Username: "family" (or appropriate name)
   - Password: Set secure password
   - Click "Save"

3. **Configure User Library Access:**
   - Click on the user you just created
   - Go to "Library Access" tab
   - **Uncheck** the private content library
   - **Check** all appropriate family-friendly libraries
   - Click "Save"

4. **Enable Parental Controls:**
   - Go to "Parental Control" tab
   - Set maximum parental rating (e.g., PG-13, R)
   - Block unrated content if desired
   - Click "Save"

### 10. Configure library-level access restrictions

For each library containing private content:

1. **Navigate to Dashboard → Libraries**
2. **Edit the Private Content Library:**
   - Click the three dots next to library name
   - Select "Manage Library"
   - Go to "Advanced" section
   - Note: Jellyfin primarily uses user-level restrictions

### 11. Test user permissions
```bash
# Create a test session by logging in with restricted user
# Verify that:
# - Private content library is not visible
# - Only age-appropriate content appears
# - User cannot access restricted media
```

## ✅ Step 7: Hardware Acceleration (Optional)

If your Kubernetes nodes have GPUs or hardware transcoding capabilities:

### 12. Enable hardware acceleration
```bash
# For Intel QuickSync (modify deployment)
kubectl patch deployment jellyfin -n $JELLYFIN_NAMESPACE --type=json -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/securityContext",
    "value": {
      "privileged": true
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/volumeMounts/-",
    "value": {
      "name": "render",
      "mountPath": "/dev/dri"
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/volumes/-",
    "value": {
      "name": "render",
      "hostPath": {
        "path": "/dev/dri"
      }
    }
  }
]'
```

**Then configure in Jellyfin Dashboard:**
1. Navigate to Dashboard → Playback
2. Select hardware acceleration type (Intel QuickSync, NVENC, etc.)
3. Save changes

## ✅ Step 8: Management and Updates

### 13. Update Jellyfin (rolling update)
```bash
kubectl set image deployment/jellyfin -n $JELLYFIN_NAMESPACE jellyfin=jellyfin/jellyfin:latest
kubectl rollout status deployment/jellyfin -n $JELLYFIN_NAMESPACE
```

### 14. Backup Jellyfin configuration
```bash
# Create a backup pod
kubectl run backup-pod --rm -i --tty \
  --image=busybox \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "backup",
      "image": "busybox",
      "command": ["sh"],
      "stdin": true,
      "tty": true,
      "volumeMounts": [{
        "name": "config",
        "mountPath": "/backup"
      }]
    }],
    "volumes": [{
      "name": "config",
      "persistentVolumeClaim": {
        "claimName": "jellyfin-config-pvc"
      }
    }]
  }
}' \
  -n $JELLYFIN_NAMESPACE \
  -- tar czf - /backup

# Or copy config locally
kubectl cp $JELLYFIN_NAMESPACE/$(kubectl get pod -n $JELLYFIN_NAMESPACE -l app=jellyfin -o jsonpath='{.items[0].metadata.name}'):/config ./jellyfin-config-backup
```

### 15. Scale and resource adjustments
```bash
# Adjust resource limits if needed
kubectl set resources deployment jellyfin -n $JELLYFIN_NAMESPACE \
  --limits=cpu=4000m,memory=8Gi \
  --requests=cpu=1000m,memory=2Gi
```

### 16. Cleanup (optional)
```bash
# Remove all Jellyfin resources
kubectl delete namespace $JELLYFIN_NAMESPACE

# Or remove while preserving PVCs
kubectl delete deployment,service -n $JELLYFIN_NAMESPACE --all

# Remove NFS PVs (these are not namespaced)
kubectl delete pv jellyfin-media-pv jellyfin-private-pv
```

## 📋 Quick Reference

### Namespace Information
- **Namespace**: `jellyfin` (default)
- **Service**: `jellyfin-service:8096`
- **NodePort**: `30865` (default)

### Persistent Storage
- **Config**: PVC `jellyfin-config-pvc` (10Gi)
- **Cache**: PVC `jellyfin-cache-pvc` (20Gi)
- **Media**: NFS PV `jellyfin-media-pv` → PVC `jellyfin-media-pvc`
- **Private Content**: NFS PV `jellyfin-private-pv` → PVC `jellyfin-private-pvc`

### NFS Mount Points (inside pod)
- `/media` - General media content
- `/media-private` - Private content (restricted access)
- `/config` - Jellyfin configuration
- `/cache` - Transcoding cache

### Useful Commands
```bash
# View all resources
kubectl get all -n $JELLYFIN_NAMESPACE

# Check pod logs
kubectl logs -n $JELLYFIN_NAMESPACE -l app=jellyfin --tail=50

# Watch transcoding activity
kubectl logs -n $JELLYFIN_NAMESPACE -l app=jellyfin -f | grep -i transcode

# Execute commands in pod
kubectl exec -n $JELLYFIN_NAMESPACE -it deployment/jellyfin -- bash

# Port forward for local access
kubectl port-forward -n $JELLYFIN_NAMESPACE service/jellyfin-service 8096:8096

# Check NFS mount status
kubectl exec -n $JELLYFIN_NAMESPACE deployment/jellyfin -- df -h | grep media
kubectl exec -n $JELLYFIN_NAMESPACE deployment/jellyfin -- ls -la /media
```

## 🔧 Troubleshooting

### Common Issues

**NFS volumes not mounting:**
```bash
# Check NFS server connectivity
kubectl exec -n $JELLYFIN_NAMESPACE deployment/jellyfin -- ping -c 3 $NFS_SERVER

# Verify NFS paths exist and are accessible
# On NFS server, check exports:
showmount -e $NFS_SERVER

# Check PV/PVC status
kubectl describe pv jellyfin-media-pv
kubectl describe pvc jellyfin-media-pvc -n $JELLYFIN_NAMESPACE
```

**Pods not starting:**
```bash
kubectl describe pod -n $JELLYFIN_NAMESPACE -l app=jellyfin
kubectl logs -n $JELLYFIN_NAMESPACE -l app=jellyfin --previous
```

**Transcoding issues:**
```bash
# Check available disk space for cache
kubectl exec -n $JELLYFIN_NAMESPACE deployment/jellyfin -- df -h /cache

# Verify hardware acceleration (if enabled)
kubectl exec -n $JELLYFIN_NAMESPACE deployment/jellyfin -- ls -la /dev/dri
```

**Permission issues with media files:**
```bash
# Check file permissions on NFS mount
kubectl exec -n $JELLYFIN_NAMESPACE deployment/jellyfin -- ls -la /media

# Jellyfin runs as UID 1000 by default
# Ensure NFS exported files are readable by this user
```

**Users seeing restricted content:**
- Verify library access settings for each user
- Check that private content is in separate library
- Confirm parental control ratings are set correctly
- Test by logging in as restricted user

**NodePort not accessible:**
```bash
# Verify service configuration
kubectl get service jellyfin-service -n $JELLYFIN_NAMESPACE

# Check firewall rules allow the NodePort
# Verify node IP addresses
kubectl get nodes -o wide
```

## 🔒 Production Considerations

### Security
- Use TLS/HTTPS with Ingress controller or reverse proxy
- Implement network policies to restrict pod communication
- Regularly update Jellyfin to latest stable version
- Use strong passwords for all user accounts
- Consider VPN access for remote streaming

### Performance
- Adjust cache PVC size based on concurrent transcoding needs
- Enable hardware acceleration when available
- Monitor resource usage and adjust limits accordingly
- Use appropriate transcoding settings in Jellyfin dashboard

### Backup Strategy
- Regularly backup config PVC (contains database, settings, users)
- Implement automated backup solution for critical data
- Test restore procedures periodically
- Media files on NFS should have separate backup strategy

### User Management
- Create separate users for family members
- Use parental controls to restrict content by age rating
- Regularly audit user permissions
- Enable activity logging for security monitoring
- Consider using LDAP/SSO for enterprise deployments

### Monitoring
- Set up resource monitoring for CPU/memory usage
- Monitor transcoding load and adjust resources
- Track storage usage for cache PVC
- Alert on pod restarts or failures

### NFS Considerations
- Ensure NFS server has adequate bandwidth for streaming
- Use NFS v4.1 or higher for better performance
- Consider read-ahead and cache settings on NFS server
- Mount media as read-only to prevent accidental modifications
- Monitor NFS server performance during peak usage

---

*Guide tested with Kubernetes 1.28+ and Jellyfin 10.8+*
