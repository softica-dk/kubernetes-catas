# Namespace

## Create a new namespace
Namespaces are virtual cluster in which you can deploy workloads. To create a new namespace simply run the following command:
```
kubectl create namespace test
```
Now verify that the new namespace exists with this command:
```
kubectl get namespaces
```

A better way to define namespaces, is to define it in a manifest file like this:
```
apiVersion: v1
kind: Namespace
metadata:
  name: test2
```
Now create the namespace from the manifest:
```
kubectl create -f test2.yaml
```

Lets remove the two namespaces again
```
kubectl delete namespace test test2
```
## Deploy a simple pod in namespace
Lets define a simple pod running nginx:
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```
Now lets deploy the pod
```
kubectl create -f pod.yaml
```
Wait for the pod to be ready, as it needs to download the docker image and start it up
```
kubectl get pods
```
> Question: What namespace did the pod get deployed to?

Lets test that the pod is working
```
kubectl port-forward nginx 8080:80
```
Open a webbrowser and go to localhost:8080

You should see the nginx default page being displayed.


## cleanup
Lets cleanup, and remove the pod we created.
```
kubectl delete pod nginx
```
Check that the pod is removed
```
kubectl get pods
```

> Question: Why did the pod not get rescheduled ?