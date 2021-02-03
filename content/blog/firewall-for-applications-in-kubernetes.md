---
title: "Firewall for Applications in Kubernetes"
date: 2021-02-03T23:18:53+05:30
---

In this blog, we will look at a kubernetes feature which is intended to improve security for applications running in a cluster

In a kubernetes cluster, we can run many applications with multiple replicas for each application. By default, any pods can talk to any other pods running in the same cluster. 

<!--more-->

![](/img/network-policy-default.jpg)


But it is not recommended to allow such feature to be in place. In case an intruder get access to a pod, then he/she can access all pods from inside that compromised pod. So we need a firewall for applications using which it can decide if the traffic(both ingress and egress) should be allowed or denied inside the cluster.

![](/img/network-policy-firewall.jpg)

There comes the savior, `Network Policy` that helps to create a firewall for applications running in kubernetes cluster.

Lets understand the need for such firewall and how Network Policy helps for the same with some examples.

### Guestbook
Consider an application `Guestbook` having 3 different components as follows:
- guestbook-ui (frontend)
- guestbook-api (backend)
- guestbook-db (db)

Expected communication between these components are like ui communicates to api and api communicates to db.

![](/img/network-policy-guestbook1.jpg)

But when all these components of guestbook application are running in a kubernetes cluster, technically ui component can communicate to db by default.

![](/img/network-policy-guestbook2.jpg)


### Use Network Policy to setup firewall for applications in cluster

> As network policy is a feature that should be implemented by the network plugin, ensure that your network plugin supports NetworkPolicy resource. Creating NetworkPolicy resource without such plugin will have no effect. Calico, Cilium, Kube-router, Romana and Weave Net are some of the network plugins that support network policy

As a first step, disable the default behavior of allowing all communications between all pods running in a kubernetes cluster by creating a network policy as follows in all namespaces.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  ingress: {}
```

Now, none of the pods can communicate to any other pods running in the kubernetes cluster. Now based on requirements, allow the ingress/egress traffic for pods in the cluster.

For the guestbook application, to allow ingress traffic to `guestbook-api` pods but only from `guestbook-ui` pods, create a network policy as follows.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ui-to-api
spec:
  podSelector: 
    matchLabels: 
      app: guestbook-api
      tier: backend   
  ingress:
  - from:
      podSelector:
        matchLabels:
          app: guestbook-ui
          tier: frontend
```

And to allow ingress traffic to `guestbook-db` pods but only from `guestbook-api` pods, create a network policy as follows. 

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
spec:
  podSelector: 
    matchLabels: 
      app: guestbook-db
      tier: db   
  ingress:
  - from:
      podSelector:
        matchLabels:
          app: guestbook-api
          tier: backend
```
Final setup with these network policies allows only the valid communications between the pods

![](/img/network-policy-guestbook3.jpg)

### More about network policy

#### NamespaceSelector
In some scenarios, we have to allow ingress from any pods in a namespace. For example, all applications pods should allow ingress from all pods in `monitoring` namespace. Namespace selector can be used to achieve this setup easily as follows.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-namespace
spec:
  podSelector: {} ## applies to all pods in the namespace
  ingress:
  - from:
      namespaceSelector:
        matchLabels:
          team: monitoring ## labels of the monitoring namespace
```

#### IP Block

Not just labels, network policy also allows configuration using IP blocks. Following network policy allows ingress from 10.72.X.X but blocks traffic from 10.72.10.X

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-using-ip
spec:
  podSelector: {}
  ingress:
  - from:
      ipBlock:
        cidr: 10.72.0.0/16
        except: 10.72.10.0/8
```



## Conclusion

It is good to have security at all levels in k8s cluster. Adding a firewall for the application pods in the kubernetes cluster increases level of security by reducing the attack surface and hence is highly recommended.