# Runbook: Deploying a Web Application with ArgoCD and Image Updater

This runbook provides a **step-by-step procedure** to deploy a web application (`myapp`) in Kubernetes using **ArgoCD** and **ArgoCD Image Updater**, with built-in validation checks after each step.

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
kubectl create namespace myapp
```

✅ **Validation**:
```bash
kubectl get ns | grep myapp
```
You should see the `myapp` namespace.

---

## 3. Create a DockerHub Secret for Pulling Private Images

```bash
kubectl create secret docker-registry mydockerhub-creds \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username="jamaldevsecops" \
  --docker-password="dckr_pat_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX" \
  -n myapp
```

✅ **Validation**:
```bash
kubectl get secret mydockerhub-creds -n myapp
```
Secret should be listed.

---

## 4. Copy DockerHub Secret into ArgoCD Namespace & Configure Image Updater

```bash
kubectl get secret mydockerhub-creds -n myapp -o yaml | \
  sed 's/namespace: myapp/namespace: argocd/' | kubectl apply -f -
```

Now configure `argocd-image-updater-config`:
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
kubectl create secret generic mygithub-creds \
  --namespace argocd \
  --from-literal=username=jamaldevsecops \
  --from-literal=password=ghp_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

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
  - myapp-deployment-hpa-service.yaml
  - myapp-nginx-ingress.yaml

images:
  - name: docker.io/jamaldevsecops/myapp
    newTag: v1.1.1
```

✅ **Validation**:
Ensure this file is committed in GitHub repo under `argocd/k8s-manifest/myapp/`.

---

## 7. Create TLS Secret for Ingress

```bash
kubectl -n myapp create secret tls myapp-tls \
  --cert=/path/to/CA_chain.crt \
  --key=/path/to/key.crt
```

✅ **Validation**:
```bash
kubectl get secret myapp-tls -n myapp
```

---

## 8. Create ArgoCD Project

`myapp-project.yaml`:
```yaml
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
```bash
kubectl apply -f myapp-project.yaml
```

✅ **Validation**:
```bash
kubectl get appproject -n argocd
```

---

## 9. Create ArgoCD Application

`myapp-application.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=docker.io/jamaldevsecops/myapp:v1.x.x
    argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/mygithub-creds
    argocd-image-updater.argoproj.io/git-branch: master
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver
    argocd-image-updater.argoproj.io/myapp.force-update: "true"
spec:
  project: myapp
  source:
    repoURL: https://github.com/jamaldevsecops/kubernetes.git
    targetRevision: HEAD
    path: argocd/k8s-manifest/myapp
    kustomize: {}
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
```bash
kubectl apply -f myapp-application.yaml
```

✅ **Validation**:
```bash
kubectl get applications -n argocd
```
Check if `myapp` status is **Healthy** and **Synced** in ArgoCD UI.

---

## 10. Deploy Application Resources

Example resources (`myapp-deployment-hpa-service.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  namespace: myapp
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
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
      imagePullSecrets:
      - name: mydockerhub-creds
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-deployment
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
  name: myapp-service
  namespace: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
```

`myapp-nginx-ingress.yaml`:
```yaml
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
    secretName: myapp-tls
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
              number: 3000
```

✅ **Validation**:
```bash
kubectl get pods -n myapp
kubectl get svc -n myapp
kubectl get ingress -n myapp
```
Test application in browser at `https://myapp.apsissolutions.com`.

---

## 11. GitHub Repo Structure

```
├── argocd
│   └── k8s-manifest
│       └── myapp
│           ├── kustomization.yaml
│           ├── myapp-deployment-hpa-service.yaml
│           └── myapp-nginx-ingress.yaml
```

✅ **Validation**:
Ensure manifests exist in GitHub repo and are accessible to ArgoCD.

---

## Final Validation Checklist

- [ ] ArgoCD and Image Updater pods are running.
- [ ] `myapp` namespace created.
- [ ] DockerHub secret in `myapp` and `argocd` namespaces.
- [ ] GitHub secret created in `argocd`.
- [ ] Kustomization.yaml committed in repo.
- [ ] TLS secret created.
- [ ] ArgoCD Project and Application exist.
- [ ] Deployment, Service, HPA, Ingress created.
- [ ] Application reachable at `https://myapp.apsissolutions.com`.
- [ ] Image updates auto-trigger and sync back to Git.

---

✅ You now have a **deploy + validate at the same time runbook** for ArgoCD-managed application deployments with auto image updates.
