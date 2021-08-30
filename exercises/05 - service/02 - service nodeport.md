# Services

Here we will create a service of type `NodePort`.

## What are we solving?
Sometimes we dont have an `Ingress controller` and need to expose a service to the outside of the cluster. This can be done via the type `NodePort`. NodePort will expose a service on every Kubernetes worker node on a specific high port (default: 30000-32767). You can then put an external loadbalancer in front of the cluster proxying the worker nodes on that port.

> This was the way to expose services in Kubernetes before Ingress.

## Create a Deployment
Lets create a Multitool deployment to expose 

Since we are not going to use the deployment object, we will do it by command, and not via yaml. 

First we create a deployment called multitool, and then we patch it to insert the imagePullSecret.
```
kubectl create deployment multitool --replicas=3 --image=praqma/network-multitool
kubectl patch deployment multitool --patch '{"spec": {"template":{"spec":{"imagePullSecrets": [{"name": "regcred"}]}}}}'
```

> On windows use: `kubectl patch deployment multitool --patch '{\"spec\": {\"template\":{\"spec\":{\"imagePullSecrets\": [{\"name\": \"regcred\"}]}}}}'`

Wait for all pods to be ready 
```
kubectl wait --for=condition=available --timeout=600s deployment/multitool
```

We now need to create a service that exposes multitool on every worker node.

Create a file multitool-nodeport-service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: multitool
spec:
  type: NodePort
  selector:
    app: multitool
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply the service definition
```
kubectl apply -f multitool-nodeport-service.yaml
```

Lets describe our service
```
kubectl describe service multitool
Name:                     multitool
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=multitool
Type:                     NodePort
IP Families:              <none>
IP:                       10.43.8.144
IPs:                      <none>
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31821/TCP
Endpoints:                10.42.0.21:80,10.42.1.16:80,10.42.2.21:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

We can see that it is served on port `31821`. So lets open a browser on a random workernode on that port

```
kubectl get nodes
NAME              STATUS   ROLES                      AGE   VERSION
192.168.122.111   Ready    controlplane,etcd,worker   14d   v1.17.4
192.168.122.112   Ready    controlplane,etcd,worker   14d   v1.17.4
192.168.122.113   Ready    controlplane,etcd,worker   14d   v1.17.4

google-chrome 192.168.122.111:31821
```

## Clean up
Lets delete the deployment and service
```
kubectl delete deployment multitool
kubectl delete service multitool
```




