
## Prerequisites:
- [Docker](https://docs.docker.com/desktop/install/mac-install/)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Fx86-64%2Fstable%2Fbinary+download)
- [Helm](https://helm.sh/docs/intro/install/)

### Creating a multi-node Cluster in Minikube
```minikube start --nodes 2 -p multinode```
Verify that two nodes are created:
```
kubectl get nodes
NAME            STATUS   ROLES           AGE    VERSION
multinode       Ready    control-plane   ...    ....
multinode-m02   Ready    <none>          ...    ....
```

### Deploying the Datadog Agents
```
helm repo add datadog https://helm.datadoghq.com
helm repo update
kubectl create secret generic datadog-agent --from-literal api-key=<DATADOG_API_KEY>
helm install datadog -f values.yaml datadog/datadog
```
Verify that the pods have been deployed successfully:
```
kubectl get pod -o wide
NAME                                     READY   STATUS    RESTARTS   AGE   IP           NODE            NOMINATED NODE   READINESS GATES
datadog-8jbjv                            3/3     Running   0          95s   10.244.0.4   multinode       <none>           <none>
datadog-cluster-agent-5997688b67-k5hx7   1/1     Running   0          95s   10.244.1.5   multinode-m02   <none>           <none>
datadog-hkjs4                            3/3     Running   0          95s   10.244.1.4   multinode-m02   <none>           <none>
```

View resources created from helm deployment:
```
kubectl get daemonset        
NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
datadog   2         2         2       2            2           kubernetes.io/os=linux   6m45s

kubectl get deployment
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
datadog-cluster-agent   1/1     1            1           7m39s
```

### Deploying an Application
https://github.com/DataDog/apm-tutorial-java-host
Create a deidicated namespace for the application resources
```
kubectl create namespace notes
```
Deploy the notes application
```
kubectl apply -f notes.yaml
```
Verify that the pod has been deployed onto the notes namespace
```
kubectl get pod -n notes
kubectl describe pod -n notes
```
Making a request to the notes app:
```
kubectl exec -it <AGENT_POD> -- curl -X POST '<POD_IP>:8080/notes?desc=Note'
kubectl exec -it <AGENT_POD> -- curl <POD_IP>:8080/notes 
```


### One-Step Instrumentation
Uncomment the following lines in the values.yaml file:
```
  # apm:
  #   instrumentation:
  #     enabled: true
```
Then redeploy the agents:
```
helm upgrade datadog -f values.yaml datadog/datadog
```
And redeploy the application pod:
```
kubectl delete -f notes.yaml
kubectl apply -f notes.yaml
```
We can now generate some traffic on the notes app and begin collecting traces.
```
kubectl exec -it <AGENT_POD> -- curl -X POST '<POD_IP>:8080/notes?desc=Note'
kubectl exec -it <AGENT_POD> -- curl <POD_IP>:8080/notes 
```

### Cluster Check
Create a service that exposes the notes app on port 80
```
kubectl apply -f notes-service.yaml
```
We can now reach the notes app's endpoints using:
```
kubectl exec -it <AGENT_POD> -- curl notes-service.notes.svc.cluster.local/notes   
kubectl exec -it <AGENT_POD> -- curl -X POST 'notes-service.notes.svc.cluster.local/notes?desc=Note'
```

Now we can set up a cluster check to schedule an http_check which targets this endpoint. Uncomment our the following lines in the values.yaml file:
```
  # confd:
  #   http_check.yaml: |-
  #     cluster_check: true
  #     instances:
  #       - name: Notes
  #         url: http://notes-service.notes.svc.cluster.local/notes   
```
Then redeploy the agents:
```
helm upgrade datadog -f values.yaml datadog/datadog
```
This will schedule a cluster check that dispatches the http_check onto a single node agent. From there, the node agent will run the http_check against the service created(``notes-service.notes.svc.cluster.local/notes``)

### Teardown instructions
Delete Kubernetes resources created:
```
kubectl delete -f notes.yaml
kubectl delete -f notes-service.yaml
helm uninstall datadog
```
Delete minikube cluster:
```
minikube stop -p multinode
minikubt delete -p multinode
```
