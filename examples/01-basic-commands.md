# Basic Kubeadm Commands

This guide covers essential kubeadm commands for cluster management.

## 🔧 Cluster Initialization

### Initialize Control Plane
```bash
# Basic initialization
sudo kubeadm init

# With specific pod network CIDR
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# With specific service subnet
sudo kubeadm init --service-cidr=10.96.0.0/12

# With specific Kubernetes version
sudo kubeadm init --kubernetes-version=v1.28.0

# With custom API server advertise address
sudo kubeadm init --apiserver-advertise-address=192.168.1.100

# Skip phases during init
sudo kubeadm init --skip-phases=addon/kube-proxy
```

### Post-Init Setup
```bash
# Setup kubectl for regular user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Setup kubectl for root user
export KUBECONFIG=/etc/kubernetes/admin.conf
```

## 👥 Node Management

### Join Worker Nodes
```bash
# Get join command from control plane
sudo kubeadm token create --print-join-command

# Join a worker node
sudo kubeadm join 192.168.1.100:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>

# Join with specific CRI socket
sudo kubeadm join 192.168.1.100:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --cri-socket=/run/containerd/containerd.sock
```

### Join Control Plane Nodes (HA)
```bash
# Upload certificates on first control plane
sudo kubeadm init --upload-certs

# Join additional control plane
sudo kubeadm join 192.168.1.100:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane --certificate-key <cert-key>
```

## 🔑 Token Management

### Create Tokens
```bash
# Create a new token
sudo kubeadm token create

# Create token with specific TTL
sudo kubeadm token create --ttl 24h

# Create token with custom description
sudo kubeadm token create --description "Worker node token"

# Create bootstrap token for joining
sudo kubeadm token create --print-join-command
```

### List and Delete Tokens
```bash
# List all tokens
sudo kubeadm token list

# Delete a specific token
sudo kubeadm token delete <token>

# Delete all tokens
sudo kubeadm token delete $(sudo kubeadm token list -o jsonpath='{.token}')
```

## 🔍 Cluster Information

### Configuration and Status
```bash
# Print cluster configuration
sudo kubeadm config print init-defaults

# Print join configuration defaults
sudo kubeadm config print join-defaults

# View cluster info
kubectl cluster-info

# View cluster info dump
kubectl cluster-info dump
```

### Certificate Information
```bash
# Check certificate expiration
sudo kubeadm certs check-expiration

# Renew all certificates
sudo kubeadm certs renew all

# Renew specific certificate
sudo kubeadm certs renew apiserver
```

## 🔄 Reset and Cleanup

### Reset Node
```bash
# Reset kubeadm on a node
sudo kubeadm reset

# Reset with cleanup
sudo kubeadm reset --cleanup-tmp-dir

# Reset and remove CNI configuration
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube/config
```

### Clean Iptables Rules
```bash
# Reset iptables
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X

# Reset ip6tables
sudo ip6tables -F && sudo ip6tables -t nat -F && sudo ip6tables -t mangle -F && sudo ip6tables -X
```

## 🔧 Configuration Management

### Generate Configuration Files
```bash
# Generate init configuration
sudo kubeadm config print init-defaults > kubeadm-init.yaml

# Generate join configuration
sudo kubeadm config print join-defaults > kubeadm-join.yaml

# Use custom configuration
sudo kubeadm init --config=kubeadm-init.yaml
```

### Migrate Configuration
```bash
# Migrate old config to new version
sudo kubeadm config migrate --old-config kubeadm-old.yaml --new-config kubeadm-new.yaml
```

## 🔍 Troubleshooting Commands

### Logs and Debugging
```bash
# View kubeadm logs
journalctl -xeu kubelet

# Reset with verbose output
sudo kubeadm reset -v=5

# Init with verbose output
sudo kubeadm init -v=5

# Check node status
kubectl get nodes -o wide

# Describe node for issues
kubectl describe node <node-name>
```

### System Verification
```bash
# Check system requirements
sudo kubeadm init phase preflight

# Verify kubelet configuration
sudo kubeadm init phase kubelet-start

# Check API server
kubectl get --raw='/readyz?verbose'
```

## 💡 Practice Exercises

1. **Basic Setup**: Initialize a single-node cluster
2. **Multi-Node**: Add 2 worker nodes to your cluster
3. **Token Rotation**: Create and rotate join tokens
4. **Certificate Management**: Check and renew certificates
5. **Troubleshooting**: Intentionally break and fix cluster
6. **Reset Practice**: Reset and reinitialize cluster
7. **Configuration**: Use custom init configuration
8. **HA Setup**: Create 3-node control plane cluster

## 📝 Common Patterns

### Complete Cluster Setup
```bash
# 1. Initialize control plane
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# 2. Setup kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 3. Install CNI (Flannel example)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# 4. Get join command for workers
sudo kubeadm token create --print-join-command
```

### Quick Reset and Reinit
```bash
# Reset everything
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube/config

# Clean iptables
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X

# Reinitialize
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```