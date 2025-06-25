# Argo CD Image Updater Deployment Guide
This guide provides step-by-step instructions for deploying Argo CD Image Updater in a Kubernetes cluster, configuring GitHub and DockerHub access, and setting up related resources for the myapp application. All YAML manifests and commands are included.

## 1. Argo CD Image Updater Installation
To deploy Argo CD Image Updater in a Kubernetes cluster where Argo CD is already installed, ensure Argo CD is in the `argocd` namespace, apply the manifests below, and verify RBAC.

**Installation Steps**:
1. Apply official installation manifest:
   ```bash
   kubectl get pods -n argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
   ```
2. Verify the deployment:
   ```bash
   kubectl -n argocd get pods -l app.kubernetes.io/name=argocd-image-updater
   ```

## 2. GitHub Fine-Grained PAT for Argo CD Image Updater (Write Access)
To create a fine-grained GitHub Personal Access Token (PAT) with write access to `argocd/k8s-manifest/myapp/kustomization.yaml` in the `jamaldevsecops/kubernetes` private repository:

**Steps**:
1. Go to GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → New token.
2. Configure:
   - **Name**: `argocd-image-updater`
   - **Expiration**: e.g., 90 days
   - **Repository access**: Select `jamaldevsecops/kubernetes`
   - **Permissions**:
     - **Contents**: Read and write
     - **Metadata**: Read-only (implicit)
3. Generate and save the token securely (e.g., `ghp_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX`).

**Note**: File-level granularity isn’t supported, but Argo CD Image Updater will only modify the specified file based on its configuration.

## 3. DockerHub Read-Only PAT Creation
To create a DockerHub PAT with read-only permission for tracking private images:

**Steps**:
1. Log in to hub.docker.com.
2. Go to Account Settings → Security → Personal Access Tokens → New Personal Access Token.
3. Configure:
   - **Description**: `argocd-image-updater-read`
   - **Access permissions**: Read-only
4. Generate and save the token (e.g., `dckr_pat_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX`).

## 4. Namespace and DockerHub Secret for Pulling Private Image
Create the `myapp` namespace and a DockerHub image pull secret named `dockerhub-pull-secret`.

**YAML Manifest**:
```yaml
# Creating the myapp namespace and DockerHub image pull secret
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

**Steps**:
1. Create the Docker config JSON:
   ```bash
   echo -n '{"auths":{"https://index.docker.io/v1/":{"username":"your-username","password":"your-dockerhub-pat"}}}' | base64
   ```
   Replace `your-username` and `your-dockerhub-pat` with your credentials.
2. Replace `<base64-encoded-docker-config>` in the YAML.
3. Apply:
   ```bash
   kubectl apply -f myapp-namespace-and-pull-secret.yaml
   ```

## 5. DockerHub Secret for Argo CD Image Updater to Track Image
Create a Kubernetes secret for Argo CD Image Updater to authenticate to DockerHub.

**YAML Manifest**:
```yaml
# Kubernetes secret for Argo CD Image Updater to access DockerHub
apiVersion: v1
kind: Secret
metadata:
  name: dockerhub-image-updater-secret
  namespace: argocd
type: Opaque
stringData:
  registry: docker.io
  username: your-username
  password: your-dockerhub-pat
```

**Steps**:
1. Replace `your-username` and `your-dockerhub-pat`.
2. Apply:
   ```bash
   kubectl apply -f dockerhub-image-updater-secret.yaml
   ```

## 6. Argo CD Image Updater ConfigMap
Configure Argo CD Image Updater to track DockerHub with polling and Git write-back.

**YAML Manifest**:
```yaml
# ConfigMap for Argo CD Image Updater
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-image-updater-config
  namespace: argocd
data:
  config.yml: |
    registries:
      - name: docker.io
        api_url: https://registry-1.docker.io
        credentials: secret:argocd/dockerhub-image-updater-secret
        prefix: docker.io
    applications:
      myapp:
        interval: 2m
        write_back_method: git
        git_branch: main
        git_commit_message: "Update image tags by Argo CD Image Updater [skip ci]"
        allow_tags: regexp:^v[0-9]+\.[0-9]+$
```

**Steps**:
1. Apply:
   ```bash
   kubectl apply -f argocd-image-updater-config.yaml
   ```

## 7. GitHub Secret for Argo CD Image Updater (Write Access)
Create a secret for Argo CD Image Updater to push updates to `kustomization.yaml`.

**YAML Manifest**:
```yaml
# Kubernetes secret for GitHub write access
apiVersion: v1
kind: Secret
metadata:
  name: github-writer-secret
  namespace: argocd
type: Opaque
stringData:
  type: github
  url: https://github.com/jamaldevsecops/kubernetes
  username: your-github-username
  password: your-github-pat
```

**Steps**:
1. Replace `your-github-username` and `your-github-pat`.
2. Apply:
   ```bash
   kubectl apply -f github-writer-secret.yaml
   ```

## 8. Argo CD Project YAML
Define an Argo CD Project to restrict syncs to the `myapp` namespace.

**YAML Manifest**:
```yaml
# Argo CD Project for myapp
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: myapp-project
  namespace: argocd
spec:
  description: Project for myapp application
  sourceRepos:
  - https://github.com/jamaldevsecops/kubernetes
  destinations:
  - namespace: myapp
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  namespaceResourceWhitelist:
  - group: '*'
    kind: '*'
```

**Steps**:
1. Apply:
   ```bash
   kubectl apply -f myapp-project.yaml
   ```

## 9. Argo CD Application YAML with Multi-Image Update Strategy
Define an Argo CD Application to track two images with different strategies.

**YAML Manifest**:
```yaml
# Argo CD Application for myapp with Image Updater
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: frontend=jamaldevsecops/myapp-frontend,backend=jamaldevsecops/myapp-backend
    argocd-image-updater.argoproj.io/frontend.update-strategy: semver:v1.x
    argocd-image-updater.argoproj.io/backend.update-strategy: digest
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
    argocd-image-updater.argoproj.io/write-back-target: kustomization:argocd/k8s-manifest/myapp/kustomization.yaml
    argocd-image-updater.argoproj.io/git-secret: github-writer-secret
spec:
  project: myapp-project
  source:
    repoURL: https://github.com/jamaldevsecops/kubernetes
    path: argocd/k8s-manifest/myapp
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Steps**:
1. Apply:
   ```bash
   kubectl apply -f myapp-application.yaml
   ```

## 10. kustomization.yaml Including Deployment and Ingress Files
Define a `kustomization.yaml` to include specified resources.

**YAML Manifest**:
```yaml
# Kustomization file for myapp
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- myapp-frontend-deployment-hpa-service.yaml
- myapp-backend-deployment-hpa-service.yaml
- myapp-nginx-ingress.yaml
```

**Steps**:
1. Ensure listed files exist in `argocd/k8s-manifest/myapp`.
2. Commit and push to the repository.

## 11. Adjust Deployment YAML for Argo CD Image Updater Annotations
Sample Deployment YAML with Image Updater annotations and best practices.

**YAML Manifest**:
```yaml
# Sample Kubernetes Deployment with Image Updater annotations
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-frontend
  namespace: myapp
  labels:
    app: myapp-frontend
  annotations:
    argocd-image-updater.argoproj.io/image-name: jamaldevsecops/myapp-frontend
    argocd-image-updater.argoproj.io/update-strategy: semver:v1.x
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
      imagePullSecrets:
      - name: dockerhub-pull-secret
      containers:
      - name: myapp-frontend
        image: jamaldevsecops/myapp-frontend:v1.0.0
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "100m"
            memory: "128Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Notes**:
- Add similar annotations to `myapp-backend-deployment-hpa-service.yaml` with `update-strategy: digest`.
- Update image and probes as needed.

## 12. TLS Secret for myapp Namespace
Create a TLS secret for Ingress.

**Command**:
```bash
kubectl -n myapp create secret tls myapp-tls --cert=/path/to/cert.pem --key=/path/to/key.pem
```

**Alternative YAML**:
```yaml
# TLS secret for myapp Ingress
apiVersion: v1
kind: Secret
metadata:
  name: myapp-tls
  namespace: myapp
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

**Steps**:
1. Encode certificate and key:
   ```bash
   cat /path/to/cert.pem | base64
   cat /path/to/key.pem | base64
   ```
2. Replace placeholders in the YAML.
3. Apply:
   ```bash
   kubectl apply -f myapp-tls-secret.yaml
   ```

## 13. Access the Application via HTTPS Ingress
Access the application using the Ingress resource and TLS.

**Sample Ingress YAML**:
```yaml
# Ingress for myapp with TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-nginx-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.apsissolutions.com
    secretName: myapp-tls
  rules:
  - host: myapp.apsissolutions.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-frontend-service
            port:
              number: 8080
```

**Steps**:
1. Ensure NGINX Ingress Controller is installed.
2. Apply the Ingress YAML (in repository).
3. Update DNS to point `myapp.apsissolutions.com` to the Ingress Controller’s IP:
   ```bash
   kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
   ```
4. Access at `https://myapp.apsissolutions.com`.

**Notes**:
- Ensure `myapp-frontend-service` exists.
- Verify TLS certificate for `myapp.apsissolutions.com`.

---
Generated on June 25, 2025
