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
1. Enable MicroK8s addons
bash
microk8s status
microk8s enable dns hostpath-storage metrics-server rbac
2. Enable Traefik ingress

Deploy Traefik using this manifest: ic-traefik-hn_update1.yaml

bash
kubectl apply -f ic-traefik-hn_update1.yaml
3. Generate an SSL certificate (self-signed)
bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=frontend.192.168.111.129.nip.io" \
  -addext "subjectAltName=DNS:frontend.192.168.111.129.nip.io"
4. Create the TLS secret
bash
kubectl create secret tls frontend-tls-secret --cert=tls.crt --key=tls.key -n default
5. Open the firewall for the kubelet port, pod network and service network
bash
systemctl is-active firewalld
firewall-cmd --list-all
firewall-cmd --zone=trusted --list-sources

firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --zone=trusted --add-source=10.1.0.0/16
firewall-cmd --permanent --zone=trusted --add-source=10.152.183.0/24
firewall-cmd --reload
Range	Purpose
10250/tcp	kubelet API (used by metrics-server)
10.1.0.0/16	Pod network
10.152.183.0/24	Service (ClusterIP) network
Implementation and Testing
1. Apply the YAML files

Apply the manifests in this order, so dependencies exist before the resources that use them:

Secrets and ConfigMaps
RBAC (ServiceAccount, Role, RoleBinding)
PostgreSQL (StatefulSet and Service)
Backend (Deployment and Service)
Frontend (Deployment and Service)
Ingress and redirect middleware
HPA, PDB and NetworkPolicy
bash
kubectl apply -f <file>.yaml
2. Verify that the resources are created and running
bash
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

Check the NetworkPolicy in action:

bash
kubectl get networkpolicy
kubectl describe networkpolicy backend-db-only
kubectl exec deploy/backend -- python -c "import socket;socket.create_connection(('postgres',5432),3);print('backend -> postgres: OPEN')"
kubectl exec deploy/frontend -- nc -zv -w3 postgres 5432; echo "exit code: $?"

Check pod metrics:

bash
kubectl top pods
3. Tests
Autoscaling test: see the HPA in action

The load generators continuously hit the backend and increase CPU usage. When the CPU threshold defined in the HPA is breached, Kubernetes starts additional pods, up to the maximum replicas.

bash
for i in 1 2 3; do
  kubectl run load-$i --image=busybox --restart=Never -- /bin/sh -c \
    "while true; do wget -q -T 5 -O- http://backend:5000/cpu >/dev/null; done"
done

Watch the result:

bash
kubectl get hpa -w
kubectl get pods -l app=backend -w
kubectl top pods -l app=backend     # real CPU per pod
kubectl top nodes                   # make sure the node isn't maxed out
kubectl describe hpa backend-hpa    # the Events section explains each scaling decision

Stop the load:

bash
kubectl delete pod load-1 load-2 load-3

The HPA scales back down to the minimum after about 5 minutes.

Pod failure simulation: self-healing, PDB and affinity

Self-healing: delete the backend pods and watch them being recreated.

bash
kubectl delete pod -l app=backend --force
kubectl get pods     # shows the backend pods created again

A plain delete removes pods directly through the API, so it does not honor the PDB. A PDB only applies to evictions (kubectl drain, node upgrades, or the Eviction API).

bash
kubectl delete pod -l app=backend

To test the PDB, use the Eviction API. The first eviction succeeds and the second is rejected while the replacement pod is not yet Ready:

bash
for POD in $(kubectl get pod -l app=backend -o jsonpath='{.items[*].metadata.name}'); do
  echo "--- evicting $POD"
  kubectl create --raw /api/v1/namespaces/default/pods/$POD/eviction -f - <<EOF
{"apiVersion":"policy/v1","kind":"Eviction","metadata":{"name":"$POD","namespace":"default"}}
EOF
done

Check pod placement:

bash
kubectl get pods -l app=backend
kubectl get pods -o wide
Secret and ConfigMap: configuration changes without downtime
Open https://frontend.192.168.111.129.nip.io/ and note the env value.
Edit 6-app-config.yaml and change APP_ENV: production to APP_ENV: staging.
Apply and roll out:
bash
   kubectl apply -f 6-app-config.yaml
   kubectl rollout restart deployment backend
   kubectl rollout status deploy/backend
Reload the URL and confirm the env value changed.

Updating the database password. Change it inside PostgreSQL first, then update the Secret, then restart the backend:

bash
kubectl exec postgres-0 -- psql -U postgres -c "ALTER USER postgres PASSWORD 'newpass123';"
kubectl create secret generic db-secret --from-literal=password=newpass123 --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deploy/backend

Also update db-secret.yaml with the new value, otherwise the next kubectl apply restores the old password and the backend can no longer log in.

Persistent data test: data survives restarts
Open https://frontend.192.168.111.129.nip.io/ and note the visits value.
Reload several times and confirm the value keeps increasing, which shows the backend is writing to the database.
Query the database:
bash
   kubectl exec postgres-0 -- psql -U postgres -c "select count(*), max(ts) from visits"
Delete the database pod:
bash
   kubectl delete pod postgres-0
Once the pod is recreated, run the query again. The count must continue from where it was, which proves the persistent storage works:
bash
   kubectl exec postgres-0 -- psql -U postgres -c "select count(*), max(ts) from visits"
Network policy validation: zero-trust networking

Backend to database is allowed, and frontend to database is denied:

bash
kubectl exec deploy/backend -- python -c "import socket;socket.create_connection(('postgres',5432),3);print('ok')"
kubectl exec deploy/frontend -- nc -zv -w3 postgres 5432

Expected: the first command prints ok. The second times out.