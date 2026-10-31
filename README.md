# Kubernetes Cluster Setup Using kubeadm on AWS EC2 – Amazon Linux 2023
Architecture: 1 control-plane EC2 + 2 worker EC2 nodes; containerd runtime; Calico CNI; Nginx NodePort demo. This is a learning environment, not a production high-availability design.

# 1. Lab Configuration

| Setting           | Value                                               |
|-------------------|-----------------------------------------------------|
| AWS Region        | us-east-1 (N. Virginia)                             |
| Availability Zone | us-east-1a (choose an available AZ in your account) |
| VPC               | kubeadm-vpc — 10.0.0.0/16                           |
| Subnet            | kubeadm-public-subnet — 10.0.1.0/24                 |
| Internet Gateway  | kubeadm-igw                                         |
| Route table       | kubeadm-public-rt                                   |
| Security Group    | kubeadm-sg                                          |
| AMI               | Amazon Linux 2023 x86_64                            |
| EC2               | 3 × t3.medium, 20 GiB gp3 each                      |
| Key pair          | eks (private file: eks.pem)                         |
| Kubernetes        | v1.34 (all nodes same minor version)                |
| Pod network       | 192.168.0.0/16                                      |
| Container runtime | containerd                                          |
| CNI               | Calico                                              |

| Node    | Private IP (assign at launch) | Role          |
|---------|-------------------------------|---------------|
| master  | 10.0.1.10                     | Control plane |
| worker1 | 10.0.1.11                     | Worker        |
| worker2 | 10.0.1.12                     | Worker        |

All private IPs are examples that must be assigned explicitly at EC2 launch, or replaced consistently with the actual EC2 private addresses. The pod CIDR must not overlap with VPC networks.

# 2. Create VPC

AWS Console → switch region to US East (N. Virginia) → VPC → Your VPCs → Create VPC. Choose VPC only; name kubeadm-vpc; IPv4 CIDR 10.0.0.0/16; no IPv6; default tenancy. Create and confirm Available.

# 3. Create Public Subnet

VPC → Subnets → Create subnet. VPC kubeadm-vpc; name kubeadm-public-subnet; AZ us-east-1a; IPv4 10.0.1.0/24. After creation: Actions → Edit subnet settings → Enable auto-assign public IPv4 address.

# 4. Internet Gateway and Route Table

VPC → Internet Gateways → Create: kubeadm-igw. Attach to kubeadm-vpc. Then VPC → Route Tables → Create: kubeadm-public-rt in kubeadm-vpc. Add default route 0.0.0.0/0 → kubeadm-igw; retain local route 10.0.0.0/16 → local. Associate kubeadm-public-subnet with this route table.

# 5. Security Group

EC2 → Security Groups → Create kubeadm-sg in kubeadm-vpc. Set inbound SSH TCP 22 from My IP only. Add All traffic from source kubeadm-sg (self-reference) for internal node-to-node communication. For browser testing, optionally allow TCP 30000–32767 from My IP. Retain outbound allow all for this lab. Never allow SSH or Kubernetes control-plane ports from 0.0.0.0/0.

The internal self-reference covers API 6443, etcd 2379–2380, kubelet 10250, scheduler/controller manager and Calico encapsulation. Restrict further only after validating the exact CNI mode and port/protocol requirements.

# 6. Key Pair in us-east-1

EC2 key pairs are region-specific. In us-east-1, create key pair named eks (RSA, PEM) or import the public key matching an existing eks.pem. Keep the private key secure. To derive a public key on macOS:

cd ~/Downloads  
chmod 400 eks.pem  
ssh-keygen -y -f eks.pem \> eks.pub

Then EC2 → Key Pairs → Import key pair → name eks → upload eks.pub. Do not upload eks.pem.

# 7. Launch Three EC2 Instances

EC2 → Launch instances. Use Amazon Linux 2023 AMI (x86_64), t3.medium, key pair eks, VPC kubeadm-vpc, subnet kubeadm-public-subnet, public IP enabled, security group kubeadm-sg and 20 GiB gp3 root volume. Launch master, worker1 and worker2. Under Advanced network configuration set primary private IP to 10.0.1.10, .11 and .12 respectively, or substitute the assigned IPs throughout. Wait for 2/2 status checks.

# 8. SSH and Hostnames

On your Mac, in separate terminal tabs:

cd ~/Downloads  
chmod 400 eks.pem  
ssh -i eks.pem ec2-user@\<master-public-ip\>  
ssh -i eks.pem ec2-user@\<worker1-public-ip\>  
ssh -i eks.pem ec2-user@\<worker2-public-ip\>

Run the following on the corresponding nodes (one command per node):

sudo hostnamectl set-hostname master \# master only  
sudo hostnamectl set-hostname worker1 \# worker1 only  
sudo hostnamectl set-hostname worker2 \# worker2 only

On ALL nodes, append the actual node private IP/name mappings to /etc/hosts (do not replace the existing localhost entries):

cat \<\<'EOF' \| sudo tee -a /etc/hosts  
10.0.1.10 master  
10.0.1.11 worker1  
10.0.1.12 worker2  
EOF  
hostname  
hostname -I  
getent hosts master worker1 worker2

# 9. OS Prerequisites — All Three Nodes

sudo dnf update -y  
sudo dnf install -y curl wget tar conntrack-tools socat iproute-tc  
sudo swapoff -a  
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab  
free -h

## Kernel modules and sysctl

cat \<\<'EOF' \| sudo tee /etc/modules-load.d/k8s.conf  
overlay  
br_netfilter  
EOF  
sudo modprobe overlay  
sudo modprobe br_netfilter  
cat \<\<'EOF' \| sudo tee /etc/sysctl.d/k8s.conf  
net.bridge.bridge-nf-call-iptables = 1  
net.bridge.bridge-nf-call-ip6tables = 1  
net.ipv4.ip_forward = 1  
EOF  
sudo sysctl --system  
sysctl net.ipv4.ip_forward

## SELinux

Check with getenforce. If enforcing, permissive mode is acceptable for this disposable lab; do not use it as a blanket production recommendation.

getenforce  
\# Only if Enforcing:  
sudo setenforce 0  
sudo sed -i 's/^SELINUX=enforcing\$/SELINUX=permissive/' /etc/selinux/config

# 10. Install containerd — All Nodes

Check availability of the containerd package in your AL2023 repositories. If unavailable, use a compatible vendor package for your architecture rather than installing an arbitrary Docker daemon.

sudo dnf list --available containerd  
sudo dnf install -y containerd  
sudo mkdir -p /etc/containerd  
containerd config default \| sudo tee /etc/containerd/config.toml \> /dev/null  
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml  
grep -nE 'SystemdCgroup\|disabled_plugins' /etc/containerd/config.toml  
sudo systemctl enable --now containerd  
sudo systemctl restart containerd  
sudo systemctl status containerd  
containerd --version

Confirm CRI is enabled (not listed under disabled_plugins) and systemd cgroups are enabled. The containerd configuration syntax varies by major version; inspect the generated file before editing.

# 11. Install kubeadm, kubelet, kubectl — All Nodes

cat \<\<'EOF' \| sudo tee /etc/yum.repos.d/kubernetes.repo  
\[kubernetes\]  
name=Kubernetes  
baseurl=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/  
enabled=1  
gpgcheck=1  
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key  
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni  
EOF  
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes  
sudo systemctl enable --now kubelet  
kubeadm version  
kubelet --version  
kubectl version --client

Kubelet may restart until the node is initialized or joined. Use the same Kubernetes minor version on every node.

# 12. Initialize Control Plane — Master Only

hostname -I  
sudo kubeadm init \\  
--apiserver-advertise-address=10.0.1.10 \\  
--pod-network-cidr=192.168.0.0/16 \\  
--cri-socket=unix:///run/containerd/containerd.sock

Replace 10.0.1.10 if the master has a different private IP. Save the join command printed at the end of initialization.

mkdir -p "\$HOME/.kube"  
sudo cp /etc/kubernetes/admin.conf "\$HOME/.kube/config"  
sudo chown "\$(id -u):\$(id -g)" "\$HOME/.kube/config"  
chmod 600 "\$HOME/.kube/config"  
kubectl cluster-info  
kubectl get nodes  
kubectl get pods -A

The master can be NotReady until CNI is installed. Protect ~/.kube/config because it contains cluster-admin credentials.

# 13. Install Calico CNI — Master Only

curl -fsSLo calico.yaml https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/calico.yaml  
grep -nE "CALICO_IPV4POOL_CIDR\|CALICO_IPV4POOL_IPIP\|CALICO_IPV4POOL_VXLAN" calico.yaml  
kubectl apply -f calico.yaml  
kubectl get pods -n kube-system  
kubectl get nodes -w

This example uses the Calico manifest for the 192.168.0.0/16 pod network. Confirm the manifest configuration and CNI compatibility before use. The security-group self-reference allows node-to-node Calico traffic.

# 14. Join Workers — Worker1 and Worker2

On master generate a fresh join command:

kubeadm token create --print-join-command

On EACH worker, run the generated command with sudo; replace the placeholders below with actual generated values:

sudo kubeadm join 10.0.1.10:6443 \\  
--token \<generated-token\> \\  
--discovery-token-ca-cert-hash sha256:\<generated-hash\>

Verify from master:

kubectl get nodes -o wide  
kubectl get pods -A

Expected: master Ready control-plane, worker1 Ready, worker2 Ready. If a node remains NotReady, inspect kubelet logs, CNI pods and security-group rules.

# 15. Deploy Nginx and Expose NodePort — Master Only

kubectl create deployment nginx --image=nginx  
kubectl scale deployment nginx --replicas=3  
kubectl rollout status deployment/nginx  
kubectl get pods -o wide  
kubectl expose deployment nginx --type=NodePort --port=80 --target-port=80  
kubectl get svc nginx

Find the assigned port (normally 30000–32767). Browse http://\<worker-public-ip\>:\<node-port\>. Ensure the SG allows that port from your IP. The example port 31245 is illustrative, not guaranteed.

# 16. Practical Kubernetes Exercises

## Namespaces and Deployment

kubectl create namespace production  
kubectl create deployment webapp --image=nginx -n production  
kubectl scale deployment webapp --replicas=3 -n production  
kubectl get pods -n production -o wide

## Scaling and Self-Healing

kubectl scale deployment nginx --replicas=5  
kubectl get pods -o wide  
kubectl delete pod \<nginx-pod-name\>  
kubectl get pods -w

## Worker Maintenance

kubectl cordon worker1  
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data  
kubectl get pods -o wide  
kubectl uncordon worker1

## Observability

kubectl get events -A  
kubectl describe node worker1  
kubectl logs deployment/nginx  
kubectl get pods -A -o wide

# 17. Troubleshooting

| Symptom             | Check                                           |
|---------------------|-------------------------------------------------|
| Node NotReady       | kubectl describe node worker1                   |
| Kubelet failing     | sudo journalctl -u kubelet -n 100 --no-pager    |
| containerd issue    | sudo journalctl -u containerd -n 100 --no-pager |
| CNI issue           | kubectl get pods -n kube-system                 |
| Pod stuck Pending   | kubectl describe pod \<pod-name\>               |
| Join token expired  | kubeadm token create --print-join-command       |
| Service unavailable | kubectl describe svc nginx                      |
| API network issue   | curl -k https://10.0.1.10:6443                  |

An HTTP authentication-related response from the API endpoint can still indicate the endpoint is reachable. Avoid interpreting it as proof that kubectl credentials work.

# 18. Reset and AWS Cleanup

If rebuilding, drain worker nodes first when practical. Run kubeadm reset on each worker and then on master. This does not completely clean CNI state or networking rules.

sudo kubeadm reset -f  
\# Master only, after reset:  
rm -rf ~/.kube

For complete cost cleanup: terminate all three EC2 instances; delete kubeadm-sg after interfaces are released; delete the custom route table; delete the subnet; detach/delete the internet gateway; delete kubeadm-vpc. Retain the eks key pair if used for other labs. Public IPv4 addresses and EC2 resources may incur charges.

# 19. Verification Checklist

☐ VPC, subnet, IGW and routes created

☐ Security group allows restricted SSH and internal node traffic

☐ Three AL2023 EC2 instances running in us-east-1

☐ SSH succeeds using eks.pem

☐ Kernel modules, sysctl and swap configured

☐ containerd active and CRI enabled

☐ Kubernetes tools installed on all nodes

☐ kubeadm init completed on master

☐ Calico networking healthy

☐ Two workers joined and all nodes Ready

☐ Nginx pods running and NodePort reachable

☐ Scaling, pod self-healing and drain tested

☐ AWS resources cleaned up after lab

# References

Kubernetes kubeadm installation: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

Kubernetes kubeadm cluster creation: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

Calico installation: https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises

AWS VPC documentation: https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html

AWS EC2 key pairs: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html


Region: US East (N. Virginia) – `us-east-1`

The complete guide will use the N. Virginia region throughout, with three Amazon Linux 2023 EC2 instances, the `eks.pem` SSH key, a custom VPC, containerd, Kubernetes v1.34 and Calico networking.

## Final infrastructure configuration

| Component         | Configuration                           |
| ----------------- | --------------------------------------- |
| AWS Region        | `us-east-1`                             |
| Availability Zone | `us-east-1a`                            |
| VPC               | `kubeadm-vpc` (`10.0.0.0/16`)           |
| Public Subnet     | `kubeadm-public-subnet` (`10.0.1.0/24`) |
| Internet Gateway  | `kubeadm-igw`                           |
| Route Table       | `kubeadm-public-rt`                     |
| Security Group    | `kubeadm-sg`                            |
| Master            | `10.0.1.10`                             |
| Worker 1          | `10.0.1.11`                             |
| Worker 2          | `10.0.1.12`                             |
| Instance Type     | `t3.medium`                             |
| OS                | Amazon Linux 2023                       |
| SSH Key           | `eks.pem`                               |
| Pod CIDR          | `192.168.0.0/16`                        |

The full document includes AWS Console steps, commands for each node, verification checkpoints, application deployment exercises, troubleshooting and cleanup.

## Complete document – Ready

I've prepared the full 19-section, 7-page Kubernetes hands-on guide for Amazon Linux 2023 in the N. Virginia (`us-east-1`) region.

Kubernetes kubeadm – Amazon Linux 2023

Word document · 7 pages · AWS VPC, EC2, installation, deployment and troubleshooting

&#x20;Download Complete Word Document

The document includes step-by-step AWS Console configuration, SSH commands using `eks.pem`, all-node and master-only commands, worker joining, Nginx deployment, hands-on exercises, verification and AWS resource cleanup.



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
