# kubeadm Practice on Amazon Linux 2023 (AWS EC2)

Goal: Create a self-managed Kubernetes cluster using `kubeadm` on three Amazon Linux 2023 EC2 instances.

## 1. AWS EC2 requirements

| Instance      | Hostname | Instance type | Private IP example |
| ------------- | -------- | ------------- | ------------------ |
| Control Plane | master   | t3.medium     | 10.0.1.10          |
| Worker Node 1 | worker1  | t3.medium     | 10.0.1.11          |
| Worker Node 2 | worker2  | t3.medium     | 10.0.1.12          |

Use the same VPC, with private connectivity between nodes. Allow SSH from your trusted IP and Kubernetes inter-node traffic through a dedicated security group. For a lab, allowing all traffic between instances sharing the cluster security group is simplest; do not expose all ports publicly.

## 2. Configure all three EC2 instances

Run on Master and both Workers.

Set hostnames separately:

```
sudo hostnamectl set-hostname master
sudo hostnamectl set-hostname worker1
sudo hostnamectl set-hostname worker2
```

Add actual private IPs on every node:

```
sudo vi /etc/hosts
```

```
10.0.1.10 master
10.0.1.11 worker1
10.0.1.12 worker2
```

Disable swap and configure networking:

```
sudo swapoff -a

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

## 3. Install containerd on all nodes

```
sudo dnf update -y
sudo dnf install -y containerd
```

```
sudo mkdir -p /etc/containerd

containerd config default | \
sudo tee /etc/containerd/config.toml > /dev/null
```

```
sudo sed -i \
's/SystemdCgroup = false/SystemdCgroup = true/' \
/etc/containerd/config.toml
```

```
sudo systemctl enable --now containerd
sudo systemctl restart containerd
sudo systemctl status containerd
```

Ensure the CRI plugin is enabled in the containerd configuration.

## 4. Install kubeadm, kubelet and kubectl

Use the Kubernetes v1.34 RPM repository for this example. All nodes must use compatible Kubernetes versions.

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
EOF
```

```
sudo dnf install -y kubelet kubeadm kubectl \
  --disableexcludes=kubernetes
```

```
sudo systemctl enable --now kubelet
```

Verify:

```
kubeadm version
kubectl version --client
kubelet --version
```

The kubelet may restart until the node has joined a cluster.

## 5. Initialize Kubernetes Master

Run only on the Master node.

```
sudo kubeadm init \
  --apiserver-advertise-address=10.0.1.10 \
  --pod-network-cidr=192.168.0.0/16
```

Configure kubectl:

```
mkdir -p $HOME/.kube

sudo cp /etc/kubernetes/admin.conf \
  $HOME/.kube/config

sudo chown $(id -u):$(id -g) \
  $HOME/.kube/config
```

Verify:

```
kubectl get nodes
kubectl get pods -A
```

## 6. Install Calico networking

On the Master, install a Calico version compatible with your Kubernetes cluster using the official [Calico installation guide](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart?utm_source=chatgpt.com).

For this lab, use a Calico configuration matching `192.168.0.0/16` and configure encapsulation appropriately for AWS VPC networking.

Verify:

```
kubectl get pods -A
kubectl get nodes
```

## 7. Join Worker Nodes

On Master:

```
kubeadm token create --print-join-command
```

Run the generated command on each Worker node, using `sudo`.

Example format:

```
sudo kubeadm join 10.0.1.10:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

Verify on Master:

```
kubectl get nodes -o wide
```

Expected:

```
NAME      STATUS   ROLES
master    Ready    control-plane
worker1   Ready    <none>
worker2   Ready    <none>
```

## 8. Deploy a test application

```
kubectl create deployment nginx \
  --image=nginx:stable \
  --replicas=3
```

```
kubectl expose deployment nginx \
  --type=NodePort \
  --port=80
```

```
kubectl get pods -o wide
kubectl get deployments
kubectl get svc
```

For external testing, use an EC2 node's reachable IP and the assigned NodePort, with a narrowly scoped security-group rule. A `NodePort` service does not automatically create an AWS load balancer.

## 9. Kubernetes administration practice

```
kubectl get nodes
kubectl describe node worker1
kubectl get pods -A
kubectl scale deployment nginx --replicas=5
kubectl rollout status deployment/nginx
kubectl cordon worker1
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon worker1
```

## 10. Troubleshooting

```
sudo systemctl status kubelet
sudo journalctl -u kubelet -f
sudo systemctl status containerd
sudo crictl ps -a
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl cluster-info
```

## 11. Delete the cluster

Run on all nodes:

```
sudo kubeadm reset -f
```

For full cleanup, remove remaining CNI and networking state before rebuilding.

AWS-specific points: Use Amazon Linux 2023, not Amazon Linux 2; keep the API server port `6443` private; ensure VPC and pod CIDRs do not overlap; use Calico encapsulation or appropriate VPC routing; and remember that kubeadm does not install AWS cloud-controller-manager, EBS CSI, or an AWS Load Balancer Controller automatically.

Official reference: [Kubernetes kubeadm installation documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/?utm_source=chatgpt.com).
