# Complete kubeadm Kubernetes Cluster Setup on AWS EC2 – Amazon Linux 2023

Lab: Self-managed Kubernetes cluster using kubeadm, containerd, and Calico on three Amazon Linux 2023 EC2 instances.

This guide covers AWS infrastructure setup, EC2 instance launch, security groups, Kubernetes installation, cluster initialization, worker joining, application deployment, verification, troubleshooting, and cleanup.

## 1. Architecture and requirements

AWS VPC

10.0.0.0/16

Subnet: 10.0.1.0/24

Security group: kubeadm-sg

master

10.0.1.10

Control Plane

worker1

10.0.1.11

Worker

worker2

10.0.1.12

Worker

Kubernetes Pod CIDR: 192.168.0.0/16

| Component         | Configuration       |
| ----------------- | ------------------- |
| Cloud             | AWS                 |
| Operating system  | Amazon Linux 2023   |
| Kubernetes        | v1.34 (lab version) |
| Container runtime | containerd          |
| Cluster bootstrap | kubeadm             |
| Networking        | Calico with VXLAN   |
| Control plane     | 1 EC2 instance      |
| Worker nodes      | 2 EC2 instances     |
| Instance type     | t3.medium           |
| Storage           | 30 GiB gp3 each     |
| Region            | us-east-1 (example) |

Use a currently supported Kubernetes version for a new long-lived deployment. This lab retains v1.34 for consistency with the earlier instructions.

## 2. Create AWS networking

For this practice, create a dedicated VPC and public subnet using the AWS Console.

### Step 1: Create VPC

AWS Console → VPC → Create VPC.

| Setting             | Value         |
| ------------------- | ------------- |
| Resources to create | VPC only      |
| Name                | `kubeadm-vpc` |
| IPv4 CIDR           | `10.0.0.0/16` |
| Tenancy             | Default       |

### Step 2: Create subnet

VPC → Subnets → Create subnet.

| Setting           | Value                   |
| ----------------- | ----------------------- |
| Name              | `kubeadm-subnet`        |
| VPC               | `kubeadm-vpc`           |
| IPv4 CIDR         | `10.0.1.0/24`           |
| Availability Zone | Any one AZ in us-east-1 |

### Step 3: Configure Internet Gateway

VPC → Internet Gateways → Create internet gateway.

Name: `kubeadm-igw`

Attach it to `kubeadm-vpc`.

### Step 4: Configure route table

VPC → Route Tables → Create route table.

Name: `kubeadm-public-rt`

Add route:

| Destination   | Target            |
| ------------- | ----------------- |
| `10.0.0.0/16` | local (automatic) |
| `0.0.0.0/0`   | `kubeadm-igw`     |

Associate `kubeadm-subnet` with this route table.

Enable auto-assign public IPv4 addresses for the subnet if you want each instance reachable directly over SSH.

## 3. Create security group

EC2 → Security Groups → Create security group.

Name: `kubeadm-sg`

VPC: `kubeadm-vpc`

### Inbound rules

| Protocol | Port        | Source               | Purpose                 |
| -------- | ----------- | -------------------- | ----------------------- |
| TCP      | 22          | Your public IP `/32` | SSH                     |
| All      | All         | `kubeadm-sg`         | Private cluster traffic |
| TCP      | 30000–32767 | Your public IP `/32` | Optional NodePort       |

Outbound: Allow all traffic for this learning environment.

Do not expose Kubernetes API port 6443 or kubelet port 10250 publicly.

## 4. Launch EC2 instances

EC2 → Instances → Launch instances.

Launch each instance separately to assign its private IP.

| Setting        | Master         | Worker 1       | Worker 2       |
| -------------- | -------------- | -------------- | -------------- |
| Name           | master         | worker1        | worker2        |
| AMI            | AL2023         | AL2023         | AL2023         |
| Architecture   | x86_64         | x86_64         | x86_64         |
| Instance type  | t3.medium      | t3.medium      | t3.medium      |
| Key pair       | kubeadm-key    | kubeadm-key    | kubeadm-key    |
| VPC            | kubeadm-vpc    | kubeadm-vpc    | kubeadm-vpc    |
| Subnet         | kubeadm-subnet | kubeadm-subnet | kubeadm-subnet |
| Private IP     | 10.0.1.10      | 10.0.1.11      | 10.0.1.12      |
| Public IP      | Enable         | Enable         | Enable         |
| Security group | kubeadm-sg     | kubeadm-sg     | kubeadm-sg     |
| Storage        | 30 GiB gp3     | 30 GiB gp3     | 30 GiB gp3     |

In Advanced network configuration, specify each primary private IP and ensure the address is available.

Launch all three instances and wait for the EC2 status checks to pass.

## 5. SSH into instances

On your local computer:

```
chmod 400 kubeadm-key.pem
```

Master:

```
ssh -i kubeadm-key.pem ec2-user@<MASTER_PUBLIC_IP>
```

Worker 1:

```
ssh -i kubeadm-key.pem ec2-user@<WORKER1_PUBLIC_IP>
```

Worker 2:

```
ssh -i kubeadm-key.pem ec2-user@<WORKER2_PUBLIC_IP>
```

Verify on each node:

```
cat /etc/os-release
hostname
hostname -I
nproc
free -h
lsblk
```

## 6. Configure hostnames

Run on Master only:

```
sudo hostnamectl set-hostname master
```

Run on Worker 1 only:

```
sudo hostnamectl set-hostname worker1
```

Run on Worker 2 only:

```
sudo hostnamectl set-hostname worker2
```

On all three nodes:

```
sudo tee -a /etc/hosts <<'EOF'
10.0.1.10 master
10.0.1.11 worker1
10.0.1.12 worker2
EOF
```

Verify:

```
hostname
getent hosts master
getent hosts worker1
getent hosts worker2
```

## 7. Configure Linux prerequisites

Run on all three nodes.

Update packages:

```
sudo dnf update -y
```

Reboot if required by the update, then reconnect over SSH.

Disable swap:

```
sudo swapoff -a
swapon --show
```

If `/etc/fstab` contains active swap entries, comment them out so swap stays disabled after reboot.

Load kernel modules:

```
sudo tee /etc/modules-load.d/k8s.conf <<'EOF'
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Configure networking:

```
sudo tee /etc/sysctl.d/k8s.conf <<'EOF'
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

Verify:

```
lsmod | grep -E 'overlay|br_netfilter'
sysctl net.ipv4.ip_forward
```

Expected IP forwarding value: `1`.

## 8. Install containerd

Run on all nodes.

```
sudo dnf install -y containerd
```

If the package is not available in your enabled AL2023 repositories, check:

```
sudo dnf list --available containerd
sudo dnf info docker
```

The Amazon Linux Docker package can provide containerd as a dependency:

```
sudo dnf install -y docker
```

Only use the Docker package fallback if necessary; kubeadm uses containerd directly in this lab.

Configure containerd:

```
sudo mkdir -p /etc/containerd

containerd config default | \
sudo tee /etc/containerd/config.toml > /dev/null
```

Enable systemd cgroups:

```
sudo sed -i \
's/SystemdCgroup = false/SystemdCgroup = true/' \
/etc/containerd/config.toml
```

For containerd 1.x, ensure the CRI plugin is enabled. For containerd 2.x, inspect the generated configuration because the CRI plugin layout differs.

Start the service:

```
sudo systemctl enable --now containerd
sudo systemctl restart containerd
```

Verify:

```
containerd --version
sudo systemctl is-active containerd
sudo grep SystemdCgroup /etc/containerd/config.toml
```

The service should report `active`.

## 9. Install Kubernetes packages

Run on all nodes.

Create the Kubernetes v1.34 repository:

```
sudo tee /etc/yum.repos.d/kubernetes.repo <<'EOF'
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
EOF
```

Install packages:

```
sudo dnf install -y \
  kubelet kubeadm kubectl \
  --disableexcludes=kubernetes
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

The kubelet may restart until the cluster is initialized or the node joins.

## 10. Initialize Kubernetes control plane

Run only on `master`.

```
sudo kubeadm init \
  --apiserver-advertise-address=10.0.1.10 \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

Wait for successful initialization.

Configure kubectl for `ec2-user`:

```
mkdir -p "$HOME/.kube"

sudo cp /etc/kubernetes/admin.conf \
  "$HOME/.kube/config"

sudo chown "$(id -u):$(id -g)" \
  "$HOME/.kube/config"
```

Verify:

```
kubectl get nodes
kubectl get pods -A
```

The master may show `NotReady` until Calico networking is installed.

## 11. Install Calico networking

Run only on `master`.

Use Calico v3.33.0 with the selected Kubernetes v1.34 cluster. The following operator-based installation follows Calico's official documentation.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.tigera.io\&sz=32)

Calico Documentation

+1



Install Calico CRDs:

```
kubectl create -f \
https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/v3_projectcalico_org-v1beta1.yaml
```

Install Tigera Operator:

```
kubectl create -f \
https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/tigera-operator.yaml
```

Create the Calico configuration:

```
cat > custom-resources.yaml <<'EOF'
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    bgp: Disabled
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
EOF
```

Apply:

```
kubectl create -f custom-resources.yaml
```

Verify:

```
kubectl get pods -n tigera-operator
kubectl get pods -n calico-system
kubectl get tigerastatus
kubectl get nodes
```

Wait for Calico components to become healthy. VXLAN requires UDP 4789 between nodes, permitted by the self-referencing cluster security group.

## 12. Join worker nodes

On Master, generate the join command:

```
kubeadm token create --print-join-command
```

Run the generated command on `worker1` and `worker2` using `sudo`.

Example format:

```
sudo kubeadm join 10.0.1.10:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --cri-socket=unix:///run/containerd/containerd.sock
```

Replace the placeholders with the actual generated values.

Verify from Master:

```
kubectl get nodes -o wide
```

Expected example:

```
NAME      STATUS   ROLES           VERSION
master    Ready    control-plane   v1.34.x
worker1   Ready    <none>          v1.34.x
worker2   Ready    <none>          v1.34.x
```

## 13. Deploy Nginx application

On Master:

```
kubectl create deployment nginx \
  --image=nginx:stable \
  --replicas=3
```

Expose the deployment:

```
kubectl expose deployment nginx \
  --type=NodePort \
  --port=80 \
  --target-port=80
```

Verify:

```
kubectl get deployments
kubectl get pods -o wide
kubectl get svc nginx
```

Example service output:

```
NAME    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)
nginx   NodePort   10.96.50.20    <none>        80:31234/TCP
```

Use the actual NodePort assigned to your service.

Test from an EC2 instance:

```
curl http://10.0.1.11:31234
```

To access Nginx from your laptop, use:

```
http://<NODE_PUBLIC_IP>:<NODEPORT>
```

Ensure the EC2 security group permits that NodePort from your trusted public IP. Kubernetes NodePort does not automatically provision an AWS Load Balancer.

## 14. Kubernetes administration practice

```
# Cluster information
kubectl cluster-info

# Nodes
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node worker1

# Pods
kubectl get pods -A
kubectl get pods -o wide

# Deployments
kubectl get deployments
kubectl describe deployment nginx

# Scale application
kubectl scale deployment nginx --replicas=5

# Check rollout
kubectl rollout status deployment/nginx

# Stop scheduling new pods on worker1
kubectl cordon worker1

# Drain worker1
kubectl drain worker1 \
  --ignore-daemonsets \
  --delete-emptydir-data

# Enable scheduling again
kubectl uncordon worker1

# Services
kubectl get svc
kubectl describe svc nginx
```

## 15. Troubleshooting commands

Run on the affected EC2 instance:

```
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 100 --no-pager

sudo systemctl status containerd
sudo journalctl -u containerd -n 100 --no-pager
```

If `crictl` is installed:

```
sudo crictl \
  --runtime-endpoint unix:///run/containerd/containerd.sock \
  ps -a
```

From Master:

```
kubectl get nodes -o wide
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl cluster-info
```

| Issue                  | Check                                     |
| ---------------------- | ----------------------------------------- |
| `kubeadm init` fails   | Swap, containerd, kubelet, port conflicts |
| Worker cannot join     | Port 6443, join token, private networking |
| Node `NotReady`        | Calico, CNI configuration, kubelet        |
| Pod `Pending`          | Scheduling, node resources, taints        |
| `ImagePullBackOff`     | Image name, registry access               |
| NodePort not reachable | Security group, NACL, service endpoints   |

## 16. Delete Kubernetes cluster

On Master, delete the test application:

```
kubectl delete deployment nginx
kubectl delete svc nginx
```

On all three nodes:

```
sudo kubeadm reset -f
```

For a dedicated disposable lab, clean remaining CNI files:

```
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni
sudo rm -rf "$HOME/.kube"
```

Reboot the instances before rebuilding if residual networking state causes issues.

Then terminate the three EC2 instances and delete the dedicated security group, subnet, route table, Internet Gateway, and VPC when no longer required.

## Official references

- [AWS EC2 Getting Started](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html?utm_source=chatgpt.com)
- [Install kubeadm, kubelet, and kubectl](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/?utm_source=chatgpt.com)
- [Create a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/?utm_source=chatgpt.com)
- [Calico installation guide](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises?utm_source=chatgpt.com)

Result: This guide provides the full procedure for building a three-node self-managed Kubernetes lab on AWS EC2. The commands have been reviewed against the documented installation approach, but the deployment has not been executed in your AWS account.
