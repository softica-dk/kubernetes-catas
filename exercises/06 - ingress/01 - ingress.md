# Services

In this exercise we will have a look at `ingress` and how they are used by `ingress controllers`.

> For this exercise you need to install an ingress controller. They come in many flavors and ways of installing, so either follow this guide or use one you prefer. https://kubernetes.github.io/ingress-nginx/deploy/

## What are we solving?
- We have running pods
- We have replicasets that keeps things up
- We have deployments that help us update pods
- We have a service that exposes a fixed endpoint for out pods

We now need to expose our service and tie it to a dns name. This is what we will do with an ingress object.

## Lets prepare some pods and service to expose
We create a deployment of multitool and expose it with a service. We do it with commands to make it easier.

```
kubectl create deployment multitool --image=praqma/network-multitool
kubectl expose deployment multitool --type=ClusterIP --name=multitool --port=80 --target-port=80
kubectl wait --for=condition=available --timeout=600s deployment/multitool
```

## Lets create an ingress
Lets create an ingress object for our multitool service

> Note: The ingress object is now out of beta, so depending on your Kubernetes version, you need to adjust your yaml. Find you version via `kubectl version`.


If you are running on Kubernetes v1.19 or newer use:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multitool
  annotations:
    ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - http:
      paths:
        - path: /
          pathType: ImplementationSpecific
          backend:
            service:
              name: multitool
              port:
                number: 80
```

If you use an older version use this:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: multitool
  annotations:
    ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - http:
      paths:
        - path: /
          backend:
            serviceName: multitool
            servicePort: 80
```

Apply the ingress
```
kubectl apply -f ingress.yaml
ingress.extensions/multitool created
```

Check that the ingress is created
```
kubectl get ingress
NAME        HOSTS   ADDRESS   PORTS   AGE
multitool   *                 80      18s
```

As we didn't specify a host, this ingress will listen to all ingoing requests. We now need to find our ingress controller. In my case, i run `nginx` as a `daemonset` on all nodes with a hostip. This way, the pod running on each host can be reached on every kubernetes worker node. Nginx will listen for `ingress` objects and route accordingly. Lets try it out.

> A daemonset is a controller like deployments. But instead of having a replicaset to control the number of pods to run, it will run the specified pod on all nodes. We wont go through this controller, but you can learn more here : https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/

First, find your worker nodes ip
```
kubectl get nodes
NAME              STATUS   ROLES                      AGE   VERSION
192.168.122.111   Ready    controlplane,etcd,worker   13d   v1.17.4
192.168.122.112   Ready    controlplane,etcd,worker   13d   v1.17.4
192.168.122.113   Ready    controlplane,etcd,worker   13d   v1.17.4
```

Open a browser and go to one of the ip's. It should give you a warning about self signed certificates `Your connection is not private`. In chrome, click `Advanced`and then `"Proceed to [IP]`. You should see the multitool output.

## Adding host names
Right now all traffic is routed to our service from the ingress. Lets changed it to only route `www.multitool.com` by editing the ingress manifest file so that it looks like this

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: multitool
  annotations:
    ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: "multitool.com"
    http:
      paths:
        - path: /
          backend:
            serviceName: multitool
            servicePort: 80
```
Apply the changes
```
kubectl apply -f /tmp/ingress.old.yaml
ingress.extensions/multitool configured
```

And check the ingress
```
kubectl get ingress
NAME        HOSTS           ADDRESS                       PORTS   AGE
multitool   multitool.com   192.168.122.111,192.168....   80      13m
```
> Try refreshing your browser with just the ip.

You should get a `default backend - 404`.

Edit your /etc/hosts file (`c:\windows\system32\drivers\etc\hosts` on Windows)

> A guide : https://www.howtogeek.com/howto/27350/beginner-geek-how-to-edit-your-hosts-file/

Add the following to the bottom of the file and then save it.
```
192.168.122.111 multitool.com
```

Open `multitool.com` in a browser, and you should see the multitool output. In a production environment we would not edit hosts file but use a proper DNS server to link `multitool.com` to the ip of the `ingress controller`. 

## Cleanup
Remove the deployment, service and ingress
```
kubectl delete deployment multitool
kubectl delete service multitool
kubectl delete ingress multitool

deployment.apps "multitool" deleted
service "multitool" deleted
ingress.extensions "multitool" deleted
```
