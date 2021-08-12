# Deployment

In this exercise we will have a look at `deployment` objects and how they differ from `pods` and `replicaset`.

## What are we solving?

The object we need, is a deployment.

> In the old days, before replicasets, rolling updates was done client side. The working group is now archived:
> https://github.com/kubernetes/community/tree/master/archive/wg-apply

## Lets create a deployment
Create a file called `my-deployment.yaml` with the following content
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dp-nginx
  labels:
    app: nginx
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
        image: nginx:1.20.1-alpine
        ports:
        - containerPort: 80
```
> Notice the labels. These are used to pair the replicaset created by the deployment, and managed pods.

Lets deploy it, and see what happens
```
kubectl apply -f my-deployment.yaml --record
```
Lets use labels to wait for the pod to be ready. This way we don't get other pods in the output
```
kubectl get pods -w -l app=my-nginx
```
> Press Ctrl + c to exit the wait

Lets check the deployment object
```
kubectl get deployment
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
dp-nginx   1/1     1            1           19s
```

## Update content
Lets try and update the image to a newer version

Change the image version from `nginx:1.20.1-alpine` to `nginx:1.21.1-alpine` in the yaml file.

Now apply the new manifest with the updated image
```
kubectl apply -f my-deployment.yaml --record
```

Check the rollout status
```
kubectl rollout status deployment dp-nginx
deployment "dp-nginx" successfully rolled out
```

Now we can check the `replicasets`
```
kubectl get rs
NAME                 DESIRED   CURRENT   READY   AGE
dp-nginx-d78cc966c   1         1         1       119s
dp-nginx-f64fcfc46   0         0         0       3m2s
```
We can see that it keeps the old one. Let rollback

Let's check the history
```
kubectl rollout history deployment dp-nginx
deployment.apps/dp-nginx 
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=my-deployment.yaml --record=true
2         kubectl apply --filename=my-deployment.yaml --record=true
```

Check the details of our last update
```
kubectl rollout history deployment dp-nginx --revision=2
deployment.apps/dp-nginx with revision #2
Pod Template:
  Labels:	app=my-nginx
	pod-template-hash=d78cc966c
  Annotations:	kubernetes.io/change-cause: kubectl apply --filename=my-deployment.yaml --record=true
  Containers:
   my-web:
    Image:	nginx:1.21.1-alpine
    Port:	80/TCP
    Host Port:	0/TCP
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```

Let's rollback to revision #1 by running
```
kubectl rollout undo deployment dp-nginx --to-revision=1
deployment.apps/dp-nginx rolled back
```

> The `--to-revision` flag is optional. If not supplied, it will rollback on step.

We can now see that we are back using the first replicaset, and the new one is still kept
```
kubectl get rs
NAME                 DESIRED   CURRENT   READY   AGE
dp-nginx-d78cc966c   0         0         0       7m3s
dp-nginx-f64fcfc46   1         1         1       8m6s
```

If we run `kubectl describe deployment dp-nginx` we can see that the image used is `nginx:1.20.1-alpine`. 

You can specify `.spec.revisionHistoryLimit` to limit the number of replicas that the system keeps in history.

## Scaling
Scaling is done in the same way as `replicaset`, just use `deployment`as keyword
```
kubectl scale deployment dp-nginx --replicas=3
```
Or by changing the yaml manifest and apply it again.

# Update strategy
The field `.spec.strategy.type` can be either `Recreate` or `RollingUpdate`. Recreate will destroy all pods before creating new ones, where RollingUpdate will update pod in a rolling update fashion. 

You can specify `.spec.strategy.rollingUpdate.maxUnavailable` and `.spec.strategy.rollingUpdate.maxSurge`. maxUnavailable indicates the number of Pods that can be unavailable during the update process either in number or percentage. 

`maxSurge` defines how many Pods can be created above the desired stated value. This can also be given in a number or percentage.

## Clean up
Since we can't delete the pods, as the replicasets keeps recreating them, we now need to delete the deployment object instead. When done, the deployment, replicaset and pods managed by the replicasets will be deleted.

```
kubectl delete deployment dp-nginx
```

Check the deployment, replicaset and number of running pods.
