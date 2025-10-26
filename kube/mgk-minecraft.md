# Minecraft Server on Kubernetes (Mini Guide)

This guide deploys a vanilla Java Minecraft server on Kubernetes using the itzg/minecraft-server Docker image with persistent world storage.

## ✅ Step 1: Setup Namespace and Verify Cluster

### 1. Create namespace and verify cluster access
```bash
# Create dedicated namespace
kubectl create namespace minecraft
kubectl config set-context --current --namespace=minecraft

# Verify cluster is accessible
kubectl cluster-info
kubectl get nodes
```

## ✅ Step 2: Create Persistent Storage

### 2. Create and verify PersistentVolumeClaim for world data
```bash
# Create PVC for world storage
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minecraft-data-pvc
  namespace: minecraft
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF

# Verify PVC is created and bound
kubectl get pvc -n minecraft
```

## ✅ Step 2b: Import Existing World (Optional)

If you have an existing Minecraft world you want to use instead of generating a new one, follow these steps:

### 2b.1. Create temporary pod to load world data
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: world-loader
  namespace: minecraft
spec:
  containers:
  - name: loader
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: world-volume
      mountPath: /data
  volumes:
  - name: world-volume
    persistentVolumeClaim:
      claimName: minecraft-data-pvc
EOF
```

### 2b.2. Wait for pod to be ready
```bash
kubectl wait --for=condition=ready pod world-loader -n minecraft --timeout=120s
```

### 2b.3. Copy your local world folder to the persistent volume
```bash
# Method 1: Direct copy (if world folder is small)
kubectl cp ./world minecraft/world-loader:/data/world

# Method 2: Compressed transfer (recommended for large worlds)
tar -czf world.tar.gz -C /path/to/your/world .
kubectl cp world.tar.gz minecraft/world-loader:/data/
kubectl exec -n minecraft -it world-loader -- sh -c "cd /data && tar -xzf world.tar.gz && rm world.tar.gz"
```

### 2b.4. Verify world data was copied
```bash
kubectl exec -n minecraft -it world-loader -- ls -la /data/world
kubectl exec -n minecraft -it world-loader -- ls /data/world/region
```

### 2b.5. Clean up temporary pod
```bash
kubectl delete pod world-loader -n minecraft
```

**Note:** If you imported an existing world, you may want to update the LEVEL configuration in the next step to match your world folder name.

## ✅ Step 3: Server Configuration

### 3. Create server configuration and admin settings
```bash
# Create server configuration
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: minecraft-config
  namespace: minecraft
data:
  EULA: "TRUE"
  TYPE: "VANILLA"
  VERSION: "LATEST"
  DIFFICULTY: "normal"
  MAX_PLAYERS: "20"
  ALLOW_NETHER: "true"
  ANNOUNCE_PLAYER_ACHIEVEMENTS: "true"
  ENABLE_COMMAND_BLOCK: "true"
  FORCE_GAMEMODE: "false"
  GENERATE_STRUCTURES: "true"
  HARDCORE: "false"
  MAX_BUILD_HEIGHT: "256"
  MAX_WORLD_SIZE: "29999984"
  PVP: "true"
  SPAWN_ANIMALS: "true"
  SPAWN_MONSTERS: "true"
  SPAWN_NPCS: "true"
  VIEW_DISTANCE: "10"
  SEED: ""
  MODE: "survival"
  MOTD: "A Kubernetes Minecraft Server"
  LEVEL: "world"
  ONLINE_MODE: "true"
  MEMORY: "2G"
EOF

# Create admin settings (optional - replace usernames with your own)
kubectl create secret generic minecraft-admin -n minecraft \
  --from-literal=ops='["your-minecraft-username"]' \
  --from-literal=whitelist='["your-minecraft-username","friend1","friend2"]'
```

## ✅ Step 4: Deploy Minecraft Server

### 4. Deploy and verify Minecraft server
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minecraft-server
  namespace: minecraft
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minecraft-server
  template:
    metadata:
      labels:
        app: minecraft-server
    spec:
      containers:
      - name: minecraft
        image: itzg/minecraft-server:latest
        ports:
        - containerPort: 25565
          protocol: TCP
        envFrom:
        - configMapRef:
            name: minecraft-config
        env:
        - name: OPS
          valueFrom:
            secretKeyRef:
              name: minecraft-admin
              key: ops
              optional: true
        - name: WHITELIST
          valueFrom:
            secretKeyRef:
              name: minecraft-admin
              key: whitelist
              optional: true
        volumeMounts:
        - name: minecraft-data
          mountPath: /data
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        readinessProbe:
          exec:
            command:
            - mcstatus
            - localhost
            - ping
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          exec:
            command:
            - mcstatus
            - localhost
            - ping
          initialDelaySeconds: 60
          periodSeconds: 30
      volumes:
      - name: minecraft-data
        persistentVolumeClaim:
          claimName: minecraft-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: minecraft-service
  namespace: minecraft
spec:
  selector:
    app: minecraft-server
  ports:
  - port: 25565
    targetPort: 25565
    nodePort: 30565
    protocol: TCP
  type: NodePort
EOF

# Verify deployment is running
kubectl get pods -n minecraft -l app=minecraft-server
kubectl logs -n minecraft -l app=minecraft-server --tail=20
```

## ✅ Step 5: Access Server

### 5. Get server connection details
```bash
# Check service and get connection info
kubectl get service minecraft-service -n minecraft
kubectl get nodes -o wide

# Get node IP and display connection address
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
if [ -z "$NODE_IP" ]; then
  NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
fi
echo "Minecraft Server Address: $NODE_IP:30565"
```

### 6. Monitor startup and test connectivity
```bash
# Watch server logs during startup (can take 2-5 minutes)
kubectl logs -n minecraft -l app=minecraft-server -f
# Look for "Done" message indicating server is ready
# Expected output: "[Server thread/INFO]: Done (X.XXXs)! For help, type "help""

# Test server connectivity (in new terminal or after logs show "Done")
kubectl exec -n minecraft -it deployment/minecraft-server -- mcstatus localhost ping
kubectl exec -n minecraft -it deployment/minecraft-server -- mcstatus localhost status
```

### 7. Connect with Minecraft client
1. **Open Minecraft Java Edition**
2. **Go to Multiplayer**
3. **Add Server**
   - Server Name: `Kubernetes Minecraft`
   - Server Address: `<NODE_IP>:30565`
4. **Join Server**

### 8. Verify world persistence
```bash
# Create something in-game, then restart server
kubectl rollout restart deployment/minecraft-server -n minecraft

# Wait for server to restart
kubectl wait --for=condition=ready pod -l app=minecraft-server -n minecraft --timeout=300s

# Reconnect - your changes should persist
```

## ✅ Step 6: Server Administration

### 9. Execute server commands
```bash
# Connect to server console
kubectl exec -n minecraft -it deployment/minecraft-server -- rcon-cli

# Or run single commands
kubectl exec -n minecraft deployment/minecraft-server -- rcon-cli "list"
kubectl exec -n minecraft deployment/minecraft-server -- rcon-cli "say Hello from Kubernetes!"
```

### 10. Manage server files
```bash
# Access server files
kubectl exec -n minecraft -it deployment/minecraft-server -- sh

# View world files
kubectl exec -n minecraft deployment/minecraft-server -- ls -la /data/world

# Check server properties
kubectl exec -n minecraft deployment/minecraft-server -- cat /data/server.properties
```

### 11. Backup world data
```bash
# Create backup directory locally
mkdir -p ~/minecraft-backups

# Copy world data
kubectl cp minecraft/$(kubectl get pod -n minecraft -l app=minecraft-server -o jsonpath='{.items[0].metadata.name}'):/data/world ~/minecraft-backups/world-$(date +%Y%m%d-%H%M%S)
```

## ✅ Step 7: Server Management

### 12. Stop and start server
```bash
# Stop server gracefully
kubectl exec -n minecraft deployment/minecraft-server -- rcon-cli "stop"
# Or scale down deployment
kubectl scale deployment minecraft-server -n minecraft --replicas=0

# Start server
kubectl scale deployment minecraft-server -n minecraft --replicas=1
kubectl wait --for=condition=ready pod -l app=minecraft-server -n minecraft --timeout=300s
```

### 13. Update server version
```bash
# Update to specific version
kubectl patch configmap minecraft-config -n minecraft --patch '{"data":{"VERSION":"1.20.1"}}'
kubectl rollout restart deployment/minecraft-server -n minecraft

# Or update to latest
kubectl set image deployment/minecraft-server -n minecraft minecraft=itzg/minecraft-server:latest
kubectl rollout status deployment/minecraft-server -n minecraft
```

### 14. Cleanup (optional)
```bash
# Remove all Minecraft resources
kubectl delete namespace minecraft
```

## 📋 Quick Reference

### Connection Information
- **Namespace**: `minecraft`
- **Server Port**: `30565` (NodePort)
- **Internal Service**: `minecraft-service:25565`

### Persistent Storage
- **World Data**: PVC `minecraft-data-pvc` (10Gi)
- **Mount Path**: `/data` (contains world, plugins, configs)

### Useful Commands
```bash
# View all resources
kubectl get all -n minecraft

# Check server logs
kubectl logs -n minecraft -l app=minecraft-server --tail=50

# Server console access
kubectl exec -n minecraft -it deployment/minecraft-server -- rcon-cli

# Check server status
kubectl exec -n minecraft deployment/minecraft-server -- mcstatus localhost ping

# File access
kubectl exec -n minecraft -it deployment/minecraft-server -- sh

# Port forward for local testing
kubectl port-forward -n minecraft service/minecraft-service 25565:25565
```

## 🔧 Troubleshooting

- **Pod not starting**: Check resource limits and node capacity with `kubectl describe pod`
- **Server won't start**: Verify EULA is set to TRUE in ConfigMap
- **Can't connect**: Ensure firewall allows port 30565 and check node IP
- **World not persisting**: Verify PVC is bound and mounted correctly
- **Out of memory**: Increase memory limits or MEMORY environment variable
- **Server lag**: Reduce view distance or increase CPU/memory resources

## ⚙️ Configuration Options

### Common Server Settings (ConfigMap)
```yaml
VERSION: "1.20.1"          # Specific Minecraft version
DIFFICULTY: "easy"         # peaceful, easy, normal, hard
MAX_PLAYERS: "10"          # Maximum concurrent players
PVP: "false"              # Enable/disable player vs player
SPAWN_MONSTERS: "false"   # Disable hostile mobs
VIEW_DISTANCE: "8"        # Reduce for better performance
MEMORY: "4G"              # Server memory allocation
SEED: "12345"             # World generation seed
MOTD: "My Server"         # Server description
LEVEL: "custom_world"     # World folder name
```

### Performance Tuning
- **Low Resource**: 1Gi memory, VIEW_DISTANCE=6, MAX_PLAYERS=5
- **Medium Resource**: 2Gi memory, VIEW_DISTANCE=10, MAX_PLAYERS=15  
- **High Resource**: 4Gi memory, VIEW_DISTANCE=12, MAX_PLAYERS=30

## 🔒 Production Considerations

- Set up regular automated backups of world data
- Configure resource quotas and limits appropriately
- Use LoadBalancer service type for cloud environments
- Implement monitoring and alerting for server health
- Consider using StatefulSet for more predictable storage
- Set up log aggregation for server events
- Configure network policies for security

---

*Guide tested with Minecraft 1.20+ and itzg/minecraft-server latest*