## End to End Flow

```text
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
Frontend Pods x2  (Deployment, nginx:alpine)     ◄─ ConfigMap frontend-content
 │   /      → serves index.html                     (nginx config + page)
 │   /api/  → proxy_pass http://backend:5000/       PDB: frontend-pdb
 ▼
Service backend (ClusterIP :5000)
 ▼
Backend Pods x2-10  (Deployment, python:3.10-slim) ◄─ ConfigMap backend-code (app.py)
 │                                                  ◄─ ConfigMap app-config (APP_ENV)
 │                                                  ◄─ Secret db-secret (DB_PASSWORD)
 │                                                     ServiceAccount: backend-sa
 │                                                     HPA: backend-hpa (70% CPU)
 │                                                     PDB: backend-pdb
 ▼
Service postgres (headless, :5432)
 │   NetworkPolicy backend-db-only: only app=backend may connect
 ▼
Pod postgres-0  (StatefulSet, postgres:15)       ◄─ Secret db-secret (POSTGRES_PASSWORD)
 ▼
PVC data-postgres-0 ─► PV (microk8s-hostpath, 5Gi)
```

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
```bash
microk8s status
microk8s enable dns hostpath-storage metrics-server rbac
```

Enable Traefik Ingress:
```bash
https://github.com/windevopscloud/docker/blob/main/k8s_ingress/ic-traefik-hn_update1.yaml
```

Generate SSL Certfificate:
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=frontend.192.168.111.129.nip.io" \
  -addext "subjectAltName=DNS:frontend.192.168.111.129.nip.io"
```

Create TLS secret:
```bash
kubectl create secret tls frontend-tls-secret --cert=tls.crt --key=tls.key -n default
```

Allow kubelet port, Service Network, and POD Network IP range:
```bash
systemctl is-active firewalld
firewall-cmd --list-all
firewall-cmd --zone=trusted --list-sources
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --zone=trusted --add-source=10.1.0.0/16
firewall-cmd --permanent --zone=trusted --add-source=10.152.183.0/24
firewall-cmd --reload
```

Implementation and Testing
=============================
1. Apply the yaml files to create the resources
   ```bash
   kubectl apply -f *.yaml
   ```

2. Check and ensure the resources are created and running successfuly 
   ```bash
   kubectl version
   kubectl get nodes
   kubectl get secrets
   kubectl get pods
   kubectl get pvc
   kubectl describe pod <backend-pod-name>
   kubectl get svc frontend
   kubectl get ingress
   kubectl get hpa
   kubectl get pdb
   ```

   Check NetworkPolicy in action:
   ```bash
   kubectl get networkpolicy
   kubectl describe networkpolicy backend-db-only
   kubectl exec deploy/backend -- python -c "import socket;socket.create_connection(('postgres',5432),3);print('backend -> postgres: OPEN')"
   kubectl exec deploy/frontend -- nc -zv -w3 postgres 5432; echo "exit code: $?"
   ```

   Check POD metrics:
   ```bash
   kubectl top pods
   ```

   URLS:
   ```bash
   https://frontend.192.168.111.129.nip.io
   https://frontend.192.168.111.129.nip.io/api/health
   https://frontend.192.168.111.129.nip.io/api/visits
   https://frontend.192.168.111.129.nip.io/api/cpu
   ```

3. Tests

   *** Autoscaling Test:** See HPA in Action: Load generator will continuously hits the backend service, increasing CPU usage. When the CPU threshold defined in the hpa is breached, it will start additional PODS (upto max replica) as defined in the hpa.
   ```bash
   for i in 1 2 3; do
     kubectl run load-$i --image=busybox --restart=Never -- /bin/sh -c \
     "while true; do wget -q -T 5 -O- http://backend:5000/cpu >/dev/null; done"
   done
   
   kubectl get hpa -w
   kubectl get pods -l app=backend -w
   kubectl top pods -l app=backend     # real CPU per pod
   kubectl top nodes                   # make sure the node isn't maxed out
   kubectl describe hpa backend-hpa    # the Events section explains each scaling decision

   kubectl delete pod load-1 load-2 load-3
   ```

   *** Pod Failure Simulation:** Test Self-Healing, PDB & Affinity
   ```bash
   kubectl delete pod -l app=backend --force
   kubectl get pods     # This will show backend pod created again

   kubectl delete pod -l app=backend     # Since delete removes pods directly through the API, it doesnt honor PDB. A PDB only applies to evictions (kubectl drain, node upgrades, or the Eviction API)

   for POD in $(kubectl get pod -l app=backend -o jsonpath='{.items[*].metadata.name}'); do
    echo "--- evicting $POD"
    kubectl create --raw /api/v1/namespaces/default/pods/$POD/eviction -f - <<EOF
    {"apiVersion":"policy/v1","kind":"Eviction","metadata":{"name":"$POD","namespace":"default"}}
    EOF
   done

   kubectl get pods -l app=backend
   kubectl get pods -o wide
   ```

   *** Secret & ConfigMap:** Test configuration change rollouts without impacting availabiltiy
   ```bash
   Access the URL and note env value: https://frontend.192.168.111.129.nip.io/
   Edit 6-app-config.yaml; APP_ENV: production to APP_ENV: staging
   kubectl apply -f 6-app-config.yaml
   kubectl rollout restart deployment backend
   kubectl rollout status deploy/backend
   Access the URL and note env value change: https://frontend.192.168.111.129.nip.io/
   
   Updating db password
   kubectl exec postgres-0 -- psql -U postgres -c "ALTER USER postgres PASSWORD 'newpass123';"
   kubectl create secret generic db-secret --from-literal=password=newpass123 --dry-run=client -o yaml | kubectl apply -f -
   kubectl rollout restart deploy/backend
   ```

   *** Persistent Data Test:** Prove Data Survives Restarts
   ```bash
   Access the URL and note visits value: https://frontend.192.168.111.129.nip.io/
   Access the URL multiple time and note the value incrementing, backend the db is getting updated.
   kubectl exec postgres-0 -- psql -U postgres -c "select count(*), max(ts) from visits"
   kubectl delete pod postgres-0
   Rerun the query again once the database pod is recreated to ensure the persistent storage works.
   kubectl exec postgres-0 -- psql -U postgres -c "select count(*), max(ts) from visits"
   ```

   *** Network Policy Validation:** Zero-Trust Networking
   ```bash
   Test Allows Traffic (Backend → Database), and Denies Traffic (Frontend → Database)
   kubectl exec deploy/backend -- python -c "import socket;socket.create_connection(('postgres',5432),3);print('ok')"
   kubectl exec deploy/frontend -- nc -zv -w3 postgres 5432
   ```