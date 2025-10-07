# Multi-Node Kubernetes Cluster Setup

Complete guide for setting up a multi-node Kubernetes cluster using kubeadm.

## 🎯 Objectives

- Set up a 3-node Kubernetes cluster (1 control plane, 2 workers)
- Configure networking between nodes
- Join worker nodes to the cluster
- Deploy multi-replica applications

## 🏗 Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Control Plane │    │   Worker Node 1 │    │   Worker Node 2 │
│   (Master)      │    │                 │    │                 │
│  192.168.1.100  │    │  192.168.1.101  │    │  192.168.1.102  │
│                 │    │                 │    │                 │
│  - API Server   │    │  - kubelet      │    │  - kubelet      │
│  - etcd         │    │  - kube-proxy   │    │  - kube-proxy   │
│  - Controller   │    │  - CNI          │    │  - CNI          │
│  - Scheduler    │    │  - Container    │    │  - Container    │
│  - kubelet      │    │    Runtime      │    │    Runtime      │
│  - kube-proxy   │    │                 │    │                 │
│  - CNI          │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## 📋 Prerequisites

### Hardware Requirements (Per Node)
- 2GB+ RAM (4GB recommended)
- 2+ CPU cores
- 20GB+ disk space
- Network connectivity between all nodes

### Network Requirements
- All nodes must be able to communicate
- Unique hostname for each node
- Unique MAC address for each node
- Port 6443 open on control plane (API server)
- Ports 10250-10252 open on all nodes (kubelet)
- Port 2379-2380 open on control plane (etcd)

## 🚀 Step-by-Step Setup

### Step 1: Prepare All Nodes

Run these commands on **ALL** nodes (control plane and workers):

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Install Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Configure Docker daemon
sudo usermod -aG docker $USER
sudo systemctl enable docker
sudo systemctl start docker

# Install Kubernetes components
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Configure containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# Set unique hostnames (run on each node with appropriate name)
sudo hostnamectl set-hostname k8s-master    # On control plane
sudo hostnamectl set-hostname k8s-worker1   # On worker 1
sudo hostnamectl set-hostname k8s-worker2   # On worker 2

# Update /etc/hosts on all nodes
cat <<EOF | sudo tee -a /etc/hosts
192.168.1.100 k8s-master
192.168.1.101 k8s-worker1
192.168.1.102 k8s-worker2
EOF
```

### Step 2: Initialize Control Plane

Run on the **control plane node only**:

```bash
# Initialize cluster
sudo kubeadm init \
  --apiserver-advertise-address=192.168.1.100 \
  --pod-network-cidr=10.244.0.0/16 \
  --node-name k8s-master \
  --control-plane-endpoint=k8s-master

# Setup kubectl for regular user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Save the join command (you'll need this for workers)
sudo kubeadm token create --print-join-command > join-command.txt
```

### Step 3: Install CNI Plugin

Run on the **control plane node**:

```bash
# Install Flannel CNI
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Wait for flannel pods to be ready
kubectl wait --for=condition=ready pod -l app=flannel -n kube-flannel --timeout=300s
```

### Step 4: Join Worker Nodes

Run on **each worker node**:

```bash
# Get the join command from control plane
# Copy the output from join-command.txt or run this on control plane:
sudo kubeadm token create --print-join-command

# Example join command (use the actual command from your control plane):
sudo kubeadm join k8s-master:6443 --token abcdef.1234567890abcdef \
    --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef \
    --node-name k8s-worker1  # Use appropriate worker name
```

## ✅ Verification

### Check Cluster Status

Run on the **control plane**:

```bash
# Check all nodes
kubectl get nodes -o wide

# Check system pods
kubectl get pods -n kube-system

# Check cluster info
kubectl cluster-info

# Verify node roles
kubectl get nodes --show-labels
```

### Expected Output
```bash
$ kubectl get nodes
NAME          STATUS   ROLES           AGE   VERSION
k8s-master    Ready    control-plane   10m   v1.28.0
k8s-worker1   Ready    <none>          5m    v1.28.0
k8s-worker2   Ready    <none>          3m    v1.28.0

$ kubectl get pods -n kube-system
NAME                                READY   STATUS    RESTARTS   AGE
coredns-5d78c9869d-abc123          1/1     Running   0          10m
coredns-5d78c9869d-def456          1/1     Running   0          10m
etcd-k8s-master                    1/1     Running   0          10m
kube-apiserver-k8s-master          1/1     Running   0          10m
kube-controller-manager-k8s-master 1/1     Running   0          10m
kube-flannel-ds-amd64-ghi789       1/1     Running   0          7m
kube-flannel-ds-amd64-jkl012       1/1     Running   0          5m
kube-flannel-ds-amd64-mno345       1/1     Running   0          3m
kube-proxy-pqr678                  1/1     Running   0          10m
kube-proxy-stu901                  1/1     Running   0          5m
kube-proxy-vwx234                  1/1     Running   0          3m
kube-scheduler-k8s-master          1/1     Running   0          10m
```

## 🧪 Test Multi-Node Deployment

### Deploy Nginx with Multiple Replicas

```bash
# Create deployment with 4 replicas
kubectl create deployment nginx-multinode --image=nginx:latest --replicas=4

# Check pod distribution
kubectl get pods -o wide

# Expose the deployment
kubectl expose deployment nginx-multinode --port=80 --type=NodePort

# Get service details
kubectl get svc nginx-multinode

# Test from any node
curl http://192.168.1.100:$(kubectl get svc nginx-multinode -o jsonpath='{.spec.ports[0].nodePort}')
```

### Deploy DaemonSet

```bash
# Create a DaemonSet that runs on all nodes
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
        hostNetwork: true
        hostPID: true
EOF

# Verify DaemonSet
kubectl get daemonset
kubectl get pods -l app=node-exporter -o wide
```

### Test Pod-to-Pod Communication

```bash
# Create test pods on different nodes
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-1
spec:
  containers:
  - name: test-container
    image: busybox
    command: ['sh', '-c', 'sleep 3600']
  nodeSelector:
    kubernetes.io/hostname: k8s-worker1
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-2
spec:
  containers:
  - name: test-container
    image: busybox
    command: ['sh', '-c', 'sleep 3600']
  nodeSelector:
    kubernetes.io/hostname: k8s-worker2
EOF

# Get pod IPs
kubectl get pods -o wide

# Test connectivity (replace IPs with actual pod IPs)
kubectl exec test-pod-1 -- ping -c 3 <test-pod-2-ip>
kubectl exec test-pod-2 -- ping -c 3 <test-pod-1-ip>
```

## 🔧 Advanced Configuration

### Configure Node Labels

```bash
# Add labels to worker nodes
kubectl label nodes k8s-worker1 node-type=compute
kubectl label nodes k8s-worker2 node-type=storage

# Deploy pods with node affinity
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: compute-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: compute-app
  template:
    metadata:
      labels:
        app: compute-app
    spec:
      containers:
      - name: app
        image: nginx
      nodeSelector:
        node-type: compute
EOF
```

### Setup Node Taints and Tolerations

```bash
# Taint a node
kubectl taint nodes k8s-worker1 special=true:NoSchedule

# Create deployment with toleration
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: special-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: special-app
  template:
    metadata:
      labels:
        app: special-app
    spec:
      containers:
      - name: app
        image: nginx
      tolerations:
      - key: "special"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
EOF

# Remove taint when done
kubectl taint nodes k8s-worker1 special=true:NoSchedule-
```

## 🔍 Monitoring and Maintenance

### Check Resource Usage

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods --all-namespaces

# Detailed node information
kubectl describe nodes
```

### Backup etcd

```bash
# On control plane node
sudo ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
sudo ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db
```

## 🛠 Troubleshooting

### Common Issues

#### 1. Worker Node Not Joining
```bash
# Check if ports are accessible from worker
telnet k8s-master 6443

# Check firewall rules
sudo ufw status
sudo iptables -L

# Check kubelet logs on worker
sudo journalctl -xeu kubelet
```

#### 2. Pods Stuck in Pending
```bash
# Check node resources
kubectl describe nodes
kubectl top nodes

# Check pod events
kubectl describe pod <pod-name>

# Check scheduler logs
kubectl logs -n kube-system kube-scheduler-k8s-master
```

#### 3. Network Issues
```bash
# Check CNI pods
kubectl get pods -n kube-flannel

# Check node network
ip route show
ip addr show

# Test DNS
kubectl run test-dns --image=busybox --rm -it -- nslookup kubernetes.default
```

### Adding New Worker Nodes

```bash
# On control plane, create new token
sudo kubeadm token create --print-join-command

# On new worker node, run the join command
sudo kubeadm join k8s-master:6443 --token <new-token> \
    --discovery-token-ca-cert-hash sha256:<hash>

# Verify new node
kubectl get nodes
```

### Removing Worker Nodes

```bash
# Drain the node
kubectl drain k8s-worker2 --ignore-daemonsets --delete-emptydir-data

# Delete from cluster
kubectl delete node k8s-worker2

# On the worker node, reset kubeadm
sudo kubeadm reset -f
```

## 🧹 Cleanup

### Reset Entire Cluster

```bash
# On all nodes
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube/config
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
```

## 📚 Practice Exercises

1. **Basic Setup**: Create 3-node cluster from scratch
2. **Node Management**: Add/remove worker nodes
3. **Load Balancing**: Deploy apps across multiple nodes
4. **Networking**: Test cross-node pod communication
5. **Storage**: Configure persistent volumes across nodes
6. **Monitoring**: Set up cluster monitoring
7. **Scaling**: Scale deployments across nodes
8. **Upgrades**: Perform rolling cluster upgrades
9. **Backup/Restore**: Practice etcd backup and restore
10. **Troubleshooting**: Simulate and fix network issues

## 🎯 Next Steps

After mastering multi-node setup:
1. High Availability (multiple control planes)
2. Advanced networking (Calico, custom CNI)
3. Storage solutions (NFS, Ceph, cloud storage)
4. Monitoring and logging (Prometheus, Grafana, ELK)
5. Security hardening
6. Production best practices

## 📖 Additional Resources

- [Kubernetes Multi-Node Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Networking Concepts](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Troubleshooting Guide](../troubleshooting/common-issues.md)