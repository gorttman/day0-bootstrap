# DNS Reference — i3sec.com.au

## Service Records

| Hostname | IP | Namespace | Backend Service |
|---|---|---|---|
| argocd.i3sec.com.au | 192.168.1.11 | argocd | argocd-server:80 |
| pihole.i3sec.com.au | 192.168.1.11 | pihole | pihole-web:8080 |
| novnc.i3sec.com.au | 192.168.1.11 | remote-access | novnc:80 |

All traffic routes via **Traefik** (k3s ServiceLB, external IP: `192.168.1.11`).

---

## 1. Pi-hole Custom DNS

Drop this file into the Pi-hole pod:

```
# /etc/dnsmasq.d/02-i3sec-custom-dns.conf
address=/argocd.i3sec.com.au/192.168.1.11
address=/pihole.i3sec.com.au/192.168.1.11
address=/novnc.i3sec.com.au/192.168.1.11
```

Apply with:

```bash
POD=$(kubectl get pod -n pihole -l app=pihole -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it "$POD" -n pihole -- bash -c 'cat > /etc/dnsmasq.d/02-i3sec-custom-dns.conf << EOF
address=/argocd.i3sec.com.au/192.168.1.11
address=/pihole.i3sec.com.au/192.168.1.11
address=/novnc.i3sec.com.au/192.168.1.11
EOF
pihole restartdns'
```

Upstream DNS: configure via Pi-hole web UI → Settings → DNS → `1.1.1.1` (primary), `8.8.8.8` (secondary).

---

## 2. ArgoCD — server.insecure patch

Apply once (safe merge, does not overwrite other keys):

```bash
kubectl patch cm argocd-cmd-params-cm -n argocd --type merge \
  -p '{"data":{"server.insecure":"true"}}'
kubectl rollout restart deployment argocd-server -n argocd
```

`argocd-cmd-params-cm.yml` in this repo will keep it in sync via GitOps after merge.

---

## 3. Pi-hole Ingress — update existing manifest

In the pihole app repo, change the existing ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pihole
  namespace: pihole
spec:
  ingressClassName: traefik
  rules:
    - host: pihole.i3sec.com.au
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: pihole-web
                port:
                  number: 8080
```

Or apply immediately:

```bash
kubectl apply -f - << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pihole
  namespace: pihole
spec:
  ingressClassName: traefik
  rules:
    - host: pihole.i3sec.com.au
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: pihole-web
                port:
                  number: 8080
EOF
```

---

## 4. noVNC Ingress — update existing manifest

In the novnc app repo, change the existing ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: novnc
  namespace: remote-access
spec:
  ingressClassName: traefik
  rules:
    - host: novnc.i3sec.com.au
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: novnc
                port:
                  number: 80
```

Or apply immediately:

```bash
kubectl apply -f - << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: novnc
  namespace: remote-access
spec:
  ingressClassName: traefik
  rules:
    - host: novnc.i3sec.com.au
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: novnc
                port:
                  number: 80
EOF
```
