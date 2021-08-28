# Multi container pod
A pod can contain multiple containers running at the same time. This is useful for a number of reasons.

1. Delegating work to different containers
   * Running metric export from separate container
   * Running a sidecar container for networking
2. Running special health containers
3. Running a container that sync remote data 

## The multi container manifest
Here is an example where we have multiple containers in a pod.

```
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  imagePullSecrets:
  - name: regcred
  restartPolicy: Never
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: proxy
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: curl
    image: curlimages/curl
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "curl https://gist.githubusercontent.com/calebgrove/7260396/raw/81cb4d57caa09dd5c605987c770c814a2070853c/helloworld.html -o /pod-data/index.html"]
```

The second container `curl` will die when done, and the other will keep on living, and serve the html `curl` downloaded from the gist.

They both mount the volume `shared-data` where `curl` puts an `index.html` that the container `nginx` then serves. 

## Check the served html
Like the init container exercise, we can check that the `nginx` container serves the correct content.

Run the port-forward command
```
kubectl port-forward two-containers 8080:80
```
and open an browser and point it to `http://localhost:8080`.

## Cleanup
Lets remove the pod
```
kubectl delete pod two-containers
```