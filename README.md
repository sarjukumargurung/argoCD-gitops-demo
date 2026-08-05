# argoCD-gitops-demo

## 1. Create Local Kind Cluster

Install prerequisites on macOS using Homebrew: 

```bash
brew install kind kubectl docker
```

Create a configuration file named kind-config.yaml to map ports if needed, or simply run a standard cluster:

```bash
kind create cluster --name argocd-demo
```

## 2.Install Argo CD

```bash
kubectl create namespace argocd
```
Apply the stable installation manifest

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## 3. Access Argo CD Dashboard

Retrieve the initial auto-generated admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
Forward the port to access the UI locally

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open https://localhost:8080 in your browser, log in with username admin and the password retrieved above

## 4. Run a Demo Build/Sync Job

```bash
kubectl apply -n argocd -f https://githubusercontent.com
```

## 5. Deploy an example guestbook app to verify that Argo CD successfully pulls and reconciles states inside your local Kind environment.

Save this as guestbook-app.yaml and run kubectl apply -f guestbook-app.yaml:

```bash
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 6. Verify and Access the Running App

Once created, Argo CD will immediately pull the deployment manifests, change from OutOfSync to Synced, and show healthy green icons for your resources.To verify the app is running locally on your machine and access it, expose the guestbook service using kubectl

```bash
kubectl port-forward svc/guestbook-ui 8090:80
```

Open http://localhost:8090 in your web browser to interact with the running guestbook web interface.

## 7.Manually deleting a pod or changing a replica count with kubectl to watch Argo CD instantly self-heal it

Simulating a configuration drift is the best way to see Argo CD's self-healing capabilities in action. Because we enabled selfHeal: true in your guestbook configuration, Argo CD will instantly detect any manual tampering and overwrite it to match the Git repository.

Experiment 1: Changing the Replica Count

In your first terminal, run the watch command to monitor the pods:

```bash
kubectl get pods -w
```
In your second terminal, manually force a scale-up:

```bash
kubectl scale deployment guestbook-ui --replicas=5
```
What happens: In your first terminal, you will see new pods briefly try to spin up (Pending or ContainerCreating), but they will immediately change to Terminating.

Why: Argo CD detects that the live cluster state (5 replicas) does not match the desired Git state (1 replica). It instantly issues a corrective command to scale it right back down to 1.

Experiment 2: Deleting a Kubernetes ServiceKubernetes itself self-heals pods if they die, but it will not automatically recreate a deleted Service, ConfigMap, or Deployment. Argo CD will.

Delete the application's network service entirely

```bash
kubectl delete svc guestbook-ui
```
Immediately check if the service is gone

```bash
kubectl get svc guestbook-ui
```

What happens: The service is still there, or it was recreated so fast that its AGE column reset to just a few seconds.Why: Deleting a service breaks the system configuration. Argo CD spots the missing resource during its quick reconciliation loop and automatically recreates it using the definition stored in the remote Git repo.

If you look at your Argo CD Web UI dashboard (https://localhost:8080) during either of these events, you will briefly see the application status icon flash to a yellow OutOfSync state before instantly snapping back to a healthy green Synced state.

## 8.Want to see how to organize your own repositories using Kustomize overlays (dev/staging/prod)

1. The Directory StructureOrganize your GitOps repository so that your core files live in base, while environment-specific tweaks live in overlays

```bash
your-gitops-repo/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── replicas.yaml
    │   └── kustomization.yaml
    ├── staging/
    │   ├── replicas.yaml
    │   └── kustomization.yaml
    └── prod/
        ├── replicas.yaml
        └── kustomization.yaml
```

2. Setting Up the BaseThe base directory holds your standard, default configuration

base/kustomization.yaml

```bash
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

3. Setting Up the Overlays

Each environment folder inside overlays/ references the base and applies patches or modifiers over it.Dev EnvironmentIn development, we want a single replica and a specialized name prefix (dev-).

overlays/dev/replicas.yaml

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app # Must match the base deployment name
spec:
  replicas: 1
```

overlays/dev/kustomization.yaml

```bash
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namePrefix: dev-
resources:
  - ../../base
patches:
  - path: replicas.yaml
```

Prod Environment In production, we need a high-availability setup with 5 replicas and a prod- prefix.

overlays/prod/replicas.yaml

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
spec:
  replicas: 5

```
overlays/prod/kustomization.yaml

```bash
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namePrefix: prod-
resources:
  - ../../base
patches:
  - path: replicas.yaml
```
4. Linking to Argo CD
 
Instead of pointing Argo CD directly to individual YAML manifests, point your Argo CD applications to the respective environment directory under overlays/.

For your production cluster environment, your argocd-app.yaml file would look like this:

```bash
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com'
    targetRevision: HEAD
    path: overlays/prod # Argo CD detects kustomization.yaml here and applies it
  destination:
    server: 'https://default.svc'
    namespace: prod-space
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    createNamespace: true
```

When Argo CD reads overlays/prod, it compiles the configurations. The deployment will deploy as prod-demo-app with 5 running replicas, whereas pointing an application to overlays/dev will generate dev-demo-app with only 1 replica.

