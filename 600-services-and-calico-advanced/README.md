# Advanced Services and Calico

In this lab you will learn how you can advertise Kubernetes services over BGP:
* Advertise the service IP range
* Advertise individual service cluster IP
* Advertise individual service external IP

## Before you begin

If you haven't already done so:
* Deploy Calico and the YAO Bank sample application as described in the [Get Started](../000-get-started/README.md) notebook.
* Add host1 as a BGP peer as described in step C of the [Advanced Pod Connectivity](../400-pod-connectivity-advanced/README.md)  notebook.

## A. Advertise service cluster IP addresses

Advertising services over BGP allows you to directly access the service without using NodePorts or a cluster Ingress Controller.

### 1. Examine routes
Before we begin take a look at the state of routes on the standalone host (host1).
```
ip route
```
```
ubuntu@host1:~/calicocon/lab-manifests$ ip route
default via 10.0.0.254 dev ens160 proto dhcp src 10.0.0.20 metric 100
10.0.0.0/24 dev ens160 proto kernel scope link src 10.0.0.20
10.0.0.254 dev ens160 proto dhcp scope link src 10.0.0.20 metric 100
10.48.2.232/29 via 10.0.0.11 dev ens160 proto bird
```
If you completed the previous lab you'll see one route that was learned from Calico that provides access to the nginx pod that was created in the externally routable namespace (the route ending in `proto bird` in this example output). In this lab we will advertise Kubernetes services (rather than individual pods) over BGP.

### 2. Add Calico BGP configuration

Examine the default BGP configuration we are about to apply
```
more 610-default-bgp.yaml
```
The `serviceClusterIPs` clause tells Calico to advertise the cluster IP range.

Apply the configuration
```
calicoctl apply -f 610-default-bgp.yaml
```

Verify the BGPConfiguration contains the `serviceClusterIPs` key
```
calicoctl get bgpconfig default -o yaml
```
```
ubuntu@host1:~/calicocon/lab-manifests$ calicoctl get bgpconfig default -o yaml
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  creationTimestamp: 2019-11-15T21:44:22Z
  name: default
  resourceVersion: "446283"
  uid: dd064f46-f82f-4f19-9c03-2e165839dac4
spec:
  serviceClusterIPs:
  - cidr: 10.49.0.0/16
```

### 3. Examine routes

Examine the routes again on host1.
```
ip route
```
```
ubuntu@host1:~/calicocon/lab-manifests$ ip route
default via 10.0.0.254 dev ens160 proto dhcp src 10.0.0.20 metric 100
10.0.0.0/24 dev ens160 proto kernel scope link src 10.0.0.20
10.0.0.254 dev ens160 proto dhcp scope link src 10.0.0.20 metric 100
10.48.2.176/29 via 10.0.0.12 dev ens160 proto bird
10.48.2.232/29 via 10.0.0.11 dev ens160 proto bird
10.49.0.0/16 proto bird
	nexthop via 10.0.0.10 dev ens160 weight 1
	nexthop via 10.0.0.11 dev ens160 weight 1
	nexthop via 10.0.0.12 dev ens160 weight 1
```
You should now see the cluster service cidr `10.49.0.0/16` advertised from each of the kubernetes cluster nodes. This means that traffic to any service's cluster IP address will get loadbalanced across all nodes in the cluster by the network using ECMP (Equal Cost Multi Path). Kube-proxy then load balances the cluster IP across the service endponts (backing pods) in exactly the same way as if a pod had accessed a service via a cluster IP.

### 4. Verify we can access cluster IPs

Find the cluster IP for the `customer` service.

```
kubectl get svc -n yaobank customer
```
```
ubuntu@host1:~/calicocon/lab-manifests$ kubectl get svc -n yaobank customer
NAME       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
customer   NodePort   10.49.217.62   <none>        80:30180/TCP   2d1h
```
In this example output it is `10.49.217.62`.  Your IP may be different.

Confirm we can access it from host1.
```
curl 10.49.217.62
```


## B. Advertise local service cluster IP addresses

You can set `externalTrafficPolicy: Local` on a Kubernetes service to request that external traffic to a service should only be routed via nodes which have a local service endpoint (backing pod). This preserves the client source IP and avoids the second hop associated  NodePort or ClusterIP services when kube-proxy loadbalances to a service endpoint (backing pod) on another node.

Traffic to the cluster IP for a service with `externalTrafficPolicy: Local` will be load-balanced across the nodes with endpoints for that service.

### 1. Add external traffic policy
Update the `customer` service to add `externalTrafficPolicy: Local`.

```
kubectl patch svc -n yaobank customer -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

### 2. Examine routes
```
ip route
```
```
ubuntu@host1:~/calicocon/lab-manifests$ ip route
default via 10.0.0.254 dev ens160 proto dhcp src 10.0.0.20 metric 100
10.0.0.0/24 dev ens160 proto kernel scope link src 10.0.0.20
10.0.0.254 dev ens160 proto dhcp scope link src 10.0.0.20 metric 100
10.48.2.176/29 via 10.0.0.12 dev ens160 proto bird
10.48.2.232/29 via 10.0.0.11 dev ens160 proto bird
10.49.0.0/16 proto bird
	nexthop via 10.0.0.10 dev ens160 weight 1
	nexthop via 10.0.0.11 dev ens160 weight 1
	nexthop via 10.0.0.12 dev ens160 weight 1
10.49.217.62 via 10.0.0.11 dev ens160 proto bird
```

You should now have a `/32` route for the yaobank customer service (`10.49.217.62` in the above example output) advertised from the node hosting the customer service pod (worker1, `10.0.0.11` in this example output).

For each active service with `externalTrafficPolicy: Local`, Calico advertise the IP for that service as a `/32` route from the nodes that have endpoints for that service. This means that external traffic to the service will get loadbalanced across all nodes in the cluster that have a service endpoint (backing pod) for the service by the network using ECMP (Equal Cost Multi Path). Kube-proxy then DNATs the traffic to the local backing pod on that node (or loadbalance equally to the local backing pods if there is more than one on the node).

The two main advantages of using `externalTrafficPolicy: Local` in this way are:
* There is a network efficiency win avoiding potential second hop of kube-proxy loadbalancing to another node.
* The client source IP addresses are preserved, which can be useful if you want to restrict access to a service to specific IP addresses using network policy applied to the backing pods.  (This is an alternative approach to that explored earlier in the [Advanced Policy](../200-policy-advanced/README.md) lab, where we used Calico host endpoint `preDNAT` policy to restict external traffic to the services.)


### 3. Note the impact on nodePorts
Earlier in these labs we accessed the YAO Bank frontend UI from your browser. The link you were using maps to a nodePort on `master1`. As we've now set `externalTrafficPolicy: Local`, this will no long work since there are no `customer` pods hosted on `master1`. Accessing via the nodePort on `worker1` would work of course.

## C. Advertise service external IP addresses

If you want to advertise a service using an IP address outside of the service cluster IP range, you can use configure the service to have one or more `externalIPs`.

### 1. Examine the existing services
Before we begin, examine the kubernetes services in the `yaobank` kubernetes namespace.

```
kubectl get svc -n yaobank
```
```
ubuntu@host1:~/calicocon/lab-manifests$ kubectl get svc -n yaobank
NAME          TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
customer      NodePort       10.49.217.62    <none>        80:30180/TCP   59m
database      ClusterIP      10.49.252.124   <none>        2379/TCP       59m
summary       ClusterIP      10.49.163.205   <none>        80/TCP         59m
```

Note that none of them currently have an `EXTERNAL-IP`.

### 2. Update BGP configuration

Update the Calico BGP configuration to advertise a service external IP CIDR range of `10.50.0.0/24`.

```
calicoctl patch BGPConfig default --patch \
   '{"spec": {"serviceExternalIPs": [{"cidr": "10.50.0.0/24"}]}}'
```

Note that `serviceExternalIPs` is a list of CIDRs, so you could for example add individual /32 IP addresses if there were just a small number of specific IPs you wanted to advertise.

### 3. Examine routes
```
ip route
```
```
ubuntu@host1:~/calicocon/lab-manifests$ ip route
default via 10.0.0.254 dev ens160 proto dhcp src 10.0.0.20 metric 100
10.0.0.0/24 dev ens160 proto kernel scope link src 10.0.0.20
10.0.0.254 dev ens160 proto dhcp scope link src 10.0.0.20 metric 100
10.48.2.176/29 via 10.0.0.12 dev ens160 proto bird
10.48.2.232/29 via 10.0.0.11 dev ens160 proto bird
10.49.0.0/16 proto bird
	nexthop via 10.0.0.10 dev ens160 weight 1
	nexthop via 10.0.0.11 dev ens160 weight 1
	nexthop via 10.0.0.12 dev ens160 weight 1
10.49.217.62 via 10.0.0.11 dev ens160 proto bird
10.50.0.0/24 proto bird
	nexthop via 10.0.0.10 dev ens160 weight 1
	nexthop via 10.0.0.11 dev ens160 weight 1
	nexthop via 10.0.0.12 dev ens160 weight 1
```

You should now have a routes for the external ID CIDR (10.50.0.10/24) with next hops to each of our cluster nodes.

### 4. Assign the service external IP

Assign the service external IP `10.50.0.10` to the `customer` service.
```
kubectl patch svc -n yaobank customer -p  '{"spec": {"externalIPs": ["10.50.0.10"]}}'
```

Examine the services again to validate everything is as expected:
```
kubectl get svc -n yaobank
```
```
ubuntu@host1:~/calicocon/lab-manifests$ kubectl get svc -n yaobank
NAME          TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
customer      NodePort       10.49.217.62    10.50.0.10    80:30180/TCP   59m
database      ClusterIP      10.49.252.124   <none>        2379/TCP       59m
summary       ClusterIP      10.49.163.205   <none>        80/TCP         59m

```

You should now see the external ip (`10.50.0.10`) assigned to the `customer` service.  We can now access the `customer` service from outside the cluster using the external ip address (10.50.0.10) we just assigned.

### 5. Verify we can access the service's external IP

Connect to the `customer` service from the standalone node using the service external IP `10.50.0.10`.
```
curl 10.50.0.10
```

As you can see the service has been made available outside of the cluster via bgp routing and load balancing.

## D. Recap

We've covered five different ways for connecting to your pods from outside the cluster during these labs.
* Via a standard NodePort on a specific node. (This is how you connected to the YAO Bank web front end from your browser.)
* Direct to the pod IP address by configuring a Calico IP Pool that is externally routable.
* Advertising the service cluster IP range. (And using ECMP to load balance across all nodes in the cluster.)
* Advertising individual cluster IPs. (Services with `externalTrafficPolicy: Local`, using ECMP to load balance only to the nodes hosting the pods backing the service.)
* Advertising service external-IPs. (So you can use service IP addresses outside of the cluster IP range.)

There are more ways, but this is all we had time for in this workshop. Nonetheless, we hope this gives you an idea of Calico's versatility and ability to fit with a broad range of networking needs.

