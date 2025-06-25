
# Step-by-Step Deployment Guide: Argo CD Image Updater with Private GitHub and DockerHub

---
## 1. Argo CD Image Updater Installation

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/master/manifests/install.yaml
```

---
## 2. Fine-Grained GitHub PAT for Write Access

1. Go to GitHub > Settings > Developer Settings > Personal access tokens (Fine-grained)
2. Generate new token:
   - Repository access: `Only select repositories` → `jamaldevsecops/kubernetes`
   - Permissions:
     - Contents: `Read and write`
     - Metadata: `Read-only`
3. Save the token securely.

---
## 3. DockerHub Read-Only PAT Creation

1. Go to https://hub.docker.com/settings/security
2. Create an access token with description `ArgoCD-RO`
3. Use your DockerHub username and token as credentials.

---
## 4. Namespace and DockerHub Secret for Image Pull

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
---
apiVersion: v1
kind: Secret
metadata:
  name: dockerhub-pull-secret
  namespace: myapp
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

Create the secret using command:
```bash
kubectl create secret docker-registry dockerhub-pull-secret   --docker-username=<username>   --docker-password=<token>   --docker-server=https://index.docker.io/v1/   --namespace=myapp
```

---
## 5. DockerHub Secret for Argo CD Image Updater

```bash
kubectl create secret docker-registry dockerhub-image-updater-secret   --docker-username=<username>   --docker-password=<token>   --docker-server=https://index.docker.io/v1/   -n argocd
```

---
## 6. Argo CD Image Updater ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-image-updater-config
  namespace: argocd
data:
  registries.conf: |
    registries:
    - name: DockerHub
      prefix: docker.io
      credentials: secret:argocd/dockerhub-image-updater-secret
  config.yaml: |
    logLevel: info
    interval: 1m
    gitWriteBehavior: commit
```

---
## 7. GitHub Secret for Image Updater

```bash
kubectl create secret generic github-writer-secret   --from-literal=username=jamaldevsecops   --from-literal=password=<GITHUB_FINE_GRAINED_PAT>   -n argocd
```

---
## 8. Argo CD Project YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: myapp-project
  namespace: argocd
spec:
  destinations:
    - namespace: myapp
      server: https://kubernetes.default.svc
  sourceRepos:
    - https://github.com/jamaldevsecops/kubernetes
  clusterResourceWhitelist:
    - group: "*"
      kind: "*"
```

---
## 9. Argo CD Application with Multi-Image Update

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/write-back-target: kustomization
spec:
  project: myapp-project
  source:
    repoURL: https://github.com/jamaldevsecops/kubernetes
    targetRevision: HEAD
    path: argocd/k8s-manifest/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
  metadata:
    annotations:
      argocd-image-updater.argoproj.io/image-list: jamaldevsecops/myapp-frontend,jamaldevsecops/myapp-backend
      argocd-image-updater.argoproj.io/myapp-frontend.update-strategy: semver
      argocd-image-updater.argoproj.io/myapp-frontend.allow-tags: regexp:^v1\..*
      argocd-image-updater.argoproj.io/myapp-backend.update-strategy: digest
```

---
## 10. kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - myapp-frontend-deployment-hpa-service.yaml
  - myapp-backend-deployment-hpa-service.yaml
  - myapp-nginx-ingress.yaml
```

---
## 11. Sample Deployment with Annotations

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-frontend
  labels:
    app: myapp-frontend
  annotations:
    argocd-image-updater.argoproj.io/image-name: jamaldevsecops/myapp-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp-frontend
  template:
    metadata:
      labels:
        app: myapp-frontend
    spec:
      containers:
        - name: myapp-frontend
          image: docker.io/jamaldevsecops/myapp-frontend:latest
          ports:
            - containerPort: 80
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
      imagePullSecrets:
        - name: dockerhub-pull-secret
```

---
## 12. TLS Secret for myapp Namespace

```bash
kubectl create secret tls myapp-tls   --cert=fullchain.pem   --key=privkey.pem   -n myapp
```

---
## 13. Access via HTTPS Ingress

Ensure Ingress definition uses:

```yaml
  tls:
    - hosts:
        - myapp.apsissolutions.com
      secretName: myapp-tls
```

Update DNS A record for `myapp.apsissolutions.com` to point to your ingress controller IP.
