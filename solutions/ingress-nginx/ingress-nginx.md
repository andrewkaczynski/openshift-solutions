# Ingress-Nginx

This solution shows how to install the ingress-nginx, an open source ingress-controller on the OpenShift cluster

[https://github.com/kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx)

## Prerequisites

Installation was tested with versions:
- ingress-nginx v1.9.6
- helm chart 4.9.1
- OpenShift 3 or 4

Cluster requirements
- Min 3x worker nodes for ingress-nginx controller copies.
- Worker nodes labeled with **ingress-controller=ingress-nginx**.
- External load balancer with direct access to worker nodes IPs and ports in range 30000-32767.

Installation environment:
- Linux (preferably) or Windows-based system
- oc or kubectl cli (they can be used interchangeably)
- helm cli >= 3.0

Diagram:

![ingress-nginx schema](ingress-nginx-openshift.drawio.png "ingress-nginx")

The following images should be transferred to the locally available registry if cluster nodes do not have direct access to the internet:
- registry.k8s.io/ingress-nginx/controller:v1.9.4

The instructions on how to transfer images to the local repository are available at: [Copy images from public registries to local one](../podman/podman-copy-images.md)

## Installation

The mentioned installation process was performed on the Linux host with direct access to the internet.

1. Download the chart:

```bash
wget https://github.com/kubernetes/ingress-nginx/releases/download/helm-chart-4.9.1/ingress-nginx-4.9.1.tgz
tar zxf ingress-nginx-4.9.1.tgz
cd ingress-nginx
```

2. Modify the template files:

```bash
rm templates/admission-webhooks/job-patch/role.yaml
vi templates/admission-webhooks/job-patch/role.yaml

{{- if and .Values.controller.admissionWebhooks.enabled .Values.controller.admissionWebhooks.patch.enabled (not .Values.controller.admissionWebhooks.certManager.enabled) -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ include "ingress-nginx.admissionWebhooks.fullname" . }}
  namespace: {{ include "ingress-nginx.namespace" . }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: admission-webhook
    {{- with .Values.controller.admissionWebhooks.patch.labels }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
rules:
  - apiGroups:
      - ""
    resources:
      - secrets
    verbs:
      - get
      - create
  - apiGroups: ["security.openshift.io"]
    resources: ["securitycontextconstraints"]
    resourceNames: ["privileged"]
    verbs: ["use"]
{{- end }}
#save

vi templates/ssc.yaml

# Create SCC for IC resources
kind: SecurityContextConstraints
apiVersion: security.openshift.io/v1
metadata:
  name: {{ include "ingress-nginx.fullname" . }}-scc
allowPrivilegedContainer: false
runAsUser:
  type: MustRunAs
  uid: 101
seLinuxContext:
  type: MustRunAs
fsGroup:
  type: MustRunAs
supplementalGroups:
  type: MustRunAs
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowHostDirVolumePlugin: false
allowHostIPC: false
readOnlyRootFilesystem: false
seccompProfiles:
- runtime/default
volumes:
- secret
requiredDropCapabilities:
- ALL
users:
- system:serviceaccount:{{ .Release.Namespace }}:{{ include "ingress-nginx.serviceAccountName" . }}
allowedCapabilities:
- NET_BIND_SERVICE
#save
```

3. Prepare the custom values.yaml for the installation:

```bash
vim values-custom.yaml

controller:
  image:
    registry: <local_registry_url>
    image: ingress-nginx/controller
    tag: "v1.9.6"
    digest: ""
  allowSnippetAnnotations: true
  kind: DaemonSet
  nodeSelector:
    ingress-controller: ingress-nginx
  service:
    type: NodePort
    externalTrafficPolicy: "Local"
    nodePorts:
      http: "30080"
      https: "30443"
  admissionWebhooks:
    enabled: false
```

4. Label nodes where ingress-nginx PODs will be scheduled:

```bash
kubectl label node <nodeName> ingress-controller=ingress-nginx
# Repeat on every node
```

5. Install ingress-nginx in its namespace:

```bash
# check if templates are correctly rendered
helm template -n ingress-nginx ingress-nginx . -f values-custom.yaml

# install
helm install -n ingress-nginx ingress-nginx . -f values-custom.yaml --create-namespace
```

## Validation

1. Check if PODs are correctly running in the ingress-nginx namespace:

```bash
kubectl get pod -n ingress-nginx
```

2. Deploy test application:

```bash
apiVersion: v1
kind: Namespace
metadata:
  name: httpbin
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  namespace: httpbin
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: httpbin
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
  namespace: httpbin
spec:
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      containers:
      - name: httpbin
        image: mccutchen/go-httpbin:latest
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: httpbin
  namespace: httpbin
spec:
  ingressClassName: nginx
  rules:
    - host: httpbin.com
      http:
        paths:
          - pathType: Prefix
            backend:
              service:
                name: httpbin
                port:
                  number: 80
            path: /
```

3. Run the test call towards the External Load Balancer IP with a custom Host header to allow ingress-controller to route the request to your application:

```bash
curl -H "Host: httpbin.com" http://<LoadBalancerIP>/get
```
