
# Pod Connectivity Advanced

In this lab you will:
* Examine the different IP address ranges used by the cluster
* Create seperate Calico IP Pools with different routability scopes
* Configure Calico BGP Peering to connect with a network outside of the cluster
* Configure a namespace to use externally routable IP addresses

## Before you begin

If you haven't already done so, deploy Calico and the YAO Bank sample application as described in the [Get Started](../000-get-started/README.md) notebook.

## A. Examine IP address ranges used by the cluster

### 1. Kubernetes configuration
There are two address ranges that Kubernetes is normally configured with that are worth understanding:
* The cluster pod CIDR is the range of IP addresses Kubernetes is expecting to be assigned to pods in the cluster.
* The services CIDR is the range of IP addresses that are used for the Cluster IPs of Kubernetes Sevices (the virtual IP that corresponds to each Kubernetes Service).

These are configured at cluster creation time (e.g. as initial kubeadm configuration).

You can find these values using the following command on host1:
```bash
kubectl cluster-info dump | grep -m 2 -E "service-cluster-ip-range|cluster-cidr"
```
```
ubuntu@host1:~/calicocon/lab-manifests$ kubectl cluster-info dump | grep -m 2 -E "service-cluster-ip-range|cluster-cidr"
                            "--service-cluster-ip-range=10.49.0.0/16",
                            "--cluster-cidr=10.48.0.0/16",
```

### 2. Calico configuration
It is also important to understand the IP Pools that Calico has been configure with, which offer finer grained control of IP address ranges to be used by pods in the cluster.
```bash
calicoctl get ippools
```
```
ubuntu@host1:~/calicocon/lab-manifests$ calicoctl get ippools
NAME                  CIDR           SELECTOR   
default-ipv4-ippool   10.48.0.0/24   all()  
```

In this cluster Calico has been configured to allocate IP addresses for pods from the `10.48.0.0/24` CIDR (which is a subset of the `10.48.0.0/16` configured on Kubernetes).

Overall we have the following address ranges:

| CIDR         |  Purpose                                                  |
|--------------|-----------------------------------------------------------|
| 10.48.0.0/16 | Kubernetes Pod Network (via kubeadm `--pod-network-cidr`) |
| 10.48.0.0/24 | Calico - Initial default IP Pool                           |
| 10.49.0.0/16 | Kubernetes Service Network (via kubeadm `--service-cidr`) |


## B. Create additional Calico IP Pools
One use of Calico IP Pools is to distinguish between different ranges of addresses that different routablity scopes. If you are operating at very large scales then IP addresses are precious. You might want to have a range of IPs that is only routable within the cluster, and another range of IPs that is routable across the whole of your enterprise. Then you would choose which pods should get IPs from which range depending on whether workloads from outside of the cluster need to directly access the pods or not.

We'll simulate this use case in this lab by creating a second IP Pool to represent the externally routable pool.  (And we've already configured the underlying network to no allow routing of the existing IP Pool outside of the cluster.)

### 1. Create externally routable IP Pool

We're going to create a new pool for `10.48.2.0/24` that is externally routable.
```
calicoctl apply -f 410-pool.yaml
calicoctl get ippools
```
```
ubuntu@host1:~/calicocon/lab-manifests$ calicoctl apply -f 410-pool.yaml
Successfully applied 1 'IPPool' resource(s)
ubuntu@host1:~/calicocon/lab-manifests$ calicoctl get ippools
NAME                  CIDR           SELECTOR   
default-ipv4-ippool   10.48.0.0/24   all()      
external-pool         10.48.2.0/24   all()      
```

We now have:

| CIDR         |  Purpose                                                  |
|--------------|-----------------------------------------------------------|
| 10.48.0.0/16 | Kubernetes Pod Network (via kubeadm `--pod-network-cidr`) |
| 10.48.0.0/24 | Calico - Initial default IP Pool                          |
| 10.48.2.0/24 | Calico - External IP Pool (externally routable)           |
| 10.49.0.0/16 | Kubernetes Service Network (via kubeadm `--service-cidr`) |

## C. Configure Calico BGP peering

### 1. Examine BGP peering status

Switch to worker1:
```
ssh worker1
```

Check the status of Calico on the node:
```bash
sudo calicoctl node status
```
```
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 10.0.0.10    | node-to-node mesh | up    | 19:34:19 | Established |
| 10.0.0.12    | node-to-node mesh | up    | 19:34:19 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```
This shows that currently Calico is only peering with the other nodes in the cluster and is not peering to any networks outside of the cluster.

Exit back to host1:
```
exit
```

### 2. Add a BGP Peer

In this lab we will simulate peering to a network outside of the cluster by peering to host1. (We've set up host1 to act as if it were a router, and it is ready to accept new BGP peering requests.)

Add the new BGP Peer:
```bash
calicoctl apply -f 420-bgp-peer.yaml
```
You should see the following output when you apply the new bgp peer resource. 

```
ubuntu@worker1:~$ calicoctl apply -f 420-bgp-peer.yaml
Successfully applied 1 'BGPPeer' resource(s)
```

### 3. Examine the new BGP peering status
Switch to worker1:
```
ssh worker1
```

Check the status of Calico on the node:
```bash
sudo calicoctl node status
```
```
ubuntu@worker1:~$ sudo calicoctl node status
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+------------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |   SINCE    |    INFO     |
+--------------+-------------------+-------+------------+-------------+
| 10.0.0.10    | node-to-node mesh | up    | 2019-11-13 | Established |
| 10.0.0.12    | node-to-node mesh | up    | 2019-11-13 | Established |
| 10.0.0.20    | global            | up    | 04:28:42   | Established |
+--------------+-------------------+-------+------------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

The output shows that Calico is now peered with host1 (`10.0.0.20`). This means Calico can share routes to and learn routes from host1.

In a real-world on-prem deployment you would typically configure Calico nodes within a rack to peer with the ToRs (Top of Rack) routers, and the ToRs are then connected to the rest of the enterprise or data center network. In this way pods, if desired, pods can be address from anywhere on your network. You could even go as far as giving some pods public IP address and have them addressable from the internet if you wanted to.

We're done with adding the peers, so exit from worker1 to return back to host1:
```
exit
```

## D. Configure a namespace to use externally routable IP addresses

Calico supports annotations on both namespaces and pods that can be used to control which IP Pool (or even which IP address) a pod will receive when it is created.  In this example we're going to create a namespace to host out externally routable.

### 1. Create the namespace

Examine the namespaces we're about to create:
```
more 430-namespace.yaml
```
Notice the annotation that will determine which IP Pool pods in the namespace will use.

Apply the namespace:
```bash
kubectl apply -f 430-namespace.yaml
```
```
ubuntu@host1:~$ kubectl apply -f 430-namespace.yaml
namespace/external-ns created
```

### 2. Deploy an nginx pod
Now deploy a NGINX example pod in the `external-ns` namespace, along with a simple network policy that allows ingress on port 80.
```
kubectl apply -f 440-nginx.yaml
```
```
ubuntu@host1:~/calicocon/lab-manifests$ kubectl apply -f 440-nginx.yaml
deployment.apps/nginx created
networkpolicy.networking.k8s.io/nginx created
```

### 3. Access the nginx pod from outside the cluster

Let's see what IP address was assigned:
```
kubectl get pods -n external-ns -o wide
```
```
ubuntu@host1:~/calicocon/lab-manifests$ kubectl get pods -n external-ns -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP            NODE      NOMINATED NODE   READINESS GATES
nginx-7bb7cd8db5-zqzks   1/1     Running   0          5m44s   10.48.2.232   worker1   <none>           <none>
```

The output shows that the nginx pod has an IP address from the externally routable IP Pool.

Try to connect to it from host1:
```
curl 10.48.2.232
```

This should have suceeded, showing that the nginx pod is directly routable on the broader network.

If you would like to see IP allocation stats from Calico-IPAM, run the following command.

```
calicoctl ipam show
```

