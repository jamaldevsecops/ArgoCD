# ArgoCD Sample Application Deployment

This guide covers creating an ArgoCD project and a new application (for both public and private Git repositories) using the Web UI and CLI. The application uses the Git repository `git@github.com:jamaldevsecops/ArgoCD.git`, manifest path `k8s-manifest/myapp/for-nginx-ingress-contoller`, and deploys to the `myapp` namespace. The ArgoCD UI is accessible at `https://argocd.apsissolutions.com`, secured with Kubernetes TLS using the `apsis-tls` secret.

## Prerequisites
- ArgoCD installed in the Kubernetes cluster (namespace: `argocd`).
- ArgoCD UI accessible at `https://argocd.apsissolutions.com` with TLS configured.
- Access to the ArgoCD Web UI and CLI (`argocd` command installed).
- Admin credentials for ArgoCD (username: `admin`, password retrieved via `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`).
- A Kubernetes cluster with the `myapp` namespace created:
  ```bash
  kubectl create namespace myapp
  ```
- For private repositories, SSH or HTTPS credentials configured in ArgoCD.
- NGINX Ingress Controller installed for application exposure.
- Git repository (`git@github.com:jamaldevsecops/ArgoCD.git`) with Kubernetes manifests in the specified path (`k8s-manifest/myapp/for-nginx-ingress-contoller`).

## 1. Creating a Project

Projects in ArgoCD organize applications, define allowed source repositories, destinations, and roles for access control.

### Using Web UI
1. **Access the Web UI**:
   - Navigate to `https://argocd.apsissolutions.com`.
   - Log in with username `admin` and the admin password.

2. **Create a Project**:
   - Go to **Settings** > **Projects** in the left menu.
   - Click **+ New Project**.
   - Fill in the details:
     - **Project Name**: `myapp-project`
     - **Description**: "Project for myapp deployments"
     - **Source Repositories**: Add `git@github.com:jamaldevsecops/ArgoCD.git` (or `*` to allow all repositories).
     - **Destinations**:
       - **Cluster**: `https://kubernetes.default.svc` (default in-cluster) or your cluster URL.
       - **Namespace**: `myapp`
     - **Roles** (optional): Add roles for team access (e.g., `read-only` or `admin`).
   - Click **Create**.

3. **Verify Project**:
   - Confirm the project appears under **Settings** > **Projects**.

### Using CLI
1. **Log in to ArgoCD CLI**:
   ```bash
   argocd login argocd.apsissolutions.com:443 --username admin
   ```
   Enter the admin password when prompted. Skip `--insecure` as TLS is configured.

2. **Create Project**:
   Create a project YAML file (`myapp-project.yaml`):
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: AppProject
   metadata:
     name: myapp-project
     namespace: argocd
   spec:
     description: Project for myapp deployments
     sourceRepos:
     - git@github.com:jamaldevsecops/ArgoCD.git
     destinations:
     - namespace: myapp
       server: https://kubernetes.default.svc
     clusterResourceWhitelist:
     - group: '*'
       kind: '*'
   ```

3. **Apply Project**:
   ```bash
   kubectl apply -f myapp-project.yaml
   ```

4. **Verify Project**:
   ```bash
   argocd proj list
   ```
   Expected output:
   ```
   NAME          DESCRIPTION                  DESTINATIONS                       SOURCES                        CLUSTER-RESOURCE-WHITELIST
   myapp-project Project for myapp deployments myapp,https://kubernetes.default.svc git@github.com:jamaldevsecops/ArgoCD.git *
   ```

## 2. Creating a New Application (Public and Private Git Repository)

This section covers creating an application linked to the repository `git@github.com:jamaldevsecops/ArgoCD.git` with manifests in `k8s-manifest/myapp/for-nginx-ingress-contoller`, deploying to the `myapp` namespace.

### For Private Git Repository
Since the provided URL (`git@github.com:jamaldevsecops/ArgoCD.git`) uses SSH, configure credentials for private repository access.

#### Configure SSH Key in ArgoCD
1. **Generate or Use SSH Key**:
   Ensure an SSH key pair exists:
   ```bash
   ssh-keygen -t rsa -b 4096 -C "argocd@apsissolutions.com" -f ~/.ssh/argocd_key
   ```
   Add the public key (`argocd_key.pub`) to the Git repository’s deploy keys (e.g., on GitHub).

2. **Add SSH Key to ArgoCD**:
   Create a secret for the SSH key:
   ```bash
   kubectl -n argocd create secret generic ssh-key-secret \
     --from-file=sshPrivateKey=/path/to/.ssh/argocd_key
   ```

3. **Register Repository**:
   ```bash
   argocd repo add git@github.com:jamaldevsecops/ArgoCD.git \
     --ssh-private-key-secret ssh-key-secret \
     --name myapp-repo
   ```

#### Using Web UI
1. **Create Application**:
   - In the Web UI at `https://argocd.apsissolutions.com`, go to **Applications** > **+ New App**.
   - Fill in the details:
     - **Application Name**: `myapp`
     - **Project**: `myapp-project`
     - **Sync Policy**: Select **Automatic** (or Manual for controlled sync).
     - **Repository URL**: `git@github.com:jamaldevsecops/ArgoCD.git`
     - **Revision**: `HEAD` (or specify a branch, e.g., `main`).
     - **Path**: `k8s-manifest/myapp/for-nginx-ingress-contoller`
     - **Cluster URL**: `https://kubernetes.default.svc`
     - **Namespace**: `myapp`
   - Click **Create**.

2. **Verify Application**:
   - Go to **Applications** and check that `myapp` appears with a status of **Healthy** and **Synced**.
   - Click the application to view resources (e.g., pods, services, ingress).

#### Using CLI
1. **Create Application**:
   Create an application YAML file (`myapp.yaml`):
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: myapp
     namespace: argocd
   spec:
     project: myapp-project
     source:
       repoURL: git@github.com:jamaldevsecops/ArgoCD.git
       targetRevision: HEAD
       path: k8s-manifest/myapp/for-nginx-ingress-contoller
     destination:
       server: https://kubernetes.default.svc
       namespace: myapp
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
   ```

2. **Apply Application**:
   ```bash
   kubectl apply -f myapp.yaml
   ```

3. **Verify Application**:
   ```bash
   argocd app get myapp
   ```
   Expected output:
   ```
   Name:               myapp
   Project:           myapp-project
   Server:            https://kubernetes.default.svc
   Namespace:         myapp
   URL:               https://argocd.apsissolutions.com/applications/myapp
   Repo:              git@github.com:jamaldevsecops/ArgoCD.git
   Target:            HEAD
   Path:              k8s-manifest/myapp/for-nginx-ingress-contoller
   SyncWindow:        Sync Allowed
   Sync Status:       Synced
   Health Status:     Healthy
   ```

4. **Sync Application (if not automatic)**:
   ```bash
   argocd app sync myapp
   ```

### For Public Git Repository
If the repository is public, use the HTTPS URL (e.g., `https://github.com/jamaldevsecops/ArgoCD.git`) without SSH credentials.

#### Using Web UI
1. Follow the same steps as for the private repository, but use the HTTPS URL in **Repository URL**.
2. No credential configuration is required for public repositories.

#### Using CLI
1. **Register Repository**:
   ```bash
   argocd repo add https://github.com/jamaldevsecops/ArgoCD.git --name myapp-repo
   ```

2. **Create Application**:
   Update `myapp.yaml` with the HTTPS URL:
   ```yaml
   spec:
     source:
       repoURL: https://github.com/jamaldevsecops/ArgoCD.git
       targetRevision: HEAD
       path: k8s-manifest/myapp/for-nginx-ingress-contoller
   ```

3. **Apply and Verify**:
   ```bash
   kubectl apply -f myapp.yaml
   argocd app get myapp
   ```

## Post-Creation Checks
1. **Verify Resources**:
   ```bash
   kubectl get all -n myapp
   ```
   Check for pods, services, and ingress resources deployed from the manifests.

2. **Check Application Status**:
   In the Web UI, ensure the application is **Synced** and **Healthy**.
   Via CLI:
   ```bash
   argocd app get myapp
   ```

3. **Access Application**:
   If the manifests include an Ingress resource, verify access via the configured domain (e.g., `http://myapp.apsissolutions.com`).

## Troubleshooting Tips
- **Sync Errors**: Check application events in the Web UI or via `argocd app events myapp` for issues with manifests or permissions.
- **Repository Access**: Ensure SSH key or HTTPS credentials are correctly configured (`argocd repo list`).
- **Namespace Issues**: Confirm the `myapp` namespace exists and manifests target it correctly.
- **Ingress Not Found**: Verify the NGINX Ingress Controller is running and the Ingress resource is valid (`kubectl get ingress -n myapp`).

## References
- ArgoCD Documentation: https://argo-cd.readthedocs.io/en/stable/
- GitOps with ArgoCD: https://argo-cd.readthedocs.io/en/stable/user-guide/projects/
- NGINX Ingress Controller: https://kubernetes.github.io/ingress-nginx/
