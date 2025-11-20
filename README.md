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
