# ArgoCD Setup (correct flow)
* Git Repo - Source of truth
* ArgoCD - Reconciler/controller
* EKS	- Desired runtime state

* Install ArgoCD:
```
kubectl create namespace argocd
```
```
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
* CLI install:
```
curl -sSL -o argocd \
https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
```
```
sudo install -m 555 argocd /usr/local/bin/argocd
```
* Get password:
```
argocd admin initial-password -n argocd
```

# ArgoCD Application concept

* Everything in ArgoCD = Application

* Each app has:

Source → Git repo (Helm/YAML)
Destination → EKS cluster + namespace
Sync policy → manual or auto

* Example:

backend-app
  source: git repo
  path: helm/backend
  dest: cluster: EKS, namespace: backend

# Why ArgoCD is used
* Advantages (refined version):
* No need to run kubectl apply manually
* No Jenkins Kubernetes access needed (no kubeconfig in CI)
* Git is single source of truth (true GitOps)
* Auto-detect drift (manual changes in cluster get reverted)
* Easy rollback → revert Git commit
* UI visibility (live cluster vs desired state)
* Multi-env support (dev/qa/prod apps)


🚀 OPTION 1: Simple ArgoCD App (Raw Kubernetes YAML)

# 1. Prerequisites
* Make sure you have:
* Kubernetes cluster (EKS/minikube)
* kubectl configured
* ArgoCD installed
* ArgoCD UI accessible

* Check:
```
kubectl get pods -n argocd
```

# 2. Create Git Repo (GitOps Repo)

Example repo:
argocd-nginx-demo/

* Structure:
```
argocd-nginx-demo/
nginx/
   deployment.yaml
   service.yaml
argocd-app.yaml
```

# 3. Create NGINX Deployment YAML
* deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```
* service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```
# 4. Push to Git
```
git init
git add .
git commit -m "nginx deployment"
git remote add origin <your-repo-url>
git push origin main
```
# 5. Create ArgoCD Application
* argocd-app.yaml
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/argocd-nginx-demo.git
    targetRevision: main
    path: nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
# 6. Apply ArgoCD App
```
kubectl apply -f argocd-app.yaml -n argocd
```
# 7. Sync happens automatically
* If auto-sync is enabled:
* ArgoCD pulls Git repo
* Detects nginx manifests
* Deploys into cluster

# 8. Verify Deployment
```
kubectl get pods
```
```
kubectl get svc
```

* You should see:
nginx pods running
* LoadBalancer service with external IP

# 9. Access NGINX
* Copy external IP:
```
kubectl get svc nginx-service
```
* Open in browser:
```
http://<EXTERNAL-IP>
```

* You’ll see:
👉 “Welcome to nginx!”

```
Developer → Git (code repo)
        ↓
Jenkins (CI builds image)
        ↓
Docker registry (ECR)
        ↓
Git (CD repo updated with image tag)
        ↓
ArgoCD watches Git
        ↓
EKS cluster updated automatically
```