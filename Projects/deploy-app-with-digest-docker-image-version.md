# Deploying a Web Application with ArgoCD and Image Updater (Semantic Docker Image Tag - jamaldevsecops/myapp2@sha256:481e4358870b417....)

This runbook provides a **step-by-step procedure** to deploy a web application (`myapp2`) in Kubernetes using **ArgoCD** and **ArgoCD Image Updater**, with built-in validation checks after each step.

---

## 1. Pre-requisites

- A working Kubernetes cluster (v1.20+ recommended).
- ArgoCD installed in the `argocd` namespace.
- ArgoCD Image Updater installed and configured.
- kubectl access configured to point to your cluster.

✅ **Validation**:
```bash
kubectl get pods -n argocd
kubectl get pods -n argocd | grep image-updater
```
You should see ArgoCD and Image Updater pods in **Running** state.

---

## 2. Create a Namespace for the Project

```bash
kubectl create namespace myapp2
```

✅ **Validation**:
```bash
kubectl get ns | grep myapp2
```
You should see the `myapp2` namespace.

---

## 3. Create a DockerHub Secret for Pulling Private Images

```bash
# safer: set env vars to avoid storing secrets in shell history
export DH_USER="jamaldevsecops"
export DH_PASS="dckr_pat_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"

kubectl create secret docker-registry mydockerhub-creds \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username="$DH_USER" \
  --docker-password="$DH_PASS" \
  -n myapp2
```
Update the Deployment to use the secret:
```
imagePullSecrets:
- name: mydockerhub-creds
```

✅ **Validation**:
```bash
kubectl get secret mydockerhub-creds -n myapp2
```
Secret should be listed.

---

## 4. Copy DockerHub Secret into ArgoCD Namespace & Configure Image Updater

```bash
kubectl get secret mydockerhub-creds -n myapp2 -o yaml | \
  sed 's/namespace: myapp2/namespace: argocd/' | kubectl apply -f -
```

Now configure `argocd-image-updater-config`:
```bash
kubectl edit configmap argocd-image-updater-config -n argocd
```
```yaml
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
```
Apply it:
```bash
kubectl apply -f argocd-image-updater-config.yaml
```

✅ **Validation**:
```bash
kubectl describe configmap argocd-image-updater-config -n argocd
```
Ensure registry config is loaded.

---

## 5. Create a GitHub Secret for Write-back Access

```bash
export GH_USER="jamaldevsecops"
export GH_PASS="ghp_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"

kubectl create secret generic mygithub-creds \
  --from-literal=username="$GH_USER" \
  --from-literal=password="$GH_PASS" \
  --namespace argocd
```
🔐 Recommendation: Replace plain secrets with SealedSecrets or ExternalSecrets for secure management.

✅ **Validation**:
```bash
kubectl get secret mygithub-creds -n argocd
```

---

## 6. Add Kustomization File in Manifest Repository

`kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- myapp2-deployment-hpa-service.yaml
- myapp2-nginx-ingress.yaml

images:
- name: docker.io/jamaldevsecops/myapp2
  newTag: latest
```

✅ **Validation**:
Ensure this file is committed in GitHub repo under `argocd/k8s-manifest/myapp2/`.

---

## 7. Create TLS Secret for Ingress

```bash
kubectl -n myapp2 create secret tls myapp2-tls \
  --cert=/path/to/CA_chain.crt \
  --key=/path/to/key.crt
```

✅ **Validation**:
```bash
kubectl get secret myapp2-tls -n myapp2
```

---

## 8. Create ArgoCD Project

`myapp2-project.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: myapp2-project
  namespace: argocd
spec:
  description: Project for myapp2 application
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
```bash
kubectl apply -f myapp2-project.yaml
```

✅ **Validation**:
```bash
kubectl get appproject -n argocd
```

---

## 9. Create ArgoCD Application

`myapp2-application.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp2-application
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp2=docker.io/jamaldevsecops/myapp2:latest
    argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/mygithub-creds
    argocd-image-updater.argoproj.io/git-branch: master
    argocd-image-updater.argoproj.io/myapp2.update-strategy: digest
    argocd-image-updater.argoproj.io/myapp2.force-update: "true"
    argocd-image-updater.argoproj.io/myapp2.pin-digest: "true"
spec:
  project: myapp2-project
  source:
    repoURL: https://github.com/jamaldevsecops/kubernetes.git
    targetRevision: master
    path: argocd/k8s-manifest/myapp2
    kustomize: {}
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp2
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```
```bash
kubectl apply -f myapp2-application.yaml
```

✅ **Validation**:
```bash
kubectl get applications -n argocd
```
Check if `myapp2` status is **Healthy** and **Synced** in ArgoCD UI.

---

## 10. Deploy Application Resources

Example resources (`myapp2-deployment-hpa-service.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp2-deployment
  namespace: myapp2
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: myapp2
  template:
    metadata:
      labels:
        app: myapp2
    spec:
      containers:
      - name: myapp2
        image: docker.io/jamaldevsecops/myapp2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
      imagePullSecrets:
      - name: mydockerhub-creds
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp2-hpa
  namespace: myapp2
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp2-deployment
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
---
apiVersion: v1
kind: Service
metadata:
  name: myapp2-service
  namespace: myapp2
spec:
  type: ClusterIP
  selector:
    app: myapp2
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

`myapp2-nginx-ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp2-ingress
  namespace: myapp2
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp2.apsissolutions.com
    secretName: myapp2-tls
  rules:
  - host: myapp2.apsissolutions.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp2-service
            port:
              number: 80
```

✅ **Validation**:
```bash
kubectl get pods -n myapp2
kubectl get svc -n myapp2
kubectl get ingress -n myapp2
```
Test application in browser at `https://myapp2.apsissolutions.com`.

---

## 11. GitHub Repo Structure

```
📦 kubernetes/
 ┣ 📂 argocd/
 ┃ ┣ 📂 k8s-manifest/
 ┃ ┃ ┗ 📂 myapp2/
 ┃ ┃   ┣ 📜 kustomization.yaml
 ┃ ┃   ┣ 📜 myapp2-deployment-hpa-service.yaml
 ┃ ┃   ┣ 📜 myapp2-nginx-ingress.yaml
 ┃ ┃   ┗ 📜 README.md (docs for app)
 ┣ 📂 base/
 ┃ ┗ (optional: reusable manifests for multiple apps)
 ┗ 📜 README.md
```
## 12. Improvements & Best Practices
```
🔐 Use SealedSecrets / ExternalSecrets for managing GitHub and DockerHub credentials.
🔒 Add RBAC restrictions in ArgoCD Project (limit deployment to myapp2 namespace only).
🔄 Use sync hooks for complex deployments (e.g., DB migrations before rollout).
⚖️ Define resource requests/limits for CPU and memory to ensure proper scheduling.
📈 Integrate monitoring (ServiceMonitor/Prometheus/Grafana) for metrics and alerting.
🧹 Enable prune and self-heal in ArgoCD sync policies for GitOps consistency.
```

✅ **Validation**:
Ensure manifests exist in GitHub repo and are accessible to ArgoCD.

---

## Final Validation Checklist

- [ ] ArgoCD and Image Updater pods are running.
- [ ] `myapp2` namespace created.
- [ ] DockerHub secret in `myapp2` and `argocd` namespaces.
- [ ] GitHub secret created in `argocd`.
- [ ] Kustomization.yaml committed in repo.
- [ ] TLS secret created.
- [ ] ArgoCD Project and Application exist.
- [ ] Deployment, Service, HPA, Ingress created.
- [ ] Application reachable at `https://myapp2.apsissolutions.com`.
- [ ] Image updates auto-trigger and sync back to Git.

---

✅ You now have a **deploy + validate at the same time runbook** for ArgoCD-managed application deployments with auto image updates.
