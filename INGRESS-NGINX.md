# Nginx Ingress Controller Installation

## Installation with Helm

```bash
# Add helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install nginx ingress controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --values ingress-nginx-values.yaml

# Or upgrade existing installation
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --values ingress-nginx-values.yaml
```

## Uninstall

```bash
helm uninstall ingress-nginx --namespace ingress-nginx
kubectl delete namespace ingress-nginx
```

## Check Status

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
kubectl get ingressclass
```

## Ports

- **30080** - HTTP (NodePort)
- **30443** - HTTPS (NodePort)

## Configuration

All configuration is in `ingress-nginx-values.yaml` file.
