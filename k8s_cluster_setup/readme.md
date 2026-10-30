# Kubernetes Cluster Setup Guide

Setup a K8s cluster from scratch with 3 Nodes (1 Master and 2 Worker) using `kubeadm`.

### VM Configuration
*   **vCPU:** 2
*   **Memory:** 4 GB
*   **Diskspace:** 40 GB
*   **OS:** RHEL 10

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
sudo hostnamectl set-hostname k8s-master        #Master Node
sudo hostnamectl set-hostname k8s-worker1       #Worker Node1
sudo hostnamectl set-hostname k8s-worker2       #Worker Node2
```
Map the static IPs of nodes inside /etc/hosts on all Nodes (Master and Both Workers)
```bash
sudo tee -a /etc/hosts <<EOF
192.168.111.130 k8s-master
192.168.111.131 k8s-worker1
192.168.111.132 k8s-worker2
EOF
```

### Troubleshooting
Issues faced while performing the setup and their respective fixes are documented in [`issues_fixes.md`](issues_fixes.md).
