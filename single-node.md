# Kubernetes Single-Node Cluster Using kubeadm on Amazon Linux 2023

This guide sets up a single-node Kubernetes cluster on an AWS EC2 instance running Amazon Linux 2023 using `kubeadm`, `kubelet`, `kubectl`, and `containerd`.

The same EC2 instance will function as both the control plane and worker node, making it suitable for Kubernetes practice and training.

## 1. AWS EC2 configuration

| Setting                 | Value                                    |
| ----------------------- | ---------------------------------------- |
| AWS Region              | US East (N. Virginia) – `us-east-1`      |
| AMI                     | Amazon Linux 2023                        |
| Instance type           | `t3.medium` (minimum practical lab size) |
| Recommended             | `t3.large`                               |
| Storage                 | 30 GB gp3                                |
| Key pair                | `eks.pem`                                |
| Node count              | 1                                        |
| Container runtime       | containerd                               |
| Kubernetes installation | kubeadm                                  |
| Networking              | VPC with internet access                 |

Security group: Allow SSH (TCP 22) only from your public IP. For remote Kubernetes API access, allow TCP 6443 only from trusted IP addresses. Keep the Kubernetes API private if remote access is unnecessary.

## 2. Connect to EC2

```
chmod 400 eks.pem

ssh -i eks.pem ec2-user@<EC2-PUBLIC-IP>
```

## 3. Update Amazon Linux

```
sudo dnf update -y

cat /etc/os-release

uname -r
```

## 4. Configure hostname

```
sudo hostnamectl set-hostname k8s-master

hostnamectl
```

## 5. Disable swap

```
sudo swapoff -a

sudo sed -i.bak '/[[:space:]]swap[[:space:]]/s/^/#/' /etc/fstab

free -h
```

## 6. Load required kernel modules

```
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Configure networking:

```
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

Verify:

```
lsmod | grep br_netfilter

sysctl net.ipv4.ip_forward
```

## 7. Install containerd

```
sudo dnf install -y containerd

sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml
```

Enable systemd cgroups:

```
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Ensure the CRI plugin is not disabled:

```
grep -n 'disabled_plugins\|SystemdCgroup' /etc/containerd/config.toml
```

If `disabled_plugins` includes `cri`, remove `cri` from that setting.

Start containerd:

```
sudo systemctl enable --now containerd

sudo systemctl restart containerd

sudo systemctl status containerd
```

## 8. Configure SELinux for the lab

```
getenforce

sudo setenforce 0
```

For persistent permissive mode, if SELinux is configured:

```
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

This is suitable for a learning environment, not a production security baseline.

## 9. Install kubeadm, kubelet and kubectl

Use Kubernetes v1.36 for this lab.

```
sudo tee /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

Install:

```
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

Enable kubelet:

```
sudo systemctl enable --now kubelet
```

Verify:

```
kubeadm version

kubelet --version

kubectl version --client
```

The kubelet may restart repeatedly before cluster initialization. This is expected.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://kubernetes.io\&sz=32)

Kubernetes+1



## 10. Initialize the Kubernetes cluster

Get the EC2 private IP:

```
PRIVATE_IP=$(hostname -I | awk '{print $1}')

echo "$PRIVATE_IP"
```

Initialize the control plane:

```
sudo kubeadm init \
  --apiserver-advertise-address="$PRIVATE_IP" \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

Wait until you see:

```
Your Kubernetes control-plane has initialized successfully!
```

## 11. Configure kubectl access

Run as `ec2-user`:

```
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify:

```
kubectl cluster-info

kubectl get nodes
```

The node may initially show `NotReady` because the CNI network plugin has not been installed.

## 12. Install Flannel CNI

```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Check system pods:

```
kubectl get pods -A

kubectl get nodes -o wide
```

Wait for the node to become `Ready`.

## 13. Allow workloads on the control-plane node

By default, Kubernetes prevents ordinary workloads from running on control-plane nodes.

For a single-node lab, remove the taint:

```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Verify:

```
kubectl describe node k8s-master | grep -i taints
```

Expected:

```
Taints: <none>
```

## 14. Deploy an Nginx application

Create a deployment:

```
kubectl create deployment nginx --image=nginx:stable
```

Scale to two replicas:

```
kubectl scale deployment nginx --replicas=2
```

Check:

```
kubectl get deployments

kubectl get pods -o wide
```

Expose Nginx using NodePort:

```
kubectl expose deployment nginx \
  --port=80 \
  --target-port=80 \
  --type=NodePort
```

Get service details:

```
kubectl get svc nginx
```

Example output:

```
NAME    TYPE       CLUSTER-IP      PORT(S)
nginx   NodePort   10.96.100.10    80:31234/TCP
```

The NodePort is dynamically allocated, so your actual value may differ.

## 15. Access Nginx

Find the allocated port:

```
kubectl get svc nginx
```

Test locally using the EC2 private IP and actual NodePort:

```
NODE_PORT=$(kubectl get svc nginx -o jsonpath='{.spec.ports[0].nodePort}')

curl "http://$(hostname -I | awk '{print $1}'):$NODE_PORT"
```

To access Nginx from a browser using the EC2 public IP, add an inbound security-group rule for the allocated TCP NodePort, restricted to your public IP.

Alternatively, avoid opening a NodePort by using SSH port forwarding.

On EC2:

```
kubectl port-forward svc/nginx 8080:80 --address 127.0.0.1
```

On your Mac, in another terminal:

```
ssh -i eks.pem -L 8080:127.0.0.1:8080 ec2-user@<EC2-PUBLIC-IP>
```

Open `http://localhost:8080` on your Mac.

## 16. Kubernetes practice commands

```
kubectl get nodes

kubectl get pods -A

kubectl get deployments

kubectl get services

kubectl get pods -o wide

kubectl describe deployment nginx

kubectl logs deployment/nginx

kubectl scale deployment nginx --replicas=3

kubectl rollout status deployment/nginx

kubectl get events -A --sort-by=.lastTimestamp
```

## 17. Troubleshooting

Check services:

```
sudo systemctl status kubelet

sudo systemctl status containerd
```

View logs:

```
sudo journalctl -u kubelet -n 100 --no-pager

sudo journalctl -u containerd -n 100 --no-pager
```

Check networking:

```
kubectl get pods -n kube-flannel

kubectl get nodes -o wide
```

Check container runtime:

```
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock info
```

If `kubeadm init` fails, inspect its error before resetting the cluster.

For a complete fresh reset (deletes the existing cluster configuration and workloads):

```
sudo kubeadm reset -f

rm -f $HOME/.kube/config

sudo rm -rf /etc/cni/net.d
```

Then reinitialize and reinstall Flannel.

## 18. Final verification

```
kubectl cluster-info

kubectl get nodes -o wide

kubectl get pods -A

kubectl get deployments

kubectl get svc
```

Expected cluster layout:

AWS EC2 — Amazon Linux 2023

Single Node: k8s-master

Kubernetes Control Plane

API Server · Scheduler · Controller Manager · etcd

Worker Components

kubelet · containerd · kube-proxy · Flannel

Nginx Pod 1

Nginx Pod 2

Result: One Amazon Linux EC2 instance hosts the Kubernetes control plane and application workloads. This is suitable for hands-on learning with Pods, Deployments, Services, ConfigMaps, Secrets, and rolling updates, but not for high availability or production use.

Official reference: [Kubernetes kubeadm installation documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/?utm_source=chatgpt.com).
