# IPVS mode kube-proxy demo

## Prerequisites

### Load required kernel modules

1. Ensure the ipvs kernel modules are loaded into memory.

```
lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

If you don't see them add them.

```
sudo modprobe -- ip_vs
sudo modprobe -- ip_vs_rr
sudo modprobe -- ip_vs_wrr
sudo modprobe -- ip_vs_sh
sudo modprobe -- nf_conntrack_ipv4
```

### Install ipvsadm and ipset

1. Make sure the ipvsadm and ipset command line tools are installed

```
apt-get install ipvsadm ipset -y
```

2. Initialize the cluster

```
sudo kubeadm reset -f
sudo kubeadm init --config kubeadm-config-ipvs-mode.yaml
```


## Demo

1. Check the `kube-proxy` logs and verify you're running `IPVS` mode

```
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

You should see something resembling the following output.

```
I1031 00:39:54.832710       1 server_others.go:170] Using ipvs Proxier.
W1031 00:39:54.833167       1 proxier.go:401] IPVS scheduler not specified, use rr by default
I1031 00:39:54.833601       1 server.go:534] Version: v1.15.5
I1031 00:39:54.839837       1 conntrack.go:52] Setting nf_conntrack_max to 131072
I1031 00:39:54.840077       1 config.go:96] Starting endpoints config controller
I1031 00:39:54.840229       1 controller_utils.go:1029] Waiting for caches to sync for endpoints config controller
I1031 00:39:54.840256       1 config.go:187] Starting service config controller
I1031 00:39:54.840275       1 controller_utils.go:1029] Waiting for caches to sync for service config controller
I1031 00:39:54.940431       1 controller_utils.go:1036] Caches are synced for endpoints config controller
I1031 00:39:54.940431       1 controller_utils.go:1036] Caches are synced for service config controller
```

If you are in `IPTables` mode you should see something resembling the following output.

```
I1031 00:41:34.590426       1 server_others.go:143] Using iptables Proxier.
I1031 00:41:34.590889       1 server.go:534] Version: v1.15.5
I1031 00:41:34.600889       1 conntrack.go:52] Setting nf_conntrack_max to 131072
I1031 00:41:34.604459       1 config.go:187] Starting service config controller
I1031 00:41:34.604484       1 controller_utils.go:1029] Waiting for caches to sync for service config controller
I1031 00:41:34.604523       1 config.go:96] Starting endpoints config controller
I1031 00:41:34.604533       1 controller_utils.go:1029] Waiting for caches to sync for endpoints config controller
I1031 00:41:34.704650       1 controller_utils.go:1036] Caches are synced for service config controller
I1031 00:41:34.704664       1 controller_utils.go:1036] Caches are synced for endpoints config controller
```

### Command line inspection of worker node network interfaces

1. Examine the network interfaces on the node

```
ip addr
```

From the output we can see IPVS proxier creates a dummy interface `kube-ipvs0` and binds service ip addresses to this interface.

```
...
4: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default
    link/ether c2:18:64:e5:f3:02 brd ff:ff:ff:ff:ff:ff
    inet 10.49.0.10/32 brd 10.49.0.10 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.49.0.1/32 brd 10.49.0.1 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
```

The service ips under `kube-ipvs0` will have corresponding records in the ipvs load balancing results.

### Command line inspection of IPVS proxy rules

```
kubectl get svc --all-namespaces
```

```
NAMESPACE     NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       kubernetes   ClusterIP   10.49.0.1    <none>        443/TCP                  24m
kube-system   kube-dns     ClusterIP   10.49.0.10   <none>        53/UDP,53/TCP,9153/TCP   24m
```

If IPVS mode has been successfully enabled you should see the ipvs proxy rules.

```
sudo ipvsadm -ln
```

```
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.49.0.1:443 rr
  -> 10.0.0.104:6443              Masq    1      0          0
TCP  10.49.0.10:53 rr
TCP  10.49.0.10:9153 rr
UDP  10.49.0.10:53 rr
```

* 10.49.0.1:443 refers to the kube-controller-manager service, the pod ips are master node ips
* 10.49.0.10:53 and 10.49.0.10:9153 refers to the CoreDNS service, the pod ips are pointing to the coredns pods

### Explore IPVS load balancing algorithms

Kube-proxy in IPVS mode supports multiple load balancing [algorithms](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/apis/config/types.go#L193])

* rr: round-robin
* wrr: weighted round-robbin
* lc: least connection
* wlc: weighted least connection
* lblc: locality based least connection
* lblcr: locality based least connection with replication
* sh: source hashing
* dh: destination hashing
* sed: shortest expected delay
* nq: never queue

The chosen load balancing algorithm applies to all services in the cluster and can be configured with the kube-proxy `–ipvs-scheduler` flag.

```
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
  scheduler: sh
```

1. Inspect the current schedular algorithm using `ipvsadm`.

```
sudo ipvsadm -ln
```

```
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.49.0.1:443 rr
  -> 10.0.0.104:6443              Masq    1      0          0
TCP  10.49.0.10:53 rr
TCP  10.49.0.10:9153 rr
UDP  10.49.0.10:53 rr
```

As you can see `round-robbin` is the default algorithm.

#### Session affinity (sticky sessions)

IPVS support client IP session affinity (persistent connection). When a service specifies session affinity, the IPVS proxier will set a timeout value (180min=10800s by default) in the IPVS service.

1. Example the service...

```
kubectl describe svc nginx-service
```

```
Name:			nginx-service
...
IP:			    10.102.128.4
Port:			http	3080/TCP
Session Affinity:	ClientIP
```

```
ipvsadm -ln
```

```
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.102.128.4:3080 rr persistent 10800
```
