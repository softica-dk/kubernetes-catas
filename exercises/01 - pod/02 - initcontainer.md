# Init container

When running a pod, we might want to do some inital work before starting the main container in a pod. 

It would be to fetch some data, manipulate configuration files, register the app or alike.

In this exercise we will create a simple init container in a pod, that writes a simple html file to a volume. This volume is shared with the main container that will then serve the content through nginx.

## The pod manifest
First we create a file with the manifest defining out pod
```
apiVersion: v1
kind: Pod
metadata:
  name: icpod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    volumeMounts:
    - name: htmldir
      mountPath: /usr/share/nginx/html
  initContainers:
  - name: busybox
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "echo '<html><h1>Hello world</h1><html>' >> /html-dir/index.html"]
    volumeMounts:
    - name: htmldir
      mountPath: "/html-dir"
  volumes:
  - name: htmldir
    emptyDir: {}
```

Check that the pod is initialized
```
kubectl get pods
NAME    READY   STATUS     RESTARTS   AGE
icpod   0/1     Init:0/1   0          5s
```
wait and look again
```
kubectl get pods
NAME    READY   STATUS            RESTARTS   AGE
icpod   0/1     PodInitializing   0          7s
```
Wait even more, and the pod should get ready
```
kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
icpod   1/1     Running   0          25s
```

Lets check that nginx is serving our content, produced by the init container
```
kubectl port-forward icpod 8080:80
```
Now use curl or a browser to check the output
```
curl localhost:8080
<html><h1>Hello world</h1><html>
```
## Cleanup
Lets cleanup by removing the pod
```
kubectl delete pod icpod
```