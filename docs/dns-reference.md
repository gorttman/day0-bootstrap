# DNS Reference — i3sec.com.au

## Service Records

| Hostname | IP | Namespace | Backend Service |
|---|---|---|---|
| argocd.i3sec.com.au | 192.168.1.11 | argocd | argocd-server:80 |
| pihole.i3sec.com.au | 192.168.1.11 | pihole | pihole-web:8080 |
| novnc.i3sec.com.au | 192.168.1.11 | remote-access | novnc:80 |

All traffic routes via **Traefik** (k3s ServiceLB, external IP: `192.168.1.11`).

---

## GitOps DNS Flow

```
gorttman/dns-conf (separate config repo)
  └── apps/pihole-custom-dns-cm.yml   ← edit this to add/remove DNS entries
        ↓ ArgoCD (dns-config Application)
  pihole namespace: ConfigMap pihole-custom-dns
        ↓ mounted into Pi-hole pod
  /etc/dnsmasq.d/02-i3sec-custom-dns.conf
```

---

## 1. dns-conf repo setup (gorttman/dns-conf)

Create these two files in the repo:

### apps/kustomization.yml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - pihole-custom-dns-cm.yml
```

### apps/pihole-custom-dns-cm.yml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pihole-custom-dns
  namespace: pihole
  labels:
    app: pihole
    app.kubernetes.io/component: dns
    app.kubernetes.io/name: pihole
    app.kubernetes.io/part-of: dns-config
data:
  02-i3sec-custom-dns.conf: |
    address=/argocd.i3sec.com.au/192.168.1.11
    address=/pihole.i3sec.com.au/192.168.1.11
    address=/novnc.i3sec.com.au/192.168.1.11
```

To add a new DNS entry in future: edit `pihole-custom-dns-cm.yml`, commit, push. ArgoCD syncs within minutes, Pi-hole picks up the new file on next restart (or `pihole restartdns` inside the pod for immediate effect without restart).

---

## 2. Pi-hole Deployment change (day2-services repo)

In the pihole Deployment manifest, add the following.

**Under `spec.template.spec.volumes` (alongside the existing pihole-data PVC):**
```yaml
- name: pihole-custom-dns
  configMap:
    name: pihole-custom-dns
```

**Under `spec.template.spec.containers[0].volumeMounts` (alongside the existing PVC mounts):**
```yaml
- mountPath: /etc/dnsmasq.d/02-i3sec-custom-dns.conf
  name: pihole-custom-dns
  subPath: 02-i3sec-custom-dns.conf
```

This mounts only the single file into `/etc/dnsmasq.d/` alongside Pi-hole's own generated configs on the PVC. The PVC mount is unchanged.

---

## 3. ArgoCD insecure mode (one-time, after merging branch to main)

```bash
kubectl patch cm argocd-cmd-params-cm -n argocd --type merge \
  -p '{"data":{"server.insecure":"true"}}'
kubectl rollout restart deployment argocd-server -n argocd
```

---

## 4. Pi-hole upstream DNS

Already configured: `PIHOLE_DNS_: 1.1.1.1;1.0.0.1` in the `pihole-config` ConfigMap.
