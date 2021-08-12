# Statefulset

In this exercise we will have a look at `statefulset` objects and how they differ from `deployments`.

## What are we solving?

Statefulsets are special. They don't create `replicasets` but control pods directly.

Some applications needs to keep their identity, eg. if you have three pods running that registers themselves and join a cluster. If a new instance joins, its treated as a new member, even if it in reality is an old pod rejoining. To fix this, `statefulsets` was create (mostly by RedHat). If pod-2 dies, the new pod that replaces it will also be called pod-2, replacing the old pods identity.

Also scaling is done in order, and one at the time.

## Lets create a Statefulset
Create a file called `my-statefulset.yaml` with the following content
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
```
> Notice the labels. These are used to pair the pods created by the statefulset.

Lets deploy it, and see what happens
```
kubectl apply -f my-statefulset.yaml 
```
Lets use labels to wait for the pod to be ready. This way we don't get other pods in the output
```
kubectl get pods -w -l app=nginx
```
> Press Ctrl + c to exit the wait

Lets check the statefulset object
```
kubectl get statefulset 
NAME   READY   AGE
web    1/1     95s
```

As we can see, the pod doesn't have a random part of its name. 
```
kubectl get pods  -l app=nginx 
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          2m12s
```

## Scaling
Lets scale our statefulset and see the naming of pods unfold
```
kubectl scale statefulset web --replicas=4
statefulset.apps/web scaled
```

We can see that every pod has an orderly name
```
kubectl get pods -w  -l app=nginx
NAME    READY   STATUS              RESTARTS   AGE
web-0   1/1     Running             0          4m7s
web-1   0/1     ContainerCreating   0          18s
web-1   1/1     Running             0          22s
web-2   0/1     Pending             0          0s
web-2   0/1     Pending             0          0s
web-2   0/1     ContainerCreating   0          0s
web-2   1/1     Running             0          22s
web-3   0/1     Pending             0          0s
web-3   0/1     Pending             0          0s
web-3   0/1     ContainerCreating   0          0s
web-3   1/1     Running             0          2s
```

## Deleting a pod
Lets see what happens when we delete `web-2`.
```
kubectl delete pod web-2
```

In another terminal we can monitor what happens while it deletes `web-2` by running
```
kubectl get pods -w  -l app=nginx 
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          6m31s
web-1   1/1     Running   0          2m42s
web-2   1/1     Running   0          26s
web-3   1/1     Running   0          118s
web-2   1/1     Terminating   0          28s
web-2   0/1     Terminating   0          29s
web-2   0/1     Terminating   0          40s
web-2   0/1     Terminating   0          40s
web-2   0/1     Pending       0          0s
web-2   0/1     Pending       0          0s
web-2   0/1     ContainerCreating   0          0s
web-2   1/1     Running             0          2s
```

The pod replacing `web-2` has the same name. Awesome!


# Update strategy
The field `.spec.updateStrategy.type` the statefulset will update the desired state but not update any pods. It will only update pods when they are deleted. This way you can patch the state, and then choose the order and speed of replacing pods with the new version. 

RollingUpdate will update pod in a rolling update fashion. 

## No replicaset ?
We can check that it doesn't use replicasets by looking
```
kubectl get replicaset
No resources found in default namespace.
```

## Clean up
Since we can't delete the pods, as the statefulset keeps recreating them, we now need to delete the statefulset object instead. 

```
kubectl delete statefulset web
```