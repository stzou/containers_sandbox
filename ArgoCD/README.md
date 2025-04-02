
### ArogCD 
Argo CD (short for Argo Continuous Delivery) is a declarative GitOps tool for Kubernetes. It automates the deployment and management of Kubernetes resources by syncing them with what’s defined in a Git repository.

| Use Case                              | Why Argo CD Helps                                                 |
|---------------------------------------|-------------------------------------------------------------------|
| **CI/CD for Kubernetes** apps         | Argo = CD, CI can be GitHub Actions, GitLab, etc.                |
| Managing **multi-environment** setups | Dev, staging, prod — each tracked in Git                         |
| Visibility into running apps          | See every resource, image version, diff from Git                 |
| GitOps automation                     | No kubectl needed! Just push to Git and Argo CD does the rest    |

### Installing Argo CD
Create a new namespace named ``argocd`` where argocd resources will live:
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
### Installing the Argo CD CLI:
```
brew install argocd
```

### Accessing the Argo CD API server:
By default, the Argo CD API server is not exposed with an external IP. There are a few ways to expose the API server - for the sandbox, we will be port-fowarding:
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
The API server can be accessed using ``https://localhost:8080``

You can now log into the Argo CD server using a username and password.
The username will default to ``admin`` and the password can be retrieved using:
```
argocd admin initial-password -n argocd
W4Lt0uOJs1A6ACvN
```

You can either log in from the CLI or from the UI.
From the CLI:
```
argocd login localhost:8080 
```
From the UI:
Open https://localhost:8080

### Deploying From Github
**Datadog Agent**
From UI:
Settings -> Repositories -> Connect Repo
Type:``git``
Project:``default``
Repository URL: ``https://github.com/stzou/containers_sandbox.git``

Create a secret for the API key:
```
kubectl create secret generic datadog-secret --from-literal api-key=...
```
Use the following YAML file in the UI:
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: datadog-agent
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  source:
    path: helm-charts/charts/datadog
    repoURL: https://github.com/stzou/containers_sandbox.git
    targetRevision: HEAD
    helm:
      valueFiles:
        - values.yaml
      parameters:
        - name: datadog.apiKeyExistingSecret
          value: datadog-secret
        - name: datadog.kubelet.tlsVerify
          value: 'false'
        - name: datadog.logs.containerCollectAll
          value: 'true'
  sources: []
  project: default
```

**Guestbook**
From UI:
Settings -> Repositories -> Connect Repo
Type:``git``
Project:``default``
Repository URL: ``https://github.com/argoproj/argocd-example-apps.git``

Create New App -> Select Edit as Yaml:
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  source:
    path: helm-guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    helm:
      valueFiles:
        - values.yaml
  sources: []
  project: default

```

From: CLI:
Ensure that you're logged in using:
``argocd login localhost:8080``
```
kubectl config set-context --current --namespace=argocd
argocd app create guestbook --repo https://github.com/argoproj/argocd-example-apps.git --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace default
argocd app sync guestbook   
```

Accessing Application:
```
kubectl port-forward svc/test-app-helm-guestbook -n default 8081:80
```

### Monitoring ArgoCD
Argo CD exposes Prometheus-formatted metrics on three of their components. We will need to add autodiscovery annotations for each one.
- Application Controller
```
annotations:
    ad.datadoghq.com/argocd-application-controller.checks: |
      {
        "argocd": {
          "init_config": {},
          "instances": [
            {
              "app_controller_endpoint": "http://%%host%%:8082/metrics"
            }
          ]
        }
      }      
```
- API Server
```
annotations:
    ad.datadoghq.com/argocd-server.checks: |
        {
        "argocd": {
            "init_config": {},
            "instances": [
            {
                "api_server_endpoint": "http://%%host%%:8083/metrics"
            }
            ]
        }
        }   
```
- Repo Server
```
annotations:
    ad.datadoghq.com/argocd-repo-server.checks: |
        {
        "argocd": {
            "init_config": {},
            "instances": [
            {
                "repo_server_endpoint": "http://%%host%%:8084/metrics"
            }
            ]
        }
        }
```
The deployment files for each of these components can be found here:
https://github.com/argoproj/argo-cd/blob/master/manifests/install.yaml
These annotations have been added to the respective yaml files. Apply the annotations using:
```
kubectl apply -f application-controller.yaml -n argocd
kubectl apply -f argocd-repo-server.yaml -n argocd
kubectl apply -f argocd-server.yaml -n argocd
```

### Cleanup:
```
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete ns argocd
```

References:
https://christianhuth.de/deploying-helm-charts-through-argocd-streamlining-kubernetes-deployments/?utm_campaign=Publish%3A+Update+Succeeded&utm_content=Publish%3A+Update+Succeeded+v2&utm_medium=email_action&utm_source=customer.io

https://argo-cd.readthedocs.io/en/stable/getting_started/?_gl=1*12a791x*_ga*MTMxOTIzNDA2MS4xNjgyNTI3NjYw*_ga_5Z1VTPDL73*MTY4MjUyNzY1OS4xLjAuMTY4MjUyNzY2Ny4wLjAuMA..

https://medium.com/@mehmetodabashi/installing-argocd-on-minikube-and-deploying-a-test-application-caa68ec55fbf
