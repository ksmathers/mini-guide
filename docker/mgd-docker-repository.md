---
title: mgd-docker-repository
description: 
published: true
date: 2025-10-26T17:51:59.100Z
tags: 
editor: markdown
dateCreated: 2025-10-26T17:51:56.482Z
---

# Docker Registry with Read-Through Cache on Debian (Simplified Guide)

## ✅ Part 1: Set Up the Debian VM Docker Registry (Read-Through Cache)

### 1. Install Docker (on Debian VM)
```
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

### 2. Create directories for registry data and configuration
```
sudo mkdir -p /opt/registry/{data,config}
```

### 3. Create the registry configuration file

Create the file `/opt/registry/config/config.yml` with the following content:

```yaml
version: 0.1
storage:
  filesystem:
    rootdirectory: /var/lib/registry
proxy:
  remoteurl: https://registry-1.docker.io
http:
  addr: :5000
```

### 4. Run the registry container
```
sudo docker run -d --name registry-cache \
  -p 5000:5000 \
  -v /opt/registry/config:/etc/docker/registry \
  -v /opt/registry/data:/var/lib/registry \
  registry:2
```

### 5. Verify that it’s running
```
sudo docker ps
```

---

## ✅ Part 2: Allow Insecure Registry (Development Only)

If you do not configure TLS, Docker clients must trust the registry as insecure.

### On your development machine (e.g., laptop), edit `/etc/docker/daemon.json`:
```json
{
  "insecure-registries": ["<VM_IP>:5000"]
}
```

Then restart Docker:
```
sudo systemctl restart docker
```

---

## ✅ Part 3: Test the Read-Through Cache (From Dev Machine)

### 1. Pull an image through the registry
```
docker pull <VM_IP>:5000/library/alpine:latest
```

✅ First pull fetches from Docker Hub and caches it.

### 2. Pull again (should be fast from cache)
```
docker pull <VM_IP>:5000/library/alpine:latest
```

---

## ✅ Part 4: Build, Push, and Pull a Custom Image

### 1. Create a simple Dockerfile
```
echo -e "FROM <VM_IP>:5000/library/alpine:latest\nRUN echo 'hello from cached registry' > /message.txt" > Dockerfile
```

### 2. Build and tag the image
```
docker build -t <VM_IP>:5000/mytest:latest .
```

### 3. Push to the registry
```
docker push <VM_IP>:5000/mytest:latest
```

### 4. Remove and re-pull to verify
```
docker rmi <VM_IP>:5000/mytest:latest
docker pull <VM_IP>:5000/mytest:latest
```

✅ Confirms registry supports both cached pulls and private pushes.

---

## ✅ Part 5: Optional Hardening (For Production)

| Security Step        | Description                              |
|---------------------|-------------------------------------------|
| Add TLS             | Use HTTPS certs in registry config         |
| Add basic auth      | Use htpasswd + registry auth config        |
| Restrict access     | Apply firewall/network rules               |
| Monitor disk usage  | Track `/opt/registry/data` size            |

---

## 🎉 Summary

You now have:
✅ A Docker registry acting as a proxy cache for Docker Hub  
✅ Faster development cycle due to cached images  
✅ Ability to host private images locally
