## 🧩 Architecture Glossary: Core Kubernetes Concepts

| Component | Primary Focus | What It Manages | Real-World Example |
| :--- | :--- | :--- | :--- |
| **Controller** | Core control loop that continuously reconciles current state to desired state. | Native cluster objects (Pods, Services, Deployments). | `ReplicaSet Controller` |
| **Custom Resource Definition (CRD)** | Extension mechanism that registers custom API schemas with the Kubernetes API server. | Data structure definitions for non-native resources. | `kind: ExternalSecret` |
| **Custom Controller** | Dedicated reconciliation loop watching custom CRD instances. | Business logic for Custom Resources. | `Cert-Manager Controller` |
| **Operator** | Domain-specific Custom Controller bundling operational human knowledge. | Full lifecycle management of complex, stateful applications (upgrades, backups, failovers). | `External Secrets Operator`, `Prometheus Operator` |
| **Driver** | Standardized plugin translating K8s primitive requests to underlying infrastructure or hardware APIs. | Physical/cloud hardware resources (CSI for storage, CNI for networking). | `AWS EBS CSI Driver`, `Longhorn Storage Driver` |

---

### Detailed Breakdown

* **Controller:** The fundamental operational loop pattern in Kubernetes. It watches the cluster state through the API server and actively makes changes until the observed state matches the spec.
* **CRD (Custom Resource Definition):** Allows you to introduce custom objects into your cluster, making the Kubernetes API extensible without modifying core source code.
* **Custom Controller:** A controller written specifically to process custom resources defined by a CRD.
* **Operator:** An advanced Custom Controller that embeds application-specific domain expertise. Instead of just maintaining pod counts, it handles complex application lifecycles like cluster bootstrapping, automated backups, and version migrations.
* **Driver:** Interface implementation (such as CSI or CNI) that allows Kubernetes to manipulate low-level infrastructure directly—like mounting physical block storage or configuring host networking rules.