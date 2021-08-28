# Services

In this exercise we will have a look at `service` and `endpoint` objects and how they relates.

## What are we solving?
When deploying pods they get a new `ip` everytime they are created. This makes it hard, if not impossible, to route traffic to them. The `ip` they get is a pod ip. We can create a service object that will get a service ip which is stable. A service also gets a cluster `dns` we can use.

## Lets create a deployment
We create a deployment that creates 3 replicas of network-multitool. This will help us see the traffic being spread out as it displays its name in the html output.

Since we are not going to use the deployment object, we will do it by command, and not via yaml:
```
kubectl create deployment multitool --image=praqma/network-multitool --replicas=3 --overrides='{"apiVersion": "apps/v1", 
"spec": {"template":{"spec":{"imagePullSecrets": [{"name": "regcred"}]}}}}'
```

Now wait for the deployment to be ready
```
kubectl wait --for=condition=available --timeout=600s deployment/multitool
```

We can find the pod ip of our pods by adding `-o wide` to our command
```
kubectl get pods -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP           NODE              NOMINATED NODE   READINESS GATES
multitool-7885b5f94f-d27kg   1/1     Running   0          2m21s   10.42.2.15   192.168.122.113   <none>           <none>
multitool-7885b5f94f-f9bng   1/1     Running   0          2m21s   10.42.0.17   192.168.122.112   <none>           <none>
multitool-7885b5f94f-gwlzg   1/1     Running   0          2m21s   10.42.1.13   192.168.122.111   <none>           <none>

```
In my case its `10.42.2.15`, `10.42.0.17` and `10.42.1.13`.

## Creating a service and endpoints
Here we will create a service that will loadbalance traffic between our three pods.

The way it will know what pods to route traffic to is controlled with labels by key/value.

First we need to get the label created by our kubectl command so that we can link it to our service

```
kubectl describe pod multitool-7885b5f94f-d27kg
Name:         multitool-7885b5f94f-d27kg
Namespace:    default
Priority:     0
Node:         192.168.122.113/192.168.122.113
Start Time:   Tue, 17 Aug 2021 18:40:53 +0200
Labels:       app=multitool
              pod-template-hash=7885b5f94f
Annotations:  <none>
Status:       Running
IP:           10.42.2.15
...
```

It shows to labes, one `app=multitool` and another with `pod-template-hash=7885b5f94f`. We can use the first to link our service to the pods, as all pods in the deployment has this label.

> If you create a real deployment yaml file, you can specify the label key/values yourself


Lets create the service yaml file
```
apiVersion: v1
kind: Service
metadata:
  name: multitool
spec:
  selector:
    app: multitool
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

In the field `.spec.selector` we can see our key/value label. This binds this service to all pods in this namespace that has this label.

In `.spec.ports` we define the port this service should be listening on, and the target port of the pods. 

Lets deploy it

```
kubectl apply -f multitool-service.yaml
service/multitool created
```

We can see our new service via
```
kubectl get services 
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.43.0.1      <none>        443/TCP   10d
multitool    ClusterIP   10.43.193.59   <none>        80/TCP    58s
```

The first service is actually the `api-server` that is exposed inside the cluster. The second is our new service.

As we can see, our service has a `cluster-ip` (10.43.193.59) and listens on ports 80.

We can describe the service to get more information
```
kubectl describe service multitool 
Name:              multitool
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=multitool
Type:              ClusterIP
IP Families:       <none>
IP:                10.43.193.59
IPs:               <none>
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.42.0.17:80,10.42.1.13:80,10.42.2.15:80
Session Affinity:  None
Events:            <none>
```

> Note the Type is set to `ClusterIP`. We will cover other types later on.

The important information here is the `Endpoints` field showing us the three pods IP's. Just as expected.

## Testing the service
We can use our multitool pods to test the service.
Lets exec into one of them and check it out.

Find the name of one of our pods
```
kubectl get pods -l app=multitool
NAME                         READY   STATUS    RESTARTS   AGE
multitool-7885b5f94f-d27kg   1/1     Running   0          16m
multitool-7885b5f94f-f9bng   1/1     Running   0          16m
multitool-7885b5f94f-gwlzg   1/1     Running   0          16m
```

Then exec into one of them
```
kubectl exec -it multitool-7885b5f94f-f9bng -- /bin/sh
/ #
```

Now we can test our service with `curl` by connecting to the service ip on port 80 and hopefully see the multitool output
```
/ # curl 10.43.193.59:80
Praqma Network MultiTool (with NGINX) - multitool-7885b5f94f-f9bng - 10.42.0.17
```

Try running the command a couple of times
```
/ # curl 10.43.193.59:80
Praqma Network MultiTool (with NGINX) - multitool-7885b5f94f-f9bng - 10.42.0.17
/ # curl 10.43.193.59:80
Praqma Network MultiTool (with NGINX) - multitool-7885b5f94f-d27kg - 10.42.2.15
/ # curl 10.43.193.59:80
Praqma Network MultiTool (with NGINX) - multitool-7885b5f94f-gwlzg - 10.42.1.13
/ # curl 10.43.193.59:80
Praqma Network MultiTool (with NGINX) - multitool-7885b5f94f-gwlzg - 10.42.1.13
/ # curl 10.43.193.59:80
Praqma Network MultiTool (with NGINX) - multitool-7885b5f94f-f9bng - 10.42.0.17
```

We can see that the traffic is spread out to our three pods.

Lets test the cluster dns
```
curl multitool:80
Praqma Network MultiTool (with NGINX) - multitool-7885b5f94f-gwlzg - 10.42.1.13
```

When only using the name of the service, you will resolve names in your current namespace (in this case `default`).

If you want to connect to a service in another namespace, do the following
```
/ # curl multitool.default.svc.cluster.local:80
Praqma Network MultiTool (with NGINX) - multitool-7885b5f94f-f9bng - 10.42.0.17
```

Exit the container with `Ctrl + d`.

### Scaling
Lets scale our deployment, and see what our service does.

```
kubectl scale deployment multitool --replicas=5
kubectl wait --for=condition=available --timeout=600s deployment/multitool
```

Lets check the service
```
kubectl describe service multitool
Name:              multitool
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=multitool
Type:              ClusterIP
IP Families:       <none>
IP:                10.43.193.59
IPs:               <none>
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.42.0.17:80,10.42.0.18:80,10.42.1.13:80 + 2 more...
Session Affinity:  None
Events:            <none>
```

We can see that our service now has 3+2 endpoints. Lets find them all

```
kubectl get endpoints
NAME         ENDPOINTS                                                        AGE
kubernetes   192.168.122.111:6443,192.168.122.112:6443,192.168.122.113:6443   10d
multitool    10.42.0.17:80,10.42.0.18:80,10.42.1.13:80 + 2 more...            16m
```

We can see that the endpoint object is called multitool. Lets describe it.
```
kubectl describe endpoints multitool
Name:         multitool
Namespace:    default
Labels:       <none>
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2021-08-17T17:07:13Z
Subsets:
  Addresses:          10.42.0.17,10.42.0.18,10.42.1.13,10.42.2.15,10.42.2.16
  NotReadyAddresses:  <none>
  Ports:
    Name     Port  Protocol
    ----     ----  --------
    <unset>  80    TCP

Events:  <none>
```

## Cleanup
Lets delete our service and deployment.

```
kubectl delete deployment multitool
deployment.apps "multitool" deleted

kubectl delete service multitool
service "multitool" deleted
```

> Note: When you delete a service, it will delete its endpoint object as well