---
title: mgk-factorio
description: 
published: true
date: 2025-10-26T17:52:09.790Z
tags: 
editor: markdown
dateCreated: 2025-10-26T17:52:06.999Z
---

# Factorio Server on Kubernetes (Mini Guide)

This guide deploys a Factorio dedicated server on Kubernetes with persistent save data and proper configuration management.

## 📋 Factorio Server Authentication Notes

**Important**: You do **NOT** need a Factorio account to host a server, and you **CAN** play on your own server!

- **No Factorio.com account required** for hosting
- **`ADMIN_USERNAME`** is just your in-game player name for admin privileges
- **`username/password/token`** in server settings are only needed for:
  - Public server listing on Factorio's public server browser
  - Factorio.com account verification (optional)
- **For homelab/LAN servers**: Leave these empty - no authentication needed
- **You can play on your own server** with any player name you choose

## ⚙️ Configuration Variables

Set these environment variables to customize your Factorio deployment:

```bash
# Basic server configuration
export SERVER_NAME="Kubernetes Factorio Server"
export SERVER_DESCRIPTION="A Factorio server running on Kubernetes"
export ADMIN_USERNAME="your-in-game-username"  # Your Factorio player name (not account)
export SERVER_PASSWORD=""  # Optional - leave empty for no password (recommended for local)

# Kubernetes configuration
export FACTORIO_NAMESPACE="factorio"

# Verify variables are set
echo "Server Name: $SERVER_NAME"
echo "Admin: $ADMIN_USERNAME" 
echo "Namespace: $FACTORIO_NAMESPACE"
```

## ✅ Step 1: Setup Namespace and Persistent Storage

### 1. Create namespace and persistent storage
```bash
# Create dedicated namespace
kubectl create namespace $FACTORIO_NAMESPACE
kubectl config set-context --current --namespace=$FACTORIO_NAMESPACE

# Create PVC for save data, mods, and configuration
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: factorio-data-pvc
  namespace: $FACTORIO_NAMESPACE
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF

# Verify PVC is created and bound
kubectl get pvc -n $FACTORIO_NAMESPACE
```

## ✅ Step 2: Server Configuration

### 2. Create server configuration and settings
```bash
# Create server settings ConfigMap
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: factorio-config
  namespace: $FACTORIO_NAMESPACE
data:
  server-settings.json: |
    {
      "name": "$SERVER_NAME",
      "description": "$SERVER_DESCRIPTION",
      "tags": ["kubernetes", "homelab"],
      "max_players": 10,
      "visibility": {
        "public": false,
        "lan": true
      },
      "username": "",
      "password": "",
      "token": "",
      "game_password": "$SERVER_PASSWORD",
      "require_user_verification": false,
      "max_upload_in_kilobytes_per_second": 0,
      "max_upload_slots": 5,
      "minimum_latency_in_ticks": 0,
      "ignore_player_limit_for_returning_players": false,
      "allow_commands": "admins-only",
      "autosave_interval": 10,
      "autosave_slots": 5,
      "afk_autokick_interval": 0,
      "auto_pause": true,
      "only_admins_can_pause_the_game": true,
      "autosave_only_on_server": true,
      "non_blocking_saving": false,
      "minimum_segment_size": 25,
      "minimum_segment_size_peer_count": 20,
      "maximum_segment_size": 100,
      "maximum_segment_size_peer_count": 10
    }
  map-gen-settings.json: |
    {
      "terrain_segmentation": 1,
      "water": 1,
      "width": 0,
      "height": 0,
      "starting_area": 1,
      "peaceful_mode": false,
      "autoplace_controls": {
        "coal": {"frequency": 1, "size": 1, "richness": 1},
        "stone": {"frequency": 1, "size": 1, "richness": 1},
        "copper-ore": {"frequency": 1, "size": 1, "richness": 1},
        "iron-ore": {"frequency": 1, "size": 1, "richness": 1},
        "uranium-ore": {"frequency": 1, "size": 1, "richness": 1},
        "crude-oil": {"frequency": 1, "size": 1, "richness": 1},
        "trees": {"frequency": 1, "size": 1, "richness": 1},
        "enemy-base": {"frequency": 1, "size": 1, "richness": 1}
      },
      "cliff_settings": {
        "name": "cliff",
        "cliff_elevation_0": 10,
        "cliff_elevation_interval": 40,
        "richness": 1
      },
      "property_expression_names": {},
      "starting_points": [{"x": 0, "y": 0}],
      "seed": null
    }
  map-settings.json: |
    {
      "difficulty_settings": {
        "recipe_difficulty": 0,
        "technology_difficulty": 0,
        "technology_price_multiplier": 1,
        "research_queue_setting": "after-victory"
      },
      "pollution": {
        "enabled": true,
        "diffusion_ratio": 0.02,
        "min_to_diffuse": 15,
        "ageing": 1,
        "expected_max_per_chunk": 150,
        "min_to_show_per_chunk": 50,
        "min_pollution_to_damage_trees": 60,
        "pollution_with_max_forest_damage": 150,
        "pollution_per_tree_damage": 50,
        "pollution_restored_per_tree_damage": 10,
        "max_pollution_to_restore_trees": 20,
        "enemy_attack_pollution_consumption_modifier": 1
      },
      "enemy_evolution": {
        "enabled": true,
        "time_factor": 0.000004,
        "destroy_factor": 0.002,
        "pollution_factor": 0.0000009
      },
      "enemy_expansion": {
        "enabled": true,
        "min_base_spacing": 3,
        "max_expansion_distance": 7,
        "friendly_base_influence_radius": 2,
        "enemy_building_influence_radius": 2,
        "building_coefficient": 0.1,
        "other_base_coefficient": 2.0,
        "neighbouring_chunk_coefficient": 0.5,
        "neighbouring_base_chunk_coefficient": 0.4,
        "max_colliding_tiles_coefficient": 0.9,
        "settler_group_min_size": 5,
        "settler_group_max_size": 20,
        "min_expand_cooldown": 14400,
        "max_expand_cooldown": 216000
      },
      "unit_group": {
        "min_group_gathering_time": 3600,
        "max_group_gathering_time": 36000,
        "max_wait_time_for_late_members": 7200,
        "max_group_radius": 30.0,
        "min_group_radius": 5.0,
        "max_member_speedup_when_behind": 1.4,
        "max_member_slowdown_when_ahead": 0.6,
        "max_group_slowdown_factor": 0.3,
        "max_group_member_fallback_factor": 3,
        "member_disown_distance": 10,
        "tick_tolerance_when_member_arrives": 60
      },
      "steering": {
        "default": {
          "radius": 1.2,
          "separation_force": 0.005,
          "separation_factor": 1.2,
          "force_unit_fuzzy_goto_behavior": false
        },
        "moving": {
          "radius": 3,
          "separation_force": 0.01,
          "separation_factor": 3,
          "force_unit_fuzzy_goto_behavior": false
        }
      },
      "path_finder": {
        "fwd2bwd_ratio": 5,
        "goal_pressure_ratio": 2,
        "max_steps_worked_per_tick": 100,
        "max_work_done_per_tick": 8000,
        "use_path_cache": true,
        "short_cache_size": 5,
        "long_cache_size": 25,
        "short_cache_min_cacheable_distance": 10,
        "short_cache_min_algo_steps_to_cache": 50,
        "long_cache_min_cacheable_distance": 30,
        "cache_max_connect_to_cache_steps_multiplier": 100,
        "cache_accept_path_start_distance_ratio": 0.2,
        "cache_accept_path_end_distance_ratio": 0.15,
        "negative_cache_accept_path_start_distance_ratio": 0.3,
        "negative_cache_accept_path_end_distance_ratio": 0.3,
        "cache_path_start_distance_rating_multiplier": 10,
        "cache_path_end_distance_rating_multiplier": 20,
        "stale_enemy_with_same_destination_collision_penalty": 30,
        "ignore_moving_enemy_collision_distance": 5,
        "enemy_with_different_destination_collision_penalty": 30,
        "general_entity_collision_penalty": 10,
        "general_entity_subsequent_collision_penalty": 3,
        "extended_collision_penalty": 3,
        "max_clients_to_accept_any_new_request": 10,
        "max_clients_to_accept_short_new_request": 100,
        "direct_distance_to_consider_short_request": 100,
        "short_request_max_steps": 1000,
        "short_request_ratio": 0.5,
        "min_steps_to_check_path_find_termination": 2000,
        "start_to_goal_cost_multiplier_to_terminate_path_find": 500.0,
        "overload_levels": [0, 100, 500],
        "overload_multipliers": [2, 3, 4],
        "negative_path_cache_delay_interval": 20
      }
    }
---
apiVersion: v1
kind: Secret
metadata:
  name: factorio-admin
  namespace: $FACTORIO_NAMESPACE
type: Opaque
stringData:
  admins.json: |
    ["$ADMIN_USERNAME"]
  whitelist.json: |
    ["$ADMIN_USERNAME"]
EOF
```

## ✅ Step 3: Deploy Factorio Server

### 3. Deploy and verify Factorio server
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: factorio-server
  namespace: $FACTORIO_NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: factorio-server
  template:
    metadata:
      labels:
        app: factorio-server
    spec:
      containers:
      - name: factorio
        image: factoriotools/factorio:stable
        ports:
        - containerPort: 34197
          protocol: UDP
        - containerPort: 27015
          protocol: TCP
        env:
        - name: FACTORIO_USER_UID
          value: "1000"
        - name: FACTORIO_USER_GID
          value: "1000"
        - name: GENERATE_NEW_SAVE
          value: "true"
        - name: SAVE_NAME
          value: "kubernetes-server"
        - name: LOAD_LATEST_SAVE
          value: "true"
        - name: UPDATE_MODS_ON_START
          value: "false"
        volumeMounts:
        - name: factorio-data
          mountPath: /factorio
        - name: server-settings
          mountPath: /factorio/config/server-settings.json
          subPath: server-settings.json
        - name: server-settings
          mountPath: /factorio/config/map-gen-settings.json
          subPath: map-gen-settings.json
        - name: server-settings
          mountPath: /factorio/config/map-settings.json
          subPath: map-settings.json
        - name: admin-config
          mountPath: /factorio/config/server-adminlist.json
          subPath: admins.json
        - name: admin-config
          mountPath: /factorio/config/server-whitelist.json
          subPath: whitelist.json
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        readinessProbe:
          tcpSocket:
            port: 27015
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          tcpSocket:
            port: 27015
          initialDelaySeconds: 60
          periodSeconds: 30
      volumes:
      - name: factorio-data
        persistentVolumeClaim:
          claimName: factorio-data-pvc
      - name: server-settings
        configMap:
          name: factorio-config
      - name: admin-config
        secret:
          secretName: factorio-admin
---
apiVersion: v1
kind: Service
metadata:
  name: factorio-service
  namespace: $FACTORIO_NAMESPACE
spec:
  selector:
    app: factorio-server
  ports:
  - name: game-udp
    port: 34197
    targetPort: 34197
    nodePort: 30419
    protocol: UDP
  - name: rcon-tcp
    port: 27015
    targetPort: 27015
    nodePort: 30270
    protocol: TCP
  type: NodePort
EOF

# Verify deployment is running
kubectl get pods -n $FACTORIO_NAMESPACE -l app=factorio-server
kubectl logs -n $FACTORIO_NAMESPACE -l app=factorio-server --tail=20
```

## ✅ Step 4: Access and Connect to Server

### 4. Get server connection details
```bash
# Check service and get connection info
kubectl get service factorio-service -n $FACTORIO_NAMESPACE
kubectl get nodes -o wide

# Get node IP and display connection address
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
if [ -z "$NODE_IP" ]; then
  NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
fi
echo "Factorio Server Address: $NODE_IP:30419"
echo "RCON Address: $NODE_IP:30270"
```

### 5. Monitor server startup and test connectivity
```bash
# Watch server logs during startup (can take 2-5 minutes for world generation)
kubectl logs -n $FACTORIO_NAMESPACE -l app=factorio-server -f
# Look for "Info ServerMultiplayerManager.cpp" messages indicating server is ready

# Test server connectivity (in new terminal after server shows as ready)
kubectl exec -n $FACTORIO_NAMESPACE -it deployment/factorio-server -- ss -tuln
kubectl get pods -n $FACTORIO_NAMESPACE -l app=factorio-server -o wide
```

### 6. Connect with Factorio client
1. **Open Factorio Game**
2. **Go to Multiplayer**
3. **Connect to Server**
   - Address: `<NODE_IP>:30419`
   - Password: `<SERVER_PASSWORD>` (if set)
4. **Join Game**

### 7. Verify save persistence
```bash
# Make changes in-game, then restart server
kubectl rollout restart deployment/factorio-server -n $FACTORIO_NAMESPACE

# Wait for server to restart
kubectl wait --for=condition=ready pod -l app=factorio-server -n $FACTORIO_NAMESPACE --timeout=300s

# Reconnect - your changes should persist
```

## ✅ Step 5: Server Administration

### 8. Server management commands
```bash
# View server files and saves
kubectl exec -n $FACTORIO_NAMESPACE -it deployment/factorio-server -- ls -la /factorio/saves
kubectl exec -n $FACTORIO_NAMESPACE -it deployment/factorio-server -- ls -la /factorio/mods

# Check server configuration
kubectl exec -n $FACTORIO_NAMESPACE deployment/factorio-server -- cat /factorio/config/server-settings.json

# View current players (when server is running)
kubectl logs -n $FACTORIO_NAMESPACE -l app=factorio-server --tail=50 | grep -i player
```

### 9. Backup save data
```bash
# Create backup directory locally
mkdir -p ~/factorio-backups

# Copy save files
kubectl cp $FACTORIO_NAMESPACE/$(kubectl get pod -n $FACTORIO_NAMESPACE -l app=factorio-server -o jsonpath='{.items[0].metadata.name}'):/factorio/saves ~/factorio-backups/saves-$(date +%Y%m%d-%H%M%S)

# Copy mods (if any)
kubectl cp $FACTORIO_NAMESPACE/$(kubectl get pod -n $FACTORIO_NAMESPACE -l app=factorio-server -o jsonpath='{.items[0].metadata.name}'):/factorio/mods ~/factorio-backups/mods-$(date +%Y%m%d-%H%M%S)
```

### 10. Mod management
```bash
# Access server filesystem for mod management
kubectl exec -n $FACTORIO_NAMESPACE -it deployment/factorio-server -- sh

# Inside container: download mods manually or use mod portal
# Example: enable auto mod updates
kubectl patch deployment factorio-server -n $FACTORIO_NAMESPACE --patch '{"spec":{"template":{"spec":{"containers":[{"name":"factorio","env":[{"name":"UPDATE_MODS_ON_START","value":"true"}]}]}}}}'
kubectl rollout restart deployment/factorio-server -n $FACTORIO_NAMESPACE
```

## ✅ Step 6: Server Management

### 11. Stop and start server
```bash
# Stop server gracefully
kubectl scale deployment factorio-server -n $FACTORIO_NAMESPACE --replicas=0

# Start server
kubectl scale deployment factorio-server -n $FACTORIO_NAMESPACE --replicas=1
kubectl wait --for=condition=ready pod -l app=factorio-server -n $FACTORIO_NAMESPACE --timeout=300s
```

### 12. Update server version
```bash
# Update to latest stable version
kubectl set image deployment/factorio-server -n $FACTORIO_NAMESPACE factorio=factoriotools/factorio:stable
kubectl rollout status deployment/factorio-server -n $FACTORIO_NAMESPACE

# Update to specific version
kubectl set image deployment/factorio-server -n $FACTORIO_NAMESPACE factorio=factoriotools/factorio:1.1.94
```

### 13. Cleanup (optional)
```bash
# Remove all Factorio resources
kubectl delete namespace $FACTORIO_NAMESPACE
```

## 📋 Quick Reference

### Connection Information
- **Game Port**: `30419` (UDP)
- **RCON Port**: `30270` (TCP)
- **Internal Service**: `factorio-service:34197`

### Persistent Storage
- **Save Data**: PVC `factorio-data-pvc` (10Gi)
- **Mount Path**: `/factorio` (contains saves, mods, config)

### Useful Commands
```bash
# View all resources
kubectl get all -n $FACTORIO_NAMESPACE

# Check server logs
kubectl logs -n $FACTORIO_NAMESPACE -l app=factorio-server --tail=50

# Server console access
kubectl exec -n $FACTORIO_NAMESPACE -it deployment/factorio-server -- sh

# Check save files
kubectl exec -n $FACTORIO_NAMESPACE deployment/factorio-server -- ls -la /factorio/saves

# Port forward for local testing
kubectl port-forward -n $FACTORIO_NAMESPACE service/factorio-service 34197:34197
```

## 🔧 Troubleshooting

- **Pod not starting**: Check resource limits and node capacity with `kubectl describe pod`
- **Can't connect to server**: Ensure firewall allows UDP port 30419 and check node IP
- **"Authentication failed"**: This server doesn't require Factorio.com accounts - try any username
- **Save files not persisting**: Verify PVC is bound and mounted correctly
- **Server crashes/OOM**: Increase memory limits or reduce map complexity
- **Slow performance**: Increase CPU limits or reduce enemy/pollution settings
- **Mods not loading**: Check mod compatibility and factorio version
- **Can't be admin**: Check that your player name matches `ADMIN_USERNAME` exactly

## ⚙️ Configuration Options

### Server Performance Settings
```yaml
# Low resource configuration
resources:
  requests: {memory: "512Mi", cpu: "250m"}
  limits: {memory: "2Gi", cpu: "1000m"}

# High performance configuration  
resources:
  requests: {memory: "2Gi", cpu: "1000m"}
  limits: {memory: "8Gi", cpu: "4000m"}
```

### World Generation Settings
- **Peaceful mode**: Set `"peaceful_mode": true` in map-gen-settings.json
- **Resource richness**: Increase richness values for easier gameplay
- **Enemy settings**: Disable evolution/expansion for casual play
- **Map size**: Set width/height to limit world size

## 🔒 Production Considerations

- Set up regular automated backups of save data
- Configure resource quotas and limits appropriately  
- Use LoadBalancer service type for cloud environments
- Implement monitoring for server performance
- Consider using StatefulSet for more predictable storage
- Set up log aggregation for server events
- Configure network policies for security
- Monitor memory usage and adjust limits accordingly

---

*Guide tested with Factorio 1.1+ and factoriotools/factorio:stable*