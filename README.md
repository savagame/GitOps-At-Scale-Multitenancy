# GitOps at Scale and Multitenancy

## App of App

A single Argo CD application corresponds to just one specific web application.

```
spec:
  project: default
  source:
    repoURL: https://github.com/savagame/GitOps-At-Scale-Multitenancy.git
    targetRevision: main
    path: ./app-of-app/simple-webapp
    directory:
      recurse: false
  destination:
    server: https://kubernetes.default.svc
    namespace: app-of-app
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    automated:
      prune: true
      selfHeal: true
```

Here, path is set to directly point to the specific application.

## App of Apps Approach

Rather than directing toward a singular application manifest, as it did previously, the root app now references a specific folder within a Git repository. This folder contains all the individual application manifests that define and facilitate the creation and deployment of each application.

```
spec:
  project: default
  source:
    repoURL: https://github.com/savagame/GitOps-At-Scale-Multitenancy.git
    targetRevision: main
    path: ./app-of-apps/simple-webapps
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: app-of-apps
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    automated:
      prune: true
      selfHeal: true
```

The path attribute instructs Argo CD to target a specific directory – in this case, named simple-webapps – located within the repository. This directory contains Kubernetes manifests that define the applications, as well as supporting various formats such as Helm, Kustomize.
In the provided configuration, there are two notable attributes worth highlighting: selfHeal: true and directory.recurse: true. The selfHeal feature ensures automatic updates of the child applications in response to any changes detected, maintaining consistent deployment states. Additionally, the recurse setting enables the iteration through the webapps folders, facilitating the deployment of all applications contained within.

## Effective Git repository strategies

### Environment per branches

The environment-per-branch approach in GitOps, which involves using branches to represent different environments such as staging or production, is often considered an anti-pattern.
I created 3 branches for this strategy: qa, staging, prod

```
⚠️ This Is Where Problems Happen (READ)

You will get:

merge conflicts

environment drift

broken pipelines

config duplication

cherry-pick hell

accidental prod changes

YAMLs diverging over time
```

### Environment per folders

Each folder represents a specific environment such as development, staging, or production. This structure allows for clear separation and management of configurations for each environment, facilitating easier updates and maintenance.

See Everything in Argo CD UI

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Promote QA → Stage?

```
cp overlays/qa/patch.yaml overlays/stage/patch.yaml
git add .
git commit -m "Promote QA version to Stage"
git push
```

Promote Stage → Prod?

```
cp overlays/stage/patch.yaml overlays/prod/patch.yaml
git commit -am "Promote Stage to Prod"
git push
```

### Native Multinenancy with ArgoCD

Step 1 – Install Argo CD core

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Step 2 – Create tenant namespace (devteam-a)

```
kubectl create namespace devteam-a
```

Step 3 – Create the AppProject for devteam-a

```
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: devteam-a
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Enable DevTeam-A Project to deploy new applications
  sourceRepos:
    - "*"
  destinations:
    - namespace: "devteam-a"
      server: https://kubernetes.default.svc

  clusterResourceBlacklist:
    - group: ""
      kind: "Namespace"

  namespaceResourceBlacklist:
    - group: "argoproj.io"
      kind: "AppProject"
    - group: "argoproj.io"
      kind: "Application"
    - group: ""
      kind: "ResourceQuota"
    - group: "networking.k8s.io"
      kind: "NetworkPolicy"
```

Apply:

```
kubectl apply -f argocd-project-devteam-a.yaml
```

Step 4 – Prepare a Git repo for devteam-a

```
git clone <your-repo-url> devteam-a-application
cd devteam-a-application
mkdir -p applicationset
mkdir -p applicationset/nginx-app
```

Create a very simple nginx deployment + service:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
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
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
```

Commit & push:

```
git add .
git commit -m "Initial nginx app for devteam-a"
git push
```

Step 5 – Create the “initializer” Application for devteam-a

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: application-initializer-devteam-a
  namespace: argocd
spec:
  project: devteam-a
  source:
    repoURL: <YOUR-REPO-URL>
    targetRevision: main
    path: ./applicationset
  destination:
    server: https://kubernetes.default.svc
    namespace: devteam-a
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

Apply:

```
kubectl apply -f application-initializer-devteam-a.yaml
```

Step 6 – Verify in Argo CD

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Step 7 – Confirm the app is running in devteam-a

```
kubectl get pods -n devteam-a
kubectl get svc -n devteam-a
```

Step 8 – See the multitenancy constraints in action

```
8.1 Try to create a Namespace (should be blocked)
Add this to applicationset/nginx-app/namespace.yaml:

apiVersion: v1
kind: Namespace
metadata:
  name: i-am-not-allowed-here

Commit & push.
Argo CD will try to sync, and you should see an error because:

clusterResourceBlacklist:
  - group: ""
    kind: "Namespace"

That’s exactly the kind of protection you read about.
```

```
8.2 Try to create another Application (should be blocked)
In the same repo, add:

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: devteam-trying-to-cheat
spec: {}

Push again.
Sync should fail for that resource because Application is blacklisted in namespaceResourceBlacklist.

This proves the AppProject restrictions are working.
```

Step 9 – Scaling the pattern (more teams)

```
To add a new team devteam-b in the same “core Argo CD” model:

Create new namespace devteam-b

Create AppProject devteam-b with its own blacklist and dest namespace

Create Application application-initializer-devteam-b pointing to their own repo
```
