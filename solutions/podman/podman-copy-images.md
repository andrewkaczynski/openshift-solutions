# Copy images from public registries to local one

1. Get a copy of your image (e.g. registry.k8s.io/ingress-nginx/controller:v1.10.0):

```bash
podman pull registry.k8s.io/ingress-nginx/controller:v1.10.0
```

2. Tag with new registry name:

```bash
podman tag registry.k8s.io/ingress-nginx/controller:v1.10.0 <your_registry>/registry.k8s.io/ingress-nginx/controller:v1.10.0
```

3. Push to local registry:

```bash
podman push <your_registry>/registry.k8s.io/ingress-nginx/controller:v1.10.0
```