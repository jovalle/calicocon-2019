
# Network Policy Fundamentals

In this lab you will:
* Simulate a compromise
* Apply network policy to the Database
* Apply default deny policy across all pods in the cluster
* Apply network policy to allow DNS access across all pods in the cluster
* Apply network policy to the rest of the sample application

## Before you begin

If you haven't already done so, deploy the YAO Bank sample application as described in the [Get Started](../000-get-started/README.md) notebook.

## A. Simulate a compromise

We can simulate a compromise of the customer pod by just exec'ing into the pod and attempting to access the database directly from there.

### 1. Exec into the Customer pod

```
# Determine customer pod name
CUSTOMER_POD=$(kubectl get pod -l app=customer -n yaobank -o jsonpath='{.items[0].metadata.name}')

# Execute bash command in customer container
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer bash
```

### 2. Access the Database
From within the customer pod, attempt to access the database directly - the attack will succeed, and the balance of all users will be returned.

```
curl http://database:2379/v2/keys?recursive=true | python -m json.tool
```

### 3. Exit the from the exec
After running the command, remember to exit from the kubectl exec of the customer pod by typing "exit".
```
exit
```

## B. Create Kubernetes Network Policy limiting access

We can use a Kubernetes Network Policy to protect the Database.

### 1. Examine the network policy manifest
``` 
more 120-k8s-network-policy-yaobank.yaml
```

### 2. Deploy the K8s Network Policy
```
kubectl apply -f 120-k8s-network-policy-yaobank.yaml
```

### 3. Try the attack again
Repeat the attack commands in Step A above. This time the direct database access should fail, however, the customer frontend access from your browser should still work.

```
# Execute bash command in customer container
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer bash
```

```
curl http://database:2379/v2/keys?recursive=true | python -m json.tool
```

The curl will be blocked and return no data.  You may need to CTRL-C to terminate the command.  Then remember to exit the pod exec and return to the host terminal.
```
exit
```

## C. Global Default Deny and Allowing Authorized DNS

Lets apply a Calico GlobalNetworkPolicy to turn on Default Deny across the cluster. 
### 1. Examine the network policy manifest
```
more 130-globaldefaultdeny.yaml
```

### 2. Deploy the network policy 
```
calicoctl apply -f 130-globaldefaultdeny.yaml
```

### 3. Verify default deny is in place

In Step B above we only defined network policy for the Database. The rest of our pods should now be hitting default deny since there's no policy defined to say who they are allowed to talk to.  

Lets try to see if basic connectivity works, e.g., DNS.
```
# Execute bash command in customer container
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer bash
```
```
dig www.google.com
```
That should fail, timing out after around 15s, because we've not written any policy to say DNS is allowed. 

After running the command, remember to exit from the kubectl exec of the customer pod by typing `exit`.
```
exit
```

### 4. Allow DNS
Lets create a policy allowing DNS to cluster-internal kube-dns, and allowing kube-dns to communicate with anything.

```
calicoctl apply -f 140-globalallowdns.yaml
```

Now attempt to do DNS lookups, they should work.

```
# Execute bash command in customer container
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer bash
```
```
dig www.google.com
```

After running the command, remember to exit from the kubectl exec of the customer pod by typing `exit`.
```
exit
```

## D. Better Pod Policy

We've now defined a default deny across all of the cluster, plus allowed DNS queries to kube-dns (CoreDNS).  We also defined a policy for the `database` in step B. But we haven't yet defined policies for the `customer` or `summary` pods.

### 1. Verify cannot access Frontend

Try to access the Yaobank front end from your browser. The Yaobank app should timeout.

This is because we haven't provided detailed policy for the `customer` and `summary` pods. 

### 2. Create policy for the remaining pods

Lets create better policy for all the Yaobank services.
```
kubectl apply -f 150-more-yaobank-policy.yaml
```

### 3. Verify everything is now working
Now try accessing the Yaobank front end from the browser, it should work again. By defining the cluster wide default deny we've forced the user to follow best practices and define network policy for all the necessary microservices.

### 4. Examine the policy we just applied
```
more 150-more-yaobank-policy.yaml
```
Notice that the egress rules in 150-more-yaobank-policy.yaml are excessively liberal, allowing outbound connection to any destination. While this may be required for some workloads, in this case it is a mistake, perhaps reflecting a lazy developer. We will look at how this can be locked down further by the cluster operator or security admin in the next module.
