---
title: A&K Compute Cluster
description: Documentation for the ProxmoxVE compute cluster
published: true
date: 2025-10-26T17:51:53.979Z
tags: 
editor: markdown
dateCreated: 2025-10-25T17:24:09.867Z
---

# ProxmoxVE Cluster

The compute cluster contains a set of nodes that provide reconfigurable services including gaming, programming, storage, and other tools such as this Wiki. 

## Cluster Nodes

The ProxmoxVE cluster can be managed through the WebUI on port 8006.  For example:
[https://tango.ank.com:8006](https://tango.ank.com:8006)


### tango.ank.com \[10.0.42.39\]

-   Hardware: Z2-mini
-   CPU: i5-13500 (20 threads)
-   Memory: 32GB
-   Disk: 1TB (nvme0n1) 100% allocated
-   Disk: 4TB (nvme1n1) 13% allocated

### victor.ank.com \[10.0.42.45\]

-   Hardware: Z2-mini
-   CPU: i5-14500 (20 threads)
-   Memory: 64GB
-   Disk: 2TB (nvme0n1) 100% allocated
-   Disk: 2TB (nvme0n1) 25% allocated

### xray.ank.com \[10.0.42.46\]

-   Hardware: Z2-mini
-   CPU: i5-14500 (20 threads)
-   Memory: 64GB
-   Disk: 2TB (nvme0n1) 25% allocated
-   Disk: 512GB (nvme1n1) 100% allocated

### zulu.ank.com \[10.0.42.49\]

-   Hardware: HP Elite Laptop
-   CPU: 
-   Memory:
-   Disk: 1TB (sda) 100% allocated

## Servers

### whiskey.ank.com \[10.0.42.47\]

-   Host: xray.ank.com
-   Type: LXC
-   OS: debian
-   Application: minecraft

### ariel.ank.com \[10.0.42.81\]

-   Host: xray.ank.com
-   Type: VM
-   CPU: 8 cores
-   Memory: 16G
-   Disk: 128G
-   Application:

### tamlyn.ank.com \[10.0.42.12\]

-   Host: victor.ank.com
-   Type: VM
-   CPU: 8 cores
-   Memory: 8GB
-   Disk: 50GB
-   Application: minikube

### jinx.ank.com \[10.0.42.13\]

-   Host: victor.an.com
-   Type: VM
-   CPU: 8 cores
-   Memory: 8GB
-   Disk: 50GB
-   Application: