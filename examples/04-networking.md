# Kubernetes Networking with kubeadm

Complete guide for configuring and troubleshooting Kubernetes networking.

## 🎯 Objectives

- Understand Kubernetes networking concepts
- Configure different CNI plugins
- Test pod-to-pod and pod-to-service communication
- Troubleshoot common networking issues

## 🌐 Networking Concepts

### Kubernetes Network Model

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│  ┌─────────────────┐       ┌─────────────────┐             │
│  │   Node 1        │       │   Node 2        │             │
│  │                 │       │                 │             │
│  │  ┌───────────┐  │       │  ┌───────────┐  │             │
│  │  │   Pod A   │  │       │  │   Pod C   │  │             │
│  │  │ 10.244.1.2│◄─┼───────┼─►│ 10.244.2.3│  │             │
│  │  └───────────┘  │       │  └───────────┘  │             │
│  │                 │       │                 │             │
│  │  ┌───────────┐  │       │  ┌───────────┐  │             │
│  │  │   Pod B   │  │       │  │   Pod D   │  │             │
│  │  │ 10.244.1.3│  │       │  │ 10.244.2.4│  │             │
│  │  └───────────┘  │       │  └───────────┘  │             │
│  │                 │       │                 │             │
│  │  CNI: 10.244.1.0/24     │  CNI: 10.244.2.0/24          │
│  └─────────────────┘       └─────────────────┘             │
│                                                             │
│  Cluster IP Range: 10.96.0.0/12                           │
│  Pod Network CIDR: 10.244.0.0/16                          │
└─────────────────────────────────────────────────────────────┘
```

### Network Components

1. **Pod Network**: Each pod gets a unique IP
2. **Service Network**: Virtual IPs for service discovery
3. **Node Network**: Physical/VM network for nodes
4. **CNI Plugin**: Implements pod networking

## 🔧 CNI Plugin Configurations

### 1. Flannel (Simple Overlay)

#### Installation
```bash
# Initialize cluster with Flannel CIDR
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Install Flannel
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

#### Custom Configuration
```yaml
# flannel-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

### 2. Calico (Advanced Features)

#### Installation
```bash
# Initialize cluster (any pod CIDR works with Calico)
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# Install Calico operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

# Configure Calico
cat <<EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
EOF
```

#### Network Policies with Calico
```yaml
# deny-all-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# allow-nginx-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: client
    ports:
    - protocol: TCP
      port: 80
```

### 3. Weave Net (Easy Setup)

#### Installation
```bash
# Initialize cluster
sudo kubeadm init --pod-network-cidr=10.32.0.0/12

# Install Weave Net
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

#### Weave Net with Encryption
```bash
# Install with password encryption
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&password=my-secret-password"
```

## 🧪 Testing Network Connectivity

### Basic Connectivity Tests

```bash
# Create test pods
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-1
  labels:
    app: test-pod-1
spec:
  containers:
  - name: test-container
    image: busybox
    command: ['sh', '-c', 'sleep 3600']
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-2
  labels:
    app: test-pod-2
spec:
  containers:
  - name: test-container
    image: busybox
    command: ['sh', '-c', 'sleep 3600']
EOF

# Get pod IPs
kubectl get pods -o wide

# Test pod-to-pod connectivity
POD1_IP=$(kubectl get pod test-pod-1 -o jsonpath='{.status.podIP}')
POD2_IP=$(kubectl get pod test-pod-2 -o jsonpath='{.status.podIP}')

kubectl exec test-pod-1 -- ping -c 3 $POD2_IP
kubectl exec test-pod-2 -- ping -c 3 $POD1_IP
```

### Service Discovery Tests

```bash
# Create a service
kubectl expose pod test-pod-1 --port=80 --target-port=80 --name=test-service

# Test service resolution
kubectl exec test-pod-2 -- nslookup test-service
kubectl exec test-pod-2 -- nslookup test-service.default.svc.cluster.local

# Test service connectivity
kubectl exec test-pod-2 -- wget -qO- http://test-service
```

### DNS Resolution Tests

```bash
# Create DNS test pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dns-test
spec:
  containers:
  - name: dns-test
    image: busybox
    command: ['sh', '-c', 'sleep 3600']
EOF

# Test DNS resolution
kubectl exec dns-test -- nslookup kubernetes.default
kubectl exec dns-test -- nslookup kube-dns.kube-system
kubectl exec dns-test -- cat /etc/resolv.conf
```

## 🔍 Network Troubleshooting

### Check CNI Status

```bash
# Check CNI pods
kubectl get pods -n kube-system | grep -E '(flannel|calico|weave)'

# Check CNI DaemonSet
kubectl get daemonset -n kube-system

# Check CNI logs
kubectl logs -n kube-system daemonset/kube-flannel-ds-amd64

# Check node network configuration
kubectl describe node <node-name> | grep -A 10 "Network"
```

### Verify Network Configuration

```bash
# Check pod networking
ip route show
ip addr show

# Check iptables rules
sudo iptables -t nat -L
sudo iptables -L

# Check bridge configuration
brctl show
```

### Common Network Issues

#### 1. Pods Cannot Communicate

```bash
# Check CNI installation
kubectl get pods -n kube-system

# Verify pod CIDR configuration
kubectl cluster-info dump | grep -i cidr

# Check if nodes have different subnets
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'

# Restart CNI pods
kubectl delete pods -n kube-system -l app=flannel
```

#### 2. DNS Not Working

```bash
# Check CoreDNS status
kubectl get pods -n kube-system | grep coredns

# Check CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml

# Test DNS manually
kubectl run dns-debug --image=busybox --rm -it -- sh
# Inside the pod:
nslookup kubernetes.default
cat /etc/resolv.conf
```

#### 3. Services Not Accessible

```bash
# Check service endpoints
kubectl get endpoints

# Check kube-proxy status
kubectl get pods -n kube-system | grep kube-proxy

# Check iptables rules for services
sudo iptables -t nat -L | grep <service-name>

# Restart kube-proxy
kubectl delete pods -n kube-system -l k8s-app=kube-proxy
```

## 🛡 Network Security

### Network Policies

```bash
# Create namespace for testing
kubectl create namespace secure-app

# Deploy test applications
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: secure-app
  labels:
    app: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: secure-app
  labels:
    app: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: secure-app
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
EOF

# Apply network policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: secure-app
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
EOF

# Test connectivity (should work)
kubectl exec -n secure-app deployment/frontend -- curl backend-service

# Test from different namespace (should fail if policy is working)
kubectl run test-pod --image=busybox --rm -it -- sh
# Inside pod: wget -qO- http://backend-service.secure-app
```

## 🌐 Advanced Networking

### Multi-Network Setup

```bash
# Install Multus CNI (for multiple network interfaces)
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml

# Create additional network
cat <<EOF | kubectl apply -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
spec:
  config: '{
    "cniVersion": "0.3.0",
    "type": "macvlan",
    "master": "eth0",
    "mode": "bridge",
    "ipam": {
      "type": "host-local",
      "subnet": "192.168.1.0/24",
      "rangeStart": "192.168.1.200",
      "rangeEnd": "192.168.1.250"
    }
  }'
EOF
```

### Load Balancer Integration

```bash
# Install MetalLB for bare metal load balancing
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml

# Configure IP pool
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2-adv
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
EOF

# Test LoadBalancer service
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
EOF
```

## 📊 Network Monitoring

### Network Performance Testing

```bash
# Deploy iperf3 for network testing
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iperf3-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: iperf3-server
  template:
    metadata:
      labels:
        app: iperf3-server
    spec:
      containers:
      - name: iperf3
        image: networkstatic/iperf3
        command: ['iperf3', '-s']
        ports:
        - containerPort: 5201
---
apiVersion: v1
kind: Service
metadata:
  name: iperf3-server
spec:
  selector:
    app: iperf3-server
  ports:
  - port: 5201
    targetPort: 5201
EOF

# Run client test
kubectl run iperf3-client --image=networkstatic/iperf3 --rm -it -- iperf3 -c iperf3-server -t 30
```

### Network Debugging Tools

```bash
# Deploy network debugging pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: netshoot
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command: ['sh', '-c', 'sleep 3600']
EOF

# Use debugging tools
kubectl exec -it netshoot -- bash
# Inside container:
# - nslookup kubernetes.default
# - traceroute 8.8.8.8
# - tcpdump -i eth0
# - netstat -rn
# - ss -tuln
```

## 📚 Practice Exercises

1. **CNI Comparison**: Set up clusters with different CNI plugins
2. **Network Policies**: Implement micro-segmentation
3. **Service Mesh**: Deploy Istio or Linkerd
4. **Load Balancing**: Configure different load balancer types
5. **Network Troubleshooting**: Simulate and fix network issues
6. **Performance Testing**: Measure network throughput
7. **Multi-tenancy**: Implement network isolation
8. **Security**: Configure network encryption

## 🔧 Troubleshooting Scripts

### Network Connectivity Check Script

```bash
#!/bin/bash
# Save as: check-network.sh

echo "=== Kubernetes Network Connectivity Check ==="

# Check nodes
echo "1. Checking nodes:"
kubectl get nodes -o wide

# Check CNI pods
echo -e "\n2. Checking CNI pods:"
kubectl get pods -n kube-system | grep -E '(flannel|calico|weave)'

# Check kube-proxy
echo -e "\n3. Checking kube-proxy:"
kubectl get pods -n kube-system | grep kube-proxy

# Check CoreDNS
echo -e "\n4. Checking CoreDNS:"
kubectl get pods -n kube-system | grep coredns

# Test DNS resolution
echo -e "\n5. Testing DNS:"
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default

echo -e "\nNetwork check complete!"
```

## 🎯 Next Steps

After mastering Kubernetes networking:
1. Service mesh implementation
2. Advanced network policies
3. Multi-cluster networking
4. Network observability
5. Performance optimization
6. Security hardening

## 📖 Additional Resources

- [Kubernetes Networking Guide](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [CNI Specification](https://github.com/containernetworking/cni)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Troubleshooting Guide](../troubleshooting/networking-issues.md)