# Longhorn Storage Setup on Raspberry Pi K3s Cluster

## Prerequisites Installation
```bash
sudo apt update
sudo apt install -y open-iscsi nfs-common
sudo systemctl enable --now iscsid
```

## Helm Installation
```bash
sudo helm repo add longhorn https://charts.longhorn.io
sudo helm repo update
sudo helm install longhorn longhorn/longhorn --namespace longhorn-system \
  --set defaultSettings.defaultDataPath="/mnt/k3s-storage/longhorn"
```

## UI Access Configuration
Created ingress for web interface:
```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ingress
  namespace: longhorn-system
spec:
  rules:
  - host: longhorn.raspb1s.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: longhorn-frontend
            port:
              number: 80
EOF
```

## Local Access
Add to `/etc/hosts`:
```
<Pi_IP> longhorn.raspb1s.local
```

Access UI at: http://longhorn.raspb1s.local

Storage path: `/mnt/k3s-storage/longhorn`
