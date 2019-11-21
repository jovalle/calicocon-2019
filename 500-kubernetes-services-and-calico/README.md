# Kubernetes Services and Calico

In this lab you will examine how kube-proxy uses iptables to implement:
* cluster IPs
* node ports

## Before you begin

If you haven't already done so, deploy Calico and the YAO Bank sample application as described in the [Get Started](../000-get-started/README.md) notebook.


## A. SSH into a node

We will run this lab from `worker1` so we can explore the iptables rules that kube-proxy has set up.

```
ssh worker1
```

## B. Examine Kubernetes Services

Let's take a look at the services and pods in the `yaobank` kubernetes namespace.

### 1. List services

```
kubectl get svc -n yaobank
```
```
ubuntu@worker1:~/calicocon/lab-manifests$ kubectl get svc -n yaobank
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
customer   NodePort    10.49.217.62    <none>        80:30180/TCP   16m
database   ClusterIP   10.49.252.124   <none>        2379/TCP       16m
summary    ClusterIP   10.49.163.205   <none>        80/TCP         16m
```

We should have three services deployed. One `NodePort` service and two `ClusterIP` services.

### 2. List service endpoints
Find the endpoints for each of the services.
```
kubectl get endpoints -n yaobank
```
```
ubuntu@worker1:~/calicocon/lab-manifests$ kubectl get endpoints -n yaobank
NAME       ENDPOINTS                     AGE
customer   10.48.0.128:80                16m
database   10.48.0.66:2379               16m
summary    10.48.0.65:80,10.48.0.67:80   16m
```

### 3. List pods
```
kubectl get pods -n yaobank -o wide
```
```
ubuntu@worker1:~/calicocon/lab-manifests$ kubectl get pods -n yaobank -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE      NOMINATED NODE   READINESS GATES
customer-6fdc8cc787-cf7jl   1/1     Running   0          17m   10.48.0.128   worker1   <none>           <none>
database-69db5dc58d-dx2pv   1/1     Running   0          17m   10.48.0.66    worker2   <none>           <none>
summary-6c755fccd5-vhqvn    1/1     Running   0          17m   10.48.0.67    worker2   <none>           <none>
summary-6c755fccd5-xcswm    1/1     Running   0          17m   10.48.0.65    worker2   <none>           <none>
```
You can see that the IP addresses listed as the service endpoints in the previous step map to the backing pods, as expected. Each service is backed by one or more pods spread across the worker nodes in our cluster.

## B. Explore ClusterIP iptables rules

Let's explore the iptables rules that implement the `summary` service.

### 1. Get service endpoints
Find the service endpoints for `summary` `ClusterIP` service.
```
kubectl get endpoints -n yaobank summary
```
```
ubuntu@worker1:~/calicocon/lab-manifests$ kubectl get endpoints -n yaobank summary
NAME      ENDPOINTS                     AGE
summary   10.48.0.65:80,10.48.0.67:80   18m
```

The `summary` service has two endpoints (`10.48.0.65` on port `80` AND `10.48.0.67` on port `80` in this example output). Starting from the `KUBE-SERVICES` iptables chain, we will traverse each chain until you get to the rule directing traffic to these endpoint IP addresses.

### 2. Examine the KUBE-SERVICE chain
```
sudo iptables -v --numeric --table nat --list KUBE-SERVICES
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-SERVICES
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.163.205        /* yaobank/summary:http cluster IP */ tcp dpt:80
    0     0 KUBE-SVC-OIQIZJVJK6E34BR4  tcp  --  *      *       0.0.0.0/0            10.49.163.205        /* yaobank/summary:http cluster IP */ tcp dpt:80
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.217.62         /* yaobank/customer:http cluster IP */ tcp dpt:80
    0     0 KUBE-SVC-PX5FENG4GZJTCELT  tcp  --  *      *       0.0.0.0/0            10.49.217.62         /* yaobank/customer:http cluster IP */ tcp dpt:80
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  *      *       0.0.0.0/0            10.49.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
    0     0 KUBE-SVC-ERIFXISQEP7F7OF4  tcp  --  *      *       0.0.0.0/0            10.49.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
    0     0 KUBE-SVC-JD5MR3NA4I4DYORP  tcp  --  *      *       0.0.0.0/0            10.49.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
    0     0 KUBE-MARK-MASQ  udp  --  *      *      !10.48.0.0/16         10.49.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
    0     0 KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  *      *       0.0.0.0/0            10.49.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.252.124        /* yaobank/database:http cluster IP */ tcp dpt:2379
    0     0 KUBE-SVC-AE2X4VPDA5SRYCA6  tcp  --  *      *       0.0.0.0/0            10.49.252.124        /* yaobank/database:http cluster IP */ tcp dpt:2379
    1    79 KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```

Each iptables chain consists of a list of rules that are executed in order until a rule matches. The key columns/elements to note in this output are:
* `target` - which chain iptables will jump to if the rule matches
* `prot` - the protocol match criteria
* `source`, and `destination` - the source and destination IP address match criteria
* the comments that kube-proxy inculdes
* the additional match criteria at the end of each rule - e.g `dpt:80` that specifies the destination port match

### 3. KUBE-SERVICES -> KUBE-SVC-XXXXXXXXXXXXXXXX

Now let's look more closely at the rules for the `summary` service.
```
sudo iptables -v --numeric --table nat --list KUBE-SERVICES | grep -E summary
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-SERVICES | grep -E summary
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.163.205        /* yaobank/summary:http cluster IP */ tcp dpt:80
    0     0 KUBE-SVC-OIQIZJVJK6E34BR4  tcp  --  *      *       0.0.0.0/0            10.49.163.205        /* yaobank/summary:http cluster IP */ tcp dpt:80
```

The second rule directs traffic destined for the summary service clusterIP (`10.49.163.205` in the example output) to the chain that load balances the service (KUBE-SVC-XXXXXXXXXXXXXXXX).

### 4. KUBE-SVC-XXXXXXXXXXXXXXXX -> KUBE-SEP-XXXXXXXXXXXXXXXX

`kube-proxy` in `iptables` mode uses a randomized equal cost selection algorithm to load balance traffic between pods.  We currently have two `summary` pods.

Let's examine how this loadbalancing works. (Remember your chain name may be different than this example.)
```
sudo iptables -v --numeric --table nat --list KUBE-SVC-OIQIZJVJK6E34BR4
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-SVC-OIQIZJVJK6E34BR4
Chain KUBE-SVC-OIQIZJVJK6E34BR4 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-SEP-QBQ2ZLCSJLFEASMX  all  --  *      *       0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-BS2BXDAXX5ZBJNP2  all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

Notice that `kube-proxy` is using the `iptables` `statistic` module to set the probability for a packet to be randomly matched.  Make sure you scroll all the way to the right to see the full output.

The first rule directs traffic destined for the `summary` service to the chain that delivers packets to the first service endpoint (KUBE-SEP-QBQ2ZLCSJLFEASMX) with a probability of 0.50000000000. The second rule unconditionally directs to the second service endpoint chain (KUBE-SEP-BS2BXDAXX5ZBJNP2). The result is that traffic is load balanced across the service endpoints equally (on average).

If there were 3 service endpoints then the first chain matches would be probability 0.33333333, the second probability 0.5, and the last unconditional. The result is each service endpoint receives a third of the traffic (on average).

### 5. KUBE-SEP-XXXXXXXXXXXXXXXX -> `summary` pod
Let's look at one of the service endpoint chains. (Remember your chain names may be different than this example.)
```
sudo iptables -v --numeric --table nat --list KUBE-SEP-QBQ2ZLCSJLFEASMX
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-SEP-QBQ2ZLCSJLFEASMX
Chain KUBE-SEP-QBQ2ZLCSJLFEASMX (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.48.0.65           0.0.0.0/0
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp to:10.48.0.65:80
```

The second rule performs the DNAT that changes the destination IP from the service's clusterIP to the IP address of the service endpoint backing pod (`10.48.0.65` in this example). After this, standard Linux routing can handle forwarding the packet like it would any other packet.

### 6. Recap
You've just traced the kube-proxy iptables rules used to load balance traffic to `summary` pods exposed as a service of type `ClusterIP`.

In summary, for a packet being sent to a clusterIP:
* The KUBE-SERVICES chain matches on the clusterIP and jumps to the corresponding KUBE-SVC-XXXXXXXXXXXXXXXX chain.
* The KUBE-SVC-XXXXXXXXXXXXXXXX chain load balances the packet to a random service endpoint KUBE-SEP-XXXXXXXXXXXXXXXX chain.
* The KUBE-SEP-XXXXXXXXXXXXXXXX chain DNATs the packet so it will get routed to the service endpoint (backing pod).

## C. Explore NodePort iptables rules

Let's explore the iptables rules that implement the `customer` service.

### 1. Get service endpoints
Find the service endpoints for `customer` `NodePort` service.
```
kubectl get endpoints -n yaobank customer
```
```
ubuntu@worker1:~$ kubectl get endpoints -n yaobank customer
NAME       ENDPOINTS        AGE
customer   10.48.0.128:80   24m
```

The `customer` service has one endpoint (`10.48.0.128` on port `80` in this example output). Starting from the `KUBE-SERVICES` iptables chain, we will traverse each chain until you get to the rule directing traffic to theis endpoint IP address.

### 2. KUBE-SERVICES -> KUBE-NODEPORTS
The `KUBE-SERVICE` chain handles the matching for service types `ClusterIP` and `LoadBalancer`. At the end of `KUBE-SERVICE` chain, another custom chain `KUBE-NODEPORTS` will handle traffic for service type `NodePort`.
```
sudo iptables -v --numeric --table nat --list KUBE-SERVICES | grep KUBE-NODEPORTS
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-SERVICES | grep KUBE-NODEPORTS
    4   278 KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```

`match dst-type LOCAL` matches any packet with a local host IP as the destination. I.e. any address that is assigned to one of the host's interfaces.

### 3. KUBE-NODEPORTS -> KUBE-SVC-XXXXXXXXXXXXXXXX
```
sudo iptables -v --numeric --table nat --list KUBE-NODEPORTS
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-NODEPORTS
Chain KUBE-NODEPORTS (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/customer:http */ tcp dpt:30180
    0     0 KUBE-SVC-PX5FENG4GZJTCELT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/customer:http */ tcp dpt:30180
```

The second rule directs traffic destined for the `customer` service to the chain that load balances the service (KUBE-SVC-PX5FENG4GZJTCELT). `tcp dpt:30180` matches any packet with the destination port of tcp 30180 (the node port of the `customer` service).

#### 4. KUBE-SVC-XXXXXXXXXXXXXXXX -> KUBE-SEP-XXXXXXXXXXXXXXXX
(Remember your chain name may be different than this example.)
```
sudo iptables -v --numeric --table nat --list KUBE-SVC-PX5FENG4GZJTCELT
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-SVC-PX5FENG4GZJTCELT
Chain KUBE-SVC-PX5FENG4GZJTCELT (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-SEP-DES65NIG7LUKIP6J  all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

As we only have a single backing pod for the `customer` service, there is no loadbalncing to do, so there is a single rule that directs all traffic to the chain that delivers the packet to the service endpoint (KUBE-SEP-XXXXXXXXXXXXXXXX).

### 5. KUBE-SEP-XXXXXXXXXXXXXXXX -> `customer` endpoint
(Remember your chain name may be different than this example.)
```
sudo iptables -v --numeric --table nat --list KUBE-SEP-DES65NIG7LUKIP6J
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-SEP-DES65NIG7LUKIP6J
Chain KUBE-SEP-DES65NIG7LUKIP6J (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.48.0.128          0.0.0.0/0
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp to:10.48.0.128:80
```

This rule delivers the packet to the `customer` service endpoint.

The second rule performs the DNAT that changes the destination IP from the service's clusterIP to the IP address of the service endpoint backing pod (`10.48.0.128` in this example). After this, standard Linux routing can handle forwarding the packet like it would any other packet.

### 6. Recap
You've just traced the kube-proxy iptables rules used to load balance traffic to `customer` pods exposed as a service of type `NodePort`.

In summary, for a packet being sent to a NodePort:
* The end of the KUBE-SERVICES chain jumps to the KUBE-NODEPORTS chain
* The KUBE-NODEPORTS chaing matches on the NodePort and jumps to the corresponding KUBE-SVC-XXXXXXXXXXXXXXXX chain.
* The KUBE-SVC-XXXXXXXXXXXXXXXX chain load balances the packet to a random service endpoint KUBE-SEP-XXXXXXXXXXXXXXXX chain.
* The KUBE-SEP-XXXXXXXXXXXXXXXX chain DNATs the packet so it will get routed to the service endpoint (backing pod).

## E. Exit back to host1
We're done with this lab.  Exit the ssh session to worker1 and return to host1.
```
exit
```
