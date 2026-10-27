End to End Flow
===================
User (browser)
 │  http://frontend.192.168.111.129.nip.io   (nip.io DNS resolves to the node IP)
 ▼
Node 192.168.111.129
 ├─ :80  ─► Traefik "web" entrypoint
 │            └─ Middleware frontend-redirect-to-https ─► 301 to https://
 └─ :443 ─► Traefik "websecure" entrypoint
              └─ TLS terminated (Secret: frontend-tls-secret)
 ▼
Ingress frontend-ingress-secure  (class: cluster-wide-ingress)
 ▼
Service frontend (ClusterIP :80)
 ▼
Frontend Pods x2  (Deployment, nginx:alpine)        ◄─ ConfigMap frontend-content
 │   /      → serves index.html                         (nginx config + page)
 │   /api/  → proxy_pass http://backend:5000/          PDB: frontend-pdb
 ▼
Service backend (ClusterIP :5000)
 ▼
Backend Pods x2-10  (Deployment, python:3.10-slim)  ◄─ ConfigMap backend-code (app.py)
 │                                                   ◄─ ConfigMap app-config (APP_ENV)
 │                                                   ◄─ Secret db-secret (DB_PASSWORD)
 │                                                      ServiceAccount: backend-sa
 │                                                      HPA: backend-hpa (70% CPU)
 │                                                      PDB: backend-pdb
 ▼
Service postgres (headless, :5432)
 │   NetworkPolicy backend-db-only: only app=backend may connect
 ▼
Pod postgres-0  (StatefulSet, postgres:15)          ◄─ Secret db-secret (POSTGRES_PASSWORD)
 ▼
PVC data-postgres-0 ─► PV (microk8s-hostpath, 5Gi)


Kubernetes features used
============================
**Deployment** | Frontend, backend | Self-healing, rolling updates, horizontal scaling

**StatefulSet** | PostgreSQL | Stable identity (postgres-0), persistent storage per pod

**Service (ClusterIP)** | Frontend, backend | Stable internal addresses and load balancing across pods

**Headless Service** | PostgreSQL | Direct, stable DNS name for the StatefulSet pod

**Ingress (Traefik)** | External access | Single entry point, host-based routing, TLS termination, HTTP-to-HTTPS redirect

**ConfigMap** | App config, nginx config, app code | Non-sensitive settings, changed without rebuilding images

**Secret** | Database password | Credentials kept out of images and manifests' pod specs

**PV / PVC** | PostgreSQL data | Data survives pod restarts

**HPA** | Backend | Scales 2 to 10 pods at 70% CPU (needs metrics-server)

**VPA** (optional) | Backend | Recommends right-sized CPU and memory requests

**RBAC + ServiceAccount** | Backend | Least-privilege identity for the backend pods

**PodDisruptionBudget** | Frontend, backend | Limits how many pods can be down during planned maintenance

**Affinity / anti-affinity** | Backend | Spreads pods across nodes (effective on multi-node clusters)

**NetworkPolicy** | PostgreSQL | Zero-trust: only backend pods can reach the database


1. Frontend (Presentation Layer)
Nginx

2. Backend API (Application Layer)
PIP

3. Database (Data Layer – Stateful)
PostgreSQL

4. Services (Internal Networking)
ClusterIP, Headless

5. Ingress (External Access)
http to https redirect, SSL Offloading

6. ConfigMaps and Secrets

7. HPA/VPA (Autoscaling)
VPA not implemented

8. RBAC (Security inside cluster)

9. PDB (POD disruption budget)

10. Network Policy (Zero Trust)

Prerequisites
================
Enable Microk8s addons:
microk8s status
microk8s enable dns hostpath-storage metrics-server rbac

Enable Traefik Ingress:
https://github.com/windevopscloud/docker/blob/main/k8s_ingress/ic-traefik-hn_update1.yaml

Generate SSL Certfificate:
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=frontend.192.168.111.129.nip.io" \
  -addext "subjectAltName=DNS:frontend.192.168.111.129.nip.io"

Create TLS secret:
kubectl create secret tls frontend-tls-secret --cert=tls.crt --key=tls.key -n default

Allow kubelet port, Service Network, and POD Network IP range:
systemctl is-active firewalld
firewall-cmd --list-all
firewall-cmd --zone=trusted --list-sources
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --zone=trusted --add-source=10.1.0.0/16
firewall-cmd --permanent --zone=trusted --add-source=10.152.183.0/24
firewall-cmd --reload

Implementation and Testing
=============================
