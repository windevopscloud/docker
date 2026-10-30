# Kubernetes Cluster Setup Guide

Setup a K8s cluster from scratch with 3 Nodes (1 Master and 2 Worker) using `kubeadm`.

### VM Configuration
*   **vCPU:** 2
*   **Memory:** 4 GB
*   **Diskspace:** 40 GB
*   **OS:** RHEL 10
*   **Master NodeIP:** 192.168.111.130
*   **Worker NodeIP:** 192.168.111.131 and 192.168.111.132
*   **Load Balancer IP_Range:** 192.168.111.200-192.168.111.220

### Post-Setup Addons & Operators
After the initial setup, the following addons and operators are installed and configured for their respective purposes:

*   **Container Network Interface (CNI):** Calico
*   **Load Balancer:** MetalLB (Configured in BGP Mode instead of the default L2 Mode)
*   **Ingress Controller:** Traefik
*   **Container Storage Interface (CSI):** Longhorn
*   **Observability:** Metrics Server, Prometheus, and Grafana
*   **K8s Dashboard:** Headlamp
*   **CI/CD:** ArgoCD
*   **Secrets Manager:** *[Pending / To be added]*
*   **Cert Manager:** *[Pending / To be added]*
*   **Security:** RBAC, NetworkPolicies, Encryption@Rest, Audit Logging, Pod Security, etc.
*   **Workload Hygiene:** Resource requests/limits, liveness/readiness probes, Pod Disruption Budgets, Autoscaling, Node Selector/Affinity, etc.

---

### Installing k8's
1. Set Hostnames & Map IPs
```bash
#Master Node
sudo hostnamectl set-hostname k8s-master

#Worker Node1
sudo hostnamectl set-hostname k8s-worker1

#Worker Node2
sudo hostnamectl set-hostname k8s-worker2       
```
Map the static IPs of nodes inside /etc/hosts on all Nodes (Master and Both Workers)
```bash
sudo tee -a /etc/hosts <<EOF
192.168.111.130 k8s-master
192.168.111.131 k8s-worker1
192.168.111.132 k8s-worker2
EOF
```

2. Disable Swap & Configure SELinux
Kubernetes requires swap memory to be disabled entirely for proper performance scheduling
```bash
#Disable swap instantly
sudo swapoff -a

#Persist the change across reboots
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

#SELinux in permissive mode
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/config/selinux
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

#Should print permissive
getenforce
```

3. Enable Kernel Modules and Sysctl Networking Bridge
Allow Linux bridging tables to inspect cluster pod traffic
```bash
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl parameters
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply system changes without restarting
sudo sysctl --system
```

4. Configure Firewalld Ports
If firewalld is active, expose the mandatory system networking hooks
```bash
#On Master Node
sudo firewall-cmd --permanent --add-port={6443,2379-2380,10250,10251,10252}/tcp
sudo firewall-cmd --reload

#On Worker Nodes
sudo firewall-cmd --permanent --add-port={10250,30000-32767}/tcp
sudo firewall-cmd --reload

#For Calico (CNI) on All Nodes
sudo firewall-cmd --permanent --add-port=179/tcp --add-port=5473/tcp --add-port=4789/udp
sudo firewall-cmd --reload
```

5. Install containerd on All Nodes
```bash
# Provide Docker CE repo to pull containerd package binary
sudo rm -f /etc/yum.repos.d/docker*.repo
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf clean all

# Install containerd package
sudo dnf install -y containerd.io

# Generate default configuration directory layout
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# Critical: Configure containerd to utilize Systemd Cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Start and persist containerd service engine
sudo systemctl restart containerd
sudo systemctl enable containerd
```

6. Install Kubernetes Tools All Node: kubeadm, kubelet, kubectl
```bash
sudo tee /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
EOF

# Synchronize repo and install packages
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

# Enable the orchestration agent on engine boot
sudo systemctl enable --now kubelet
```

7.Initialize Cluster: Bootstrap the Control Plane
```bash
systemctl enable kubelet.service
sudo kubeadm init --apiserver-advertise-address=192.168.111.130 --pod-network-cidr=10.244.0.0/16
kubectl get nodes
```
<pre>
Your Kubernetes control-plane has initialized successfully!
To start using your cluster, you need to run the following as a regular user:
  mkdir -p \$HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf \$HOME/.kube/config
  sudo chown \$(id -u):(id -g) HOME/.kube/config

Nodes show NotReady until the CNI is installed.
[root@localhost ~]# kubectl get nodes
NAME         STATUS     ROLES           AGE     VERSION
k8s-master   NotReady   control-plane   9m12s   v1.30.14
</pre>

---

### Installing Calico (CNI)
1. Deploy a pod network to the cluster.
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/tigera-operator.yaml
kubectl get crd | grep -i projectcalico
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/custom-resources.yaml

#Update the pod cidr value in custom-resources.yaml to (10.244.0.0/16)
vi custom-resources.yaml
```
```bash
kubectl create -f custom-resources.yaml

#Wait till the pods are in running state
kubectl get pods -n calico-system -w
```
```bash
#Create Token to add nodes to k8s cluster
kubeadm token create --print-join-command

#Then join any number of worker nodes by running the join command (generated) on each worker node as root
```
```bash
kubectl get nodes
```
<pre>
Your Kubernetes control-plane has initialized successfully!
To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
</pre>








### Troubleshooting
Issues faced while performing the setup and their respective fixes are documented in [`issues_fixes.md`](issues_fixes.md).
