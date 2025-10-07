# Single Node Kubernetes Cluster Setup

Complete guide for setting up a single-node Kubernetes cluster using kubeadm.

## 🎯 Objectives

- Set up a single-node Kubernetes cluster
- Configure the control plane to run pods
- Install a CNI plugin
- Deploy sample applications

## 📋 Prerequisites

### System Requirements
- Ubuntu 20.04+ or CentOS 7+
- 2GB+ RAM
- 2+ CPU cores
- 20GB+ disk space
- Internet connectivity

### Software Requirements
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

# Configure Docker
sudo usermod -aG docker $USER
sudo systemctl enable docker
sudo systemctl start docker
```

## 🚀 Step-by-Step Setup

### Step 1: Install Kubernetes Components

```bash
# Add Kubernetes repository
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install kubelet, kubeadm, and kubectl
sudo apt update
sudo apt install -y kubelet kubeadm kubectl

# Hold packages to prevent automatic updates
sudo apt-mark hold kubelet kubeadm kubectl
```

### Step 2: Configure Container Runtime

```bash
# Configure containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Step 3: System Configuration

```bash
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
```

### Step 4: Initialize Cluster

```bash
# Initialize the cluster
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=$(hostname -I | awk '{print $1}') \
  --node-name $(hostname)

# Setup kubectl for user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Step 5: Remove Control Plane Taint

```bash
# Allow pods to be scheduled on control plane (single node)
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Step 6: Install CNI Plugin

#### Option A: Flannel
```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

#### Option B: Calico
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
```

#### Option C: Weave Net
```bash
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

## ✅ Verification

### Check Cluster Status
```bash
# Check nodes
kubectl get nodes -o wide

# Check system pods
kubectl get pods -n kube-system

# Check cluster info
kubectl cluster-info

# Verify all pods are running
kubectl get pods --all-namespaces
```

### Expected Output
```bash
$ kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
master     Ready    control-plane   5m    v1.28.0

$ kubectl get pods -n kube-system
NAME                                READY   STATUS    RESTARTS   AGE
coredns-5d78c9869d-abc123          1/1     Running   0          5m
coredns-5d78c9869d-def456          1/1     Running   0          5m
etcd-master                        1/1     Running   0          5m
kube-apiserver-master              1/1     Running   0          5m
kube-controller-manager-master     1/1     Running   0          5m
kube-flannel-ds-xyz789             1/1     Running   0          3m
kube-proxy-ghi012                  1/1     Running   0          5m
kube-scheduler-master              1/1     Running   0          5m
```

## 🧪 Test Deployments

### Deploy Sample Application
```bash
# Create nginx deployment
kubectl create deployment nginx --image=nginx:latest --replicas=2

# Expose deployment
kubectl expose deployment nginx --port=80 --type=NodePort

# Check deployment
kubectl get deployments
kubectl get pods
kubectl get services

# Test access
curl http://$(hostname -I | awk '{print $1}'):$(kubectl get svc nginx -o jsonpath='{.spec.ports[0].nodePort}')
```

### Deploy Sample Pod
```bash
# Create a simple pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: test-container
    image: busybox
    command: ['sh', '-c', 'echo "Hello from single node cluster!" && sleep 3600']
EOF

# Check pod
kubectl get pods
kubectl logs test-pod
```

## 🔍 Monitoring and Logs

### Useful Commands
```bash
# View kubelet logs
sudo journalctl -xeu kubelet

# View container runtime logs
sudo journalctl -xeu containerd

# Check resource usage
kubectl top nodes
kubectl top pods

# Describe node for detailed info
kubectl describe node $(hostname)
```

## 🛠 Troubleshooting

### Common Issues

#### 1. Pod Network Not Ready
```bash
# Check CNI installation
kubectl get pods -n kube-system | grep -E '(flannel|calico|weave)'

# Reinstall CNI if needed
kubectl delete -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

#### 2. Node Not Ready
```bash
# Check node status
kubectl describe node $(hostname)

# Check kubelet status
sudo systemctl status kubelet

# Restart kubelet if needed
sudo systemctl restart kubelet
```

#### 3. DNS Issues
```bash
# Check CoreDNS pods
kubectl get pods -n kube-system | grep coredns

# Test DNS resolution
kubectl run test-dns --image=busybox --rm -it -- nslookup kubernetes.default
```

## 🧹 Cleanup

### Reset Cluster
```bash
# Reset kubeadm
sudo kubeadm reset -f

# Clean up
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube/config
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X

# Stop and disable services
sudo systemctl stop kubelet
sudo systemctl disable kubelet
```

## 📚 Practice Exercises

1. **Setup**: Complete the full single-node setup
2. **Application Deployment**: Deploy different types of workloads
3. **Networking**: Test pod-to-pod communication
4. **Storage**: Create and mount persistent volumes
5. **Monitoring**: Install and configure monitoring tools
6. **Backup**: Practice etcd backup and restore
7. **Upgrade**: Upgrade Kubernetes version
8. **Troubleshooting**: Break and fix various components

## 🎯 Next Steps

After mastering single-node setup:
1. Try different CNI plugins
2. Experiment with various applications
3. Learn about Kubernetes resources
4. Move to multi-node setup
5. Explore cluster administration

## 📖 Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [kubeadm Documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [CNI Plugins Comparison](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Troubleshooting Guide](../troubleshooting/common-issues.md)