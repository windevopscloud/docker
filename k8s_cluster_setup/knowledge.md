# Kubernetes Objects: Controller, Custom Resource Definition, Custom Controller, Operator, Driver
| Component | Primary Focus | What It Manages | Real-World Example |
| :--- | :--- | :--- | :--- |
| **Controller** | Core control loop that continuously reconciles current state to desired state. | Native cluster objects (Pods, Services, Deployments). | `ReplicaSet Controller` |
| **Custom Resource Definition (CRD)** | Extension mechanism that registers custom API schemas with the Kubernetes API server. | Data structure definitions for non-native resources. | `kind: ExternalSecret` |
| **Custom Controller** | Dedicated reconciliation loop watching custom CRD instances. | Business logic for Custom Resources. | `Cert-Manager Controller` |
| **Operator** | Domain-specific Custom Controller bundling operational human knowledge. | Full lifecycle management of complex, stateful applications (upgrades, backups, failovers). | `External Secrets Operator`, `Prometheus Operator` |
| **Driver** | Standardized plugin translating K8s primitive requests to underlying infrastructure or hardware APIs. | Physical/cloud hardware resources (CSI for storage, CNI for networking). | `AWS EBS CSI Driver`, `Longhorn Storage Driver` |

* **Controller:** The fundamental operational loop pattern in Kubernetes. It watches the cluster state through the API server and actively makes changes until the observed state matches the spec.
* **CRD (Custom Resource Definition):** Allows you to introduce custom objects into your cluster, making the Kubernetes API extensible without modifying core source code.
* **Custom Controller:** A controller written specifically to process custom resources defined by a CRD.
* **Operator:** An advanced Custom Controller that embeds application-specific domain expertise. Instead of just maintaining pod counts, it handles complex application lifecycles like cluster bootstrapping, automated backups, and version migrations.
* **Driver:** Interface implementation (such as CSI or CNI) that allows Kubernetes to manipulate low-level infrastructure directly—like mounting physical block storage or configuring host networking rules.

# Kubernetes Scheduling: Taints, Tolerations, & Node Affinity, Anti Affinity
Reference guide for controlling where pods land across master and worker nodes using taints, tolerations, and node affinity rules.

| Mechanism | Direction | Strength & Behavior |
| :--- | :--- | :--- |
| **Taint** | Repels pods away from a node | Pod requires a matching `toleration` |
| **nodeSelector** | Pod chooses nodes by label | Hard, exact-match only |
| **nodeAffinity** | Pod chooses nodes by label | Hard or soft rules with richer operators |

## 🛑 1. Taints and Tolerations
Taints allow a node to **repel** a set of pods unless those pods explicitly have a matching toleration.

### Key Commands
```bash
# Check existing taints on the control plane node
kubectl describe node k8s-master | grep -i taint

# Add control-plane taint (default behavior in kubeadm)
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane=:NoSchedule

# Remove a taint (note the trailing hyphen)
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule-

# Apply a custom taint to a worker node (e.g., reserve for databases)
kubectl taint nodes k8s-worker1 dedicated=database:NoSchedule

# Apply a maintenance taint with key-only and NoExecute effect
kubectl taint nodes k8s-worker2 maintenance:NoExecute
```
<pre>
Taint Effects Explained
NoSchedule: New pods without a matching toleration cannot be scheduled on the node. Existing running pods remain unaffected.

PreferNoSchedule: The scheduler tries to avoid the node, but will use it as a fallback if no other option exists.

NoExecute: New pods are blocked, and any existing running pods without a matching toleration are immediately evicted.
</pre>

<pre>
Basic Toleration (Equal)
YAML
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"

Control-Plane Toleration (Exists)
YAML
spec:
  tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"

Complete Pod Manifest Targeting Control-Plane
YAML
apiVersion: v1
kind: Pod
metadata:
  name: nginx-master
spec:
  nodeSelector:
    node-role.kubernetes.io/control-plane: ""
  tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx
</pre>

## 🧲 2. Node Affinity
Node affinity functions like a more expressive, rule-based version of nodeSelector.
<pre>
Supported Types & Operators
requiredDuringSchedulingIgnoredDuringExecution: Hard rule. If no matching node is found, the pod remains Pending. Supports operators: In, NotIn, Exists, DoesNotExist, Gt, Lt.

preferredDuringSchedulingIgnoredDuringExecution: Soft rule. Uses weights (1-100) to prioritize matching nodes, but schedules elsewhere if unavailable.

Hard Rule Example (Must run on Control-Plane)
YAML
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-role.kubernetes.io/control-plane
            operator: Exists
  tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule

Negative Constraint Example (Avoid Control-Plane)
YAML
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-role.kubernetes.io/control-plane
            operator: DoesNotExist

Soft Rule / Weighted Preference Example
kubectl label node k8s-worker1 disktype=ssd

YAML
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd"]

YAML
spec:
  #nodeSelector forces the pod to land *only* on nodes matching these exact labels
  nodeSelector:
    disktype: ssd 
  containers:
  - name: nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80

Deployment with Pod Anti-Affinity
YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-app-deployment
  labels:
    app: secure-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-web
  template:
    metadata:
      labels:
        app: secure-web
    spec:
      affinity:
        podAntiAffinity:
          # Hard rule: Kubernetes will refuse to schedule pods on the same node
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - secure-web
            topologyKey: "kubernetes.io/hostname"
            #topologyKey: "kubernetes.io/hostname": This tells Kubernetes what boundary defines a "node". Every standard Kubernetes node has the built-in label kubernetes.io/hostname set to its unique node name. This ensures the anti-affinity check checks at the individual node level.
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
</pre>
