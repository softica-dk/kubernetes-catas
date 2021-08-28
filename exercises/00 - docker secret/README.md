# Docker hub secret

## Why
In order to pull docker images as a know user, and thereby get 200 pulls pr 6 hours, we need to create a secret with our credentials. This secret will then be used by our Kubernetes manifests throughout the course.

## Create the secret
Run this command
```
kubectl create secret docker-registry regcred --docker-server=https://index.docker.io/v1/ --docker-username=<your-user-name> --docker-password=<your-pword> --docker-email=<your-email>
```

Check that you have a secret called regcred by running
```
kubectl get secrets
```

## More info
Visit `https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/`