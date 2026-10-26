1. Implementing ingress controller using traefik.
kubectl apply -f ic-traefik-hn_update1.yaml

This resource is configured to use hostNetwork, port 80 and 443 is bind using hostPort.
Entrypoints - http:web, and https:websecure

2. Create ssl key and certificate, update the CN and SAN as per the requirement.
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=192.168.111.129/O=windevopscloud" -addext "subjectAltName = IP:192.168.111.129,IP:192.168.111.240,DNS:cheddar.192.168.111.129.nip.io,DNS:service1.192.168.111.129.nip.io"

3. Create secrets. Since we are trying with two applications, separate secrets are created.
kubectl create secret tls service1-tls-secret --cert=tls.crt --key=tls.key
kubectl create secret tls cheddar-tls-secret --cert=tls.crt --key=tls.key

4. Deploy service1 and make use of traefik ingressroute to route requests, ssl offloading and http to https redirection.
kubectl apply -f service1-ingressroute.yaml

Following resources are created: Deployment, Service, Middleware, Ingressroute

5. Once testing and resource review is completed, delete the resources.
kubectl delete -f service1-ingressroute.yaml

6. Deploy service1 and make use of native Ingress resource and traefik annotations to route requests, ssl offloading and http to https redirection.
kubectl apply -f service1-annotations.yaml

7. Once testing and resource review is completed, delete the resources.
kubectl delete -f service1-annotations.yaml