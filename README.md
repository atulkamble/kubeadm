# Kubeadm Practice Guide

A comprehensive guide for practicing kubeadm commands, cluster setup, and troubleshooting.

## 📁 Directory Structure

```
kubeadm/
├── configs/          # Configuration files and manifests
├── examples/         # Example scenarios and use cases
├── scripts/          # Automation scripts
├── troubleshooting/  # Common issues and solutions
└── README.md        # This file
```

## 🚀 Quick Start

1. **Initialize a cluster**: `./scripts/cluster-init.sh`
2. **Join worker nodes**: `./scripts/worker-join.sh`
3. **Install CNI**: `./scripts/install-cni.sh`
4. **Troubleshoot issues**: Check `troubleshooting/` directory

## 📚 Learning Path

### Beginner
1. [Basic Commands](examples/01-basic-commands.md)
2. [Single Node Setup](examples/02-single-node.md)
3. [Multi-Node Setup](examples/03-multi-node.md)

### Intermediate
4. [Custom Configurations](configs/)
5. [Networking Setup](examples/04-networking.md)
6. [Security Configurations](examples/05-security.md)

### Advanced
7. [High Availability](examples/06-ha-setup.md)
8. [Upgrade Procedures](examples/07-upgrades.md)
9. [Troubleshooting](troubleshooting/)

## 🛠 Prerequisites

- Ubuntu 20.04+ or CentOS 7+
- 2GB+ RAM per node
- 2+ CPUs per node
- Docker or containerd runtime
- Network connectivity between nodes

## 📖 Documentation

Each directory contains detailed documentation and examples for hands-on practice.