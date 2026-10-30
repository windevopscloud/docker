Setup up K8s cluster from scratch with 3 Nodes (1 Master and 2 Worker) using kubeadm.
VM Configuration:
    2vCPU
    4gb Memory
    40gb Diskspace
    RHEL10

After initial setup, follwing addons/operators are installed and configured for their respective purpose.
    Container Network Interface(CNI) - Calico
    Load Balancer - MetalLB (Using BGP Mode, rather than its default L2 Mode)
    Ingress Controller - Traefik
    Container Storage Interface(CSI) - Longhorn
    Observability - Metrics Server, Prometheus and Grafana
    K8s Dashboard - Headlamp
    CI/CD - ArgoCD
    Secrets Manager - 
    Cert Manager -
    Security - RBAC, NetworkPolicies, Encryption@Rest, Audit Logging, POD Security, etc.
    Workload Hygiene - Resource requests/limits, liveness/readiness probes, Pod Distribution Budget, Autoscaling, Node Selector/Affinity, etc. 

Issues faced while performing the setup and its fixes are documented in issues_fixes.md







