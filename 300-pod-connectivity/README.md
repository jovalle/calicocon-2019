
# Pod Connectivity

In this lab you will:
* Examine what the network looks like from the perspecitve of a pod (the pod's network namespace)
* Examine the veth pair that connects the pod to the host networking (the host network namespace)
* Examine the routes that tell Linux how to handle traffic to/from pods

## Before you begin

If you haven't already done so, deploy Calico and the YAO Bank sample application as described in the [Get Started](../000-get-started/README.md) notebook.

## A. Examine pod network namespace

We'll start by examining what the network looks like from the pod's point of view. Each pod get's its own Linux network namespace, which you can think of as giving it an isolated copy of the Linux networking stack. 

### 1. Find the name and location of the customer pod
From host 1, get the details the customer pod:
```
kubectl get pods -n yaobank -l app=customer -o wide
```
```
ubuntu@host1:~/calicocon/lab-manifests$ kubectl get pods -n yaobank -l app=customer -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE      NOMINATED NODE   READINESS GATES
customer-5df6b999fb-cf7jl   1/1     Running   0          24h   10.48.0.67   worker1   <none>           <none>
```

Note the node on which the pod is running on (`worker1` in this example.)

### 2. Exec into the customer pod
Use kubectl to exec into the pod so we can take a look around. 
```
kubectl exec -ti -n yaobank $(kubectl get pods -n yaobank -l app=customer -o name) bash
```

### 3. Examine the pod's networking
First we will use `ip addr` to list the addresses and associated network interfaces that the pod sees.
```
ip addr
```
```
root@customer-5df6b999fb-cf7jl:/app# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 1a:ce:f0:6c:20:00 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.48.0.67/32 scope global eth0
       valid_lft forever preferred_lft forever
```

The key things to note in this output are:
* There is a `lo` loopback interface with an IP address of `127.0.0.1`. This is the standard loopback interface that every network namespace has by default. You can think of it as `localhost` for the pod itself.
* There is an `eth0` interface which has the pods actual IP address, `10.48.0.67`. Notice this matches the IP address that `kubectl get pods` returned earlier.

Next let's look more closely at the interfaces using `ip link`.  We will use the `-c` option, which colors the output to make it easier to read.

```
ip -c link
```
```
root@customer-5df6b999fb-cf7jl:/app# ip -c link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 1a:ce:f0:6c:20:00 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

Look at the `eth0` part of the output. The key things to note are:
* The `eth0` interface is interface number 3 in the pod's network namespace.
* `eth0` is a link to the host network namespace (indicated by `link-netnsid 0`). i.e. It is the pod's side of the veth pair (virtual ethernet pair) that connects the pod to the host's networking.
* The `@if9` at the end of the interface name is the interface number of the other end of the veth pair within the host's network namespaces. In this example, interface number 9.  Remember this for later. We will take look at the other end of the veth pair shortly.

Finally, let's look at the routes the pod sees.

```
ip route
```
```
root@customer-5df6b999fb-cf7jl:/app# ip route
default via 169.254.1.1 dev eth0 
169.254.1.1 dev eth0  scope link 
```
This shows that the pod's default route is out over the `eth0` interface. i.e. Anytime it wants to send traffic to anywhere other than itself, it will send the traffic over `eth0`.

### 4. Exit from the customer pod
We've finished our tour of the pod's view of the network, so we'll exit out of the exec to return to host1.
```
exit
```

## B. Examine the host's network namespace

### 1. SSH into the customer pod's host node
We'll start by switching to the node where the customer pod is running. In our example earlier this was worker 1. (If you've forgotten which node it was for you then repeat A.1 above to find the node.)
```
ssh worker1
```

### 2. Examine interfaces
Now we're on the node hosting the customer pod we'll take a look to examine the other end of the veth pair. In our example output earlier, the `@if9` indicated it should be interface number 9 in the host network namespace. (Your interface numbers may be different, but you should be able to follow along the same logic.)
```
ip -c link
```
```
ubuntu@worker1:~$ ip -c link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 00:50:56:37:2e:a2 brd ff:ff:ff:ff:ff:ff
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
    link/ether 02:42:40:0f:c5:68 brd ff:ff:ff:ff:ff:ff
4: cali282de2964bf@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 0
7: cali4b8cf63cac1@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 1
8: cali2b0ff6bbf7d@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2
9: calie965c96dc90@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 3
```

Looking at interface number 9 in this example we see `calie965c96dc90` which links to `@if3` in network namespace ID 3 (the customer pod's network namespace).  You may recall that interace 3 in the pod's network namespace was `eth0`, so this looks exactly as expected for the veth pair that connects the customer pod to the host network namespace.  

You can also see the host end of the veth pairs to other pods running on this node, all beginning with `cali`.

### 2. Examine routes
First let's remind ourselves of the `customer` pod's IP address:
```
kubectl get pods -n yaobank -l app=customer -o wide
````

Now lets look at the routes on the host.
```
ip route
```
```
ubuntu@worker1:~$ ip route
default via 10.0.0.254 dev ens160 proto dhcp src 10.0.0.11 metric 100 
10.0.0.0/24 dev ens160 proto kernel scope link src 10.0.0.11 
10.0.0.254 dev ens160 proto dhcp scope link src 10.0.0.11 metric 100 
10.48.0.64 dev cali282de2964bf scope link 
blackhole 10.48.0.64/26 proto bird 
10.48.0.65 dev cali4b8cf63cac1 scope link 
10.48.0.66 dev cali2b0ff6bbf7d scope link 
10.48.0.67 dev calie965c96dc90 scope link 
10.48.0.128/26 via 10.0.0.12 dev ens160 proto bird 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
```

In this example output, we can see the route to the customer pod's IP (`10.48.0.67`) is via the `calie965c96dc90` interface, the host end of the veth pair for the customer pod. You can see similar routes for each of the IPs of the other pods hosted on this node. It's these routes that tell Linux where to send traffic that is destined to a local pod on the node.

We can also see several routes labeled `proto bird`. These are routes to pods on other nodes that Calico has learned over BGP. 

To understand these better, consider this route in the example output above `10.48.0.128/26 via 10.0.0.12 dev ens160 proto bird `.  It indicates pods with IP addresses falling within the `10.48.0.128/26` CIDR can be reached via `10.0.0.12` (which is worker2) through the `ens160` network interface (the host's main interface to the rest of the network). You should see similar routes in your output for each node.

Calico uses route aggregation to reduce the number of routes when possible. (e.g. `/26` in this example). The `/26` corresponds to the default block size that Calico IPAM (IP Address Management) allocates on demand as nodes need pod IP addresses. (If desired, the block size can be configured in Calico IPAM settings.)  

You can also see the `blackhole 10.48.0.64/26 proto bird` route. The `10.48.0.64/26` corresponds to the block of IPs that Calico IPAM allocated on demand for this node. This is the block from which each of the local pods got their IP addresses. The blackhole route tells Linux that if it can't find a more specific route for an individual IP in that block then it should discard the packet (rather than sending it out the default route to the network). You will only see traffic that hits this rule if something is trying to send traffic to a pod IP that doesn't exist, for example sending traffic to a recently deleted pod.

If Calico IPAM runs out of blocks to allocate to nodes, then it will use unused IPs from other nodes' blocks. These will be announced over BGP as more specific routes, so traffic to pods will always find its way to the right host.

### 4. Exit from the node
We've finished our tour of the Customer pod's host's view of the network. Remember exit out of the exec to return to host1.
```
exit
```
