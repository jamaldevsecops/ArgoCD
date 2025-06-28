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
Create the `myapp` namespace and a DockerHub image pull secret named `mygithub-creds`.
```bash
# Creating the myapp namespace and DockerHub image pull secret
kubectl create namespace myapp
```
```bash
kubectl create secret docker-registry mydockerhub-creds \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username="jamaldevsecops" \
  --docker-password="dckr_pat_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX" \
  -n "myapp"
```
## 5. DockerHub Secret for Argo CD Image Updater to Track Image
Create a Kubernetes secret for Argo CD Image Updater to authenticate to DockerHub.

```bash
kubectl create secret docker-registry mydockerhub-creds \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username="jamaldevsecops" \
  --docker-password="dckr_pat_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX" \
  -n "argocd"
```

**Steps**:
Replace `jamaldevsecops` and `your-dockerhub-pat`.

**Notes**:
- If Image is public then no required to create this dockerhub secret for argocd.

## 6. Argo CD Image Updater ConfigMap
Configure Argo CD Image Updater to track DockerHub with polling. Reference The Secret in Argo CD Image Updater ConfigMap
```bash
kubectl edit configmap argocd-image-updater-config -n argocd
```
Or, 
```bash
kubectl -n argocd delete configmap argocd-image-updater-config
```
```bash
cat > argocd-image-updater-config.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-image-updater-config
  namespace: argocd
data:
  log.level: info
  registries.conf: |
    registries:
      - name: Docker Hub
        prefix: docker.io
        api_url: https://index.docker.io/v1/
        ping: yes
        credentials: secret:argocd/mydockerhub-creds
EOF
```

**Steps**:
Apply:
   ```bash
   kubectl apply -f argocd-image-updater-config.yaml
   ```
Label the Secret for Argo CD Image Updater
```bash
kubectl label secret mydockerhub-creds \
  argocd.argoproj.io/secret-type=registry-credential \
  -n argocd
```
Restart Image Updater
```bash
kubectl rollout restart deployment argocd-image-updater -n argocd
```

## 7. GitHub Secret for Argo CD Image Updater (Write Access)
Create a secret for Argo CD Image Updater to push updates to `kustomization.yaml`.

```bash
# Kubernetes secret for GitHub write access
kubectl create secret generic mygithub-creds \
  --namespace argocd \
  --from-literal=username=jamaldevsecops \
  --from-literal=password=ghp_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

## 8. Argo CD Project YAML
Define an Argo CD Project to restrict syncs to the `myapp` namespace.

**YAML Manifest**:
```yaml
# Argo CD Project for myapp
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: myapp
  namespace: argocd
spec:
  description: Project for myapp application
  sourceRepos:
    - https://github.com/jamaldevsecops/kubernetes.git
  destinations:
    - namespace: '*'
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

## 9. Argo CD Application YAML with Multi-Image Update Strategy (Sementic Version)
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
    # Image list with semver tags
    argocd-image-updater.argoproj.io/image-list: myapp=docker.io/jamaldevsecops/myapp:v1.x

    # Git write-back settings
    argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/mygithub-creds
    argocd-image-updater.argoproj.io/git-branch: master

    # Use semver strategy for both images
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver

    # Optional: force update if needed
    argocd-image-updater.argoproj.io/myapp.force-update: "true"

spec:
  project: myapp
  source:
    repoURL: https://github.com/jamaldevsecops/kubernetes.git
    targetRevision: HEAD
    path: argocd/k8s-manifest/myapp
    kustomize: {}  # Optional, but explicit for Kustomize-based apps
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
For multiple images
```yaml
# Argo CD Project for myapp
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: myapp
  namespace: argocd
spec:
  description: Project for myapp application
  sourceRepos:
    - https://github.com/jamaldevsecops/kubernetes.git
  destinations:
    - namespace: '*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
---
# Argo CD Application for myapp with Image Updater
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  annotations:
    # Image list with semver tags
    argocd-image-updater.argoproj.io/image-list: |
      myapp1=docker.io/jamaldevsecops/myapp1:v1.x,
      myapp2=docker.io/jamaldevsecops/myapp2:v1.x

    # Git write-back settings
    argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/mygithub-creds
    argocd-image-updater.argoproj.io/git-branch: master

    # Use semver strategy for both images
    argocd-image-updater.argoproj.io/myapp1.update-strategy: semver
    argocd-image-updater.argoproj.io/myapp2.update-strategy: semver

    # Optional: force update if needed
    argocd-image-updater.argoproj.io/myapp1.force-update: "true"
    argocd-image-updater.argoproj.io/myapp2.force-update: "true"

spec:
  project: myapp
  source:
    repoURL: https://github.com/jamaldevsecops/kubernetes.git
    targetRevision: HEAD
    path: argocd/k8s-manifest/myapp
    kustomize: {}  # Optional, but explicit for Kustomize-based apps
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
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - myapp-deployment-hpa-service.yaml
  - myapp-namespace-and-nginx-ingress.yaml

images:
  - name: docker.io/jamaldevsecops/myapp
    newTag: v1.1 # This tag will be automatically updated by Argo CD Image Updater
```
For multiple images:
```yaml
# Kustomization file for myapp
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - myapp1-deployment-hpa-service.yaml
  - myapp2-deployment-hpa-service.yaml
  - myapp-nginx-ingress.yaml

images:
  - name: docker.io/jamaldevsecops/myapp1
    newTag: v1.1  # Placeholder; will be updated by Argo CD Image Updater
  - name: docker.io/jamaldevsecops/myapp2
    newTag: v1.1  # Placeholder; will be updated by Argo CD Image Updater
```

**Steps**:
1. Ensure listed files exist in `argocd/k8s-manifest/myapp`.
2. Commit and push to the repository.

## 11. Adjust Deployment YAML Image Pull Secret
Sample Deployment YAML `myapp-frontend-deployment-hpa-service.yaml` with `imagePullSecrets`.

**YAML Manifest**:
```yaml
# Deployment for myapp
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  namespace: myapp # You can change this to your specific namespace
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: docker.io/jamaldevsecops/myapp
        ports:
        - containerPort: 80
        #resources:
          #requests:
            #memory: "16Mi"
            #cpu: "50m"
          #limits:
            #memory: "1Gi"
            #cpu: "1"
      imagePullSecrets:
      - name: mydockerhub-creds
```

**Notes**:
- Add similar configuration to `myapp-backend-deployment-hpa-service.yaml` with `update-strategy: digest`.
- If Image is public then remove `imagePullSecrets`.

## 12. TLS Secret for myapp Namespace
Create a TLS secret for Ingress.

**Command**:
```bash
kubectl -n myapp create secret tls myapp-tls --cert=/path/to/CA_chain.crt --key=/path/to/key.pem
```

## 13. Access the Application via HTTPS Ingress
Access the application using the Ingress resource and TLS.

**Sample Ingress YAML**:
```yaml
 # Ingress Resource for myapp
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.apsissolutions.com
    secretName: apsis-tls
  rules:
  - host: myapp.apsissolutions.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
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
