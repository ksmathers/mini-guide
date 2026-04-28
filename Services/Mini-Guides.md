---
title: README
description: 
published: true
date: 2025-10-26T17:51:50.502Z
tags: 
editor: markdown
dateCreated: 2025-10-26T17:51:48.127Z
---

# Mini Guides Documentation Repository

Welcome to the mini guides documentation repository! This repository contains concise, practical guides for common infrastructure and containerization tasks.

## 📚 Available Guides

### Docker Guides

- **[Docker Registry with Read-Through Cache on Debian](./docker/mgd-docker-repository.md)**
  - Complete setup guide for creating a private Docker registry with read-through cache functionality
  - Covers installation on Debian VM, configuration, and registry deployment
  - Includes security considerations and testing procedures

- **[WikiJS Docker Container Setup](./docker/mgd-wikijs.md)**
  - Step-by-step guide for deploying WikiJS wiki platform using Docker
  - Includes PostgreSQL database setup with persistent storage
  - Complete with web browser installation and testing procedures

### Kubernetes Guides

- **[Docker Registry with Read-Through Cache on Kubernetes](./kube/mgk-docker-repository.md)**
  - Step-by-step guide for deploying a private Docker registry on Kubernetes
  - Features read-through caching of Docker Hub images
  - Includes ConfigMap setup, deployment configuration, and service exposure

- **[WikiJS on Kubernetes](./kube/mgk-wikijs.md)**
  - Complete deployment guide for WikiJS wiki platform on Kubernetes
  - Includes PostgreSQL database with persistent storage and secrets management
  - Features scaling, updates, and production considerations

- **[Minecraft Server on Kubernetes](./kube/mgk-minecraft.md)**
  - Step-by-step guide for deploying a vanilla Java Minecraft server on Kubernetes
  - Uses itzg/minecraft-server image with persistent world storage
  - Includes optional world migration, server administration, backups, and performance tuning

- **[Step-CA Certificate Authority on Kubernetes](./kube/mgk-step-ca.md)**
  - Complete guide for deploying step-ca server for homelab certificate management
  - Includes ACME integration with cert-manager for automatic TLS certificates
  - Features manual certificate management, mTLS examples, and production considerations

- **[Factorio Server on Kubernetes](./kube/mgk-factorio.md)**
  - Step-by-step guide for deploying a Factorio dedicated server on Kubernetes
  - Uses factoriotools/factorio image with persistent save data and mod support
  - Includes server administration, backup procedures, and performance tuning

### Tutorials

- **[AI-Assisted Application Development with GitHub Copilot, VS Code, copilot-cli, and Amazon AI-DLC](../tutorials/tut-ai-dlc-copilot-vscode.md)**
  - End-to-end tutorial for using AI-DLC workflow rules with GitHub Copilot in VS Code
  - Covers installation of AI-DLC rules, the `gh copilot` CLI extension, and the three-phase Inception → Construction → Operations workflow
  - Includes extension authoring for custom security and compliance rules

- **[EKS with Dynamic Graviton Node Provisioning Using Karpenter](../tutorials/tut-eks-graviton-karpenter.md)**
  - Deep dive into EKS cluster setup with Karpenter for ARM64 Graviton node provisioning
  - Explains Graviton instance families, Karpenter's provisioning loop, and EKS Pod Identity

## 🎯 Guide Categories

| Category | Count | Description |
|----------|-------|-------------|
| Docker   | 2     | Container runtime and registry management |
| Kubernetes | 5   | Container orchestration and cluster management |
| Tutorials | 2   | Conceptual deep-dives and workflow guides |

## 🚀 Quick Start

Each guide is self-contained and includes:
- ✅ Step-by-step instructions
- 📋 Prerequisites and requirements
- 🔧 Configuration examples
- 🧪 Testing and verification steps

Simply navigate to the guide you need and follow the instructions!

## 📁 Repository Structure

```
mini-guides/
├── README.md                           # This index file
├── docker/                             # Docker-related guides
│   ├── mgd-docker-repository.md        # Docker registry setup on Debian
│   └── mgd-wikijs.md                   # WikiJS wiki platform setup
└── kube/                               # Kubernetes-related guides
    ├── mgk-docker-repository.md        # Docker registry setup on Kubernetes
    ├── mgk-wikijs.md                   # WikiJS wiki platform on Kubernetes
    ├── mgk-minecraft.md                # Minecraft server on Kubernetes
    ├── mgk-step-ca.md                  # Step-CA certificate authority
    └── mgk-factorio.md                 # Factorio dedicated server
└── tutorials/                          # Conceptual tutorials
    ├── tut-eks-graviton-karpenter.md   # EKS with Graviton + Karpenter
    └── tut-ai-dlc-copilot-vscode.md   # AI-assisted dev with AI-DLC + GitHub Copilot
```

## 🤝 Contributing

When adding new guides:
1. Place Docker-related guides in the `docker/` directory
2. Place Kubernetes-related guides in the `kube/` directory
3. Use descriptive filenames with appropriate prefixes (e.g., `mgd-` for Docker, `mgk-` for Kubernetes)
4. Update this README.md to include your new guide in the index
5. Follow the existing format with clear step-by-step instructions and checkmarks (✅)

---

*Last updated: October 2025*