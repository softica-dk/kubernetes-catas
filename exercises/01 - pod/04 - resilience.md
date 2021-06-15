# Resilience

## Lets delete a pod

Lets create a pod with a simple container, and then delete it. 

Create a file `simple-web.yaml` with the following content
```
apiVersion: v1
kind: Pod
metadata:
  name: willidie
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

Deploy the pod
```
kubectl create -f simple-web.yaml
```

Wait for the pod to be read
```
kubectl get pods -w
NAME             READY   STATUS              RESTARTS   AGE
willidie         0/1     ContainerCreating   0          3s
willidie         1/1     Running             0          5s

```
> press Ctrl + c to break the wait

What happens when we delete the pod? Will Kubernetes create a new pod ?

```
kubectl delete pod willidie
```

Did Kubernetes create a new pod ?

Why not?