# Network Policy Advanced

In this lab you will:
* Create a cluster egress lockdown network policy 
* Selectively allow some pods to bypass the lock down
* Learn how Kubernetes RBAC can be used to manage trust across teams
* Protect hosts using network policy

## Before you begin

This lab builds on the previous lab and cannot be run independently. So if you haven't already done so, complete the [Policy Fundamentals](../100-policy-fundamentals/README.md) lab.

## A. Lazy User's policy has wide open pod Egress

In the K8s Network Policy deployed in the Policy fundamentals lab, the developer's policy allows all pod egress, even to the Internet.

### 1. Confirm that pods are able to initiate connections to the Internet

```
CUSTOMER_POD=$(kubectl get pod -l app=customer -n yaobank -o jsonpath='{.items[0].metadata.name}')
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer bash
ping -c 3 8.8.8.8
curl -I www.google.com
exit
```

This succeeds, i.e., your pods have unfettered Internet access because the developer hasn't properly defined their policy - not good!

## B. Create Egress Lockdown policy as a Security Admin for the cluster

We'll create a Calico GlobalNetworkPolicy to restrict Egress to the Internet to only pods that have the ServiceAccount that is labeled  "internet-egress = allowed".

### 1. Examine the network policy

Examine the policy. While Kubernetes network policies only have Allow rules, Calico network policies also support Deny rules. As this policy has Deny rules in it, it is important that we set its precedence higher than the lazy developer's Allow rules in their Kubernetes policy. To do this we specify `order`  value of 600 in this policy, which give this higher precedence than Kubernetes Network Policy (which does not have the concept of policy precedence, and is assigned a fixed order value of 1000 by Calico - i.e, policy order 600 gets precedence over policy order 1000). 

```
more 210-egress-lockdown.yaml
```

### 2. Apply the network policy
```
calicoctl apply -f 210-egress-lockdown.yaml
```

### 3. Now lets try to access the Internet

```
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer bash
ping -c 3 8.8.8.8
curl -I www.google.com
```

These commands should fail - pods are now restricted to only accessing other pods and nodes within the cluster. You may need to terminate the command with CTRL-C.  Then don't forget to `exit` from the pod to get back to your host terminal.
```
exit
```

## C. Granting selective Internet access

Now imagine there was a legitimate reason to allow connections from the Customer pod to the internet. As we used a Service Account label selector in our egress policy rules, we can enable this by adding the appropriate label to the pod's Service Account.

### 1. Add "internet-egress=allowed" to the Service Account

```
kubectl label serviceaccount -n yaobank customer internet-egress=allowed
```

### 2. Verify the pod can now access the internet
```
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer bash
ping -c 3 8.8.8.8
curl -I www.google.com
exit
```

Now you should find that the customer pod is allowed Internet Egress, but other pods (like Summary and Database) are not.

### 3. Managing trust across teams

There are many ways of dividing responsibilities across teams using Kubernetes RBAC.

Imagine the following:
* The secops team is responsible for creating Namespaces and Services accounts for dev teams. Kubernetes RBAC is setup so that only they can do this.
* Dev teams are given Kubernetes RBAC permissions to create pods in their Namespaces, and they can use, but not modify any Service Account in their Namespaces.

In this scenario, the secops team can control which teams should be allowed to have pods that access the internet.  If a dev team is allowed to have pods that access the internet then the dev team can choose which pods access the internet using by using the appropriate Service Account. 

This is just one way of dividing responsibilities across teams.  Pods, Namespaces, and Service Accounts all have separate Kubernetes RBAC controls and they can all be used to select workloads in Calico network policies.

## D. Protecting the Host

Thus far, we've created policies that protect pods in Kubernetes. However, Calico Policy can also be used to protect the host interfaces in any standalone Linux node (such as a baremetal node, cloud instance or virtual machine) outside the cluster. Furthermore, it can also be used to protect the Kubernetes nodes themselves.

The protection of Kubernetes nodes themselves highlights some of the unique capabilities of Calico - since this needs to account for various control plane services (such as the apiserver, kubelet, controller-manager, etcd, and others. In addition, one needs to also account for certain pods that might be running with host networking (i.e., using the host IP address for the pod) or using hostports. To add an additional layer of challenge, there are also various services (such as Kubernetes NodePorts) that can take traffic coming to reserved port ranges in the host (such as 30000-32767) and NAT it prior to forwarding to a local destination (and perhaps even SNAT traffic prior to redirecting it to a different worker node). 

Lets explore these more advanced scenarios, and how Calico policy can be used to protect these.

### 1. Observe that Kubernetes Control Plane has been left exposed by Kubeadm

Lets start by seeing how the default cluster deployment have left some control plane services exposed to the world.

Run the command below from the standalone Host (host1):

```
curl -v 10.0.0.10:2379
```

This should succeed - i.e., the Kubernetes cluster's ETCD store is left exposed for attacks, along with the rest of the control plane. Not good!!

## E. Adding Host Protection for the Kubernetes Nodes

### 1. Create Network Policy for Hosts
Lets create some host endpoint policies for the kubernetes master node and worker nodes

```
calicoctl apply -f 220-global-host-policy.yaml
```

Now, we have locked down the control plane such that the worker nodes only have kubelet exposed, and the master nodes only have important services (like Etcd) accessible only from other master nodes.

### 2. Create Host Endpoints
Lets now create the Host Endpoints themselves, allowing Calico to start policy enforcement on the nodes ethernet interface. While we're at it, lets also lockdown nodePorts.

```
calicoctl apply -f 230-host-endpoint.yaml
```


### 3. Now lets attempt to attack etcd from the standalone host. Run the curl again from the standalone host (host1):

```
curl -v  -m 5 10.0.0.10:2379
```

This time the curl should fail, and timeout after 5 seconds. 

Terrific, we've locked down the Kubernetes control plane to only be accessible from the relevant places.


## F. Adding Policy Protection for Kubernetes NodePorts

### 1. Verify cannot access yaobank frontend
Now open up a new incognito browser tab (to ensure that a new connection is attempted from your browser) and try to access the Yaobank service again. It fails because our host protection explicitly locked down nodePorts too. This is optional, you don't have to lock down node ports, but if you want to you can.  And you can then use network policy to open up just the specific access you would like.

### 2. Apply policy
Lets use Calico to provide advanced policy to allow specific NodePorts through. From the master, apply the following policy:

```
calicoctl apply -f 240-global-host-policy-allow.yaml
```

### 3. Verify access
Now attempt to access the Yaobank app from your browser again - now it works! 

We hope this illustrates the power of Calico for enabling advanced network security across your Kubernetes cluster, including for the control plane, the master and worker nodes, hostports and host-networked pods, and even for services and nodePorts! 

You'll notice we allowed Service Cluster IPs / External IPs and the PodCIDR within 240-global-host-policy-allow.yaml, together with nodePorts. Why did we do this, after all, aren't cluster IPs and pod IPs inaccessible from outside the Kubernetes cluster? Come to the afternoon session to find out more!
