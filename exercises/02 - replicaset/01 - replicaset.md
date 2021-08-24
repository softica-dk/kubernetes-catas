# Replicaset

In this exercise we will have a look at `replicaset` objects and how they differ from `pods`.

## What are we solving?
When we create a pod, there is no central controller that watches the state of it. A pod manifest is read by an assigned `kubelet` on a worker node. If the pod is deleted, the kubelet will remove it, and a new is not created.

So, if we want resilience, we need a controller that can recreate pods if they should be killed. 

The object we need, is a replicaset.

> In the old days, before replicasets, it was called a replication controller.

## Lets create a replicaset
Create a file called `my-replicaset.yaml` with the following content
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - name: my-web
        image: nginx
```
> Notice the labels. These are used to pair the replicaset and managed pods

Lets deploy it, and see what happens
```
kubectl apply -f my-replicaset.yaml
```
Lets use labels to wait for the pod to be ready. This way we don't get other pods in the output
```
kubectl get pods -w -l app=my-nginx
```
> Press Ctrl + c to exit the wait

Lets check the replicaset object
```
kubectl get replicaset
NAME       DESIRED   CURRENT   READY   AGE
rs-nginx   1         1         1       3m43s
```

## Check resilience
Lets try and delete a pod, and check what happens
```
kubectl get pods -l app=my-nginx
NAME             READY   STATUS    RESTARTS   AGE
rs-nginx-k9bxn   1/1     Running   0          3m48s

kubectl delete pod rs-nginx-k9bxn
```

Lets see what happened
```
kubectl get pods -l app=my-nginx
NAME             READY   STATUS    RESTARTS   AGE
rs-nginx-f84d7   1/1     Running   0          11s
```
We got a new pod

> Tip, if you want to see what node the pod is running on, try adding `-o wide`

```
kubectl get pods -l app=my-nginx -o wide
NAME             READY   STATUS    RESTARTS   AGE   IP          NODE              NOMINATED NODE   READINESS GATES
rs-nginx-f84d7   1/1     Running   0          49s   10.42.0.8   192.168.122.111   <none>           <none>
```

## Scaling
Let's say that one pod isn't enough to handle the load of our popular website. We would then like to scale the number of pods.

We can do that by telling the replicaset object a new desired state.

There a multiple ways to do this.

### Run a command
We can run this command to tell the replicaset that we now want 3 pods running at all times.
```
kubectl scale replicaset rs-nginx --replicas=3
```
Check the replicaset and number of running pods.


### Change the yaml
We can also edit the yaml file field `spec.replicas` and set it to `3` and apply the file
```
kubectl apply -f my-replicaset.yaml
```

Check the replicaset and number of running pods.

The great this about this approach is that we can keep track of all changes with `git`. By running commands, that's not possible.

## Clean up
Since we can't delete the pods, as the replicaset keeps recreating them, we now need to delete the replicaset object instead. When done, the replicaset and pods managed by the replicaset will be deleted.

```
kubectl delete replicaset rs-nginx
```

Check the replicaset and number of running pods.

> Note: You can't use apply on objects that hasn't been started by either `kubectl create --save-config` or `kubectl apply`