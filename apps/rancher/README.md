# Rancher

Rancher adalah **Kubernetes management platform** untuk mengelola multi-cluster via UI.

## Deployment

### Prerequisites
- cert-manager
- ArgoCD
- Domain: k8s.sawangan.web.id

### Struktur GitOps
```
apps/rancher/
├── README.md            ← Dokumentasi ini
├── app.yaml             ← ArgoCD Application manifest
├── namespace.yaml       ← Namespace cattle-system
└── service-nodeport.yaml ← NodePort expose
```

### Cara Deploy
ArgoCD auto-sync dari repo ini. Atau manual:
```bash
kubectl apply -f apps/rancher/namespace.yaml
helm upgrade --install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=k8s.sawangan.web.id \
  --set bootstrapPassword=admin \
  --set replicas=1
```

## NodePort
| Port | Fungsi |
|------|--------|
| 8086 | HTTP → Rancher |
| 8446 | HTTPS → Rancher |

## Arsitektur
```
Internet → Caddy-WAF (443) → NodePort 8086 → Rancher Service → Rancher Pod
                                                                    │
                                                        ┌───────────┴──────────┐
                                                    Manage Cluster    Manage Apps
```

## Troubleshooting
| Problem | Solusi |
|---------|--------|
| Image pull lambat | `crictl pull rancher/rancher:v2.14.3` langsung di node |
| bootstrap-secret not found | `kubectl create secret generic bootstrap-secret -n cattle-system --from-literal=bootstrapPassword=admin` |
| kubeVersion incompatible | Hapus `kubeVersion:` dari Chart.yaml sebelum install |

## Caddy Config
```caddy
k8s.sawangan.web.id {
    reverse_proxy 100.106.55.29:8086
}
```
