# CalicoCon 2019
Welcome to CalicoCon 2019!

## Agenda

| Time        | Lesson                                                                           |
|-------------|----------------------------------------------------------------------------------|
| 09:00-09:45 | [Big Picture Overview](./000-get-started/README.md)                              |
| 10:00-10:45 | [Policy Fundamentals](./100-policy-fundamentals/README.md)                       |
| 11:00-12:00 | [Policy Advanced](./200-policy-advanced/README.md)                              |
| 12:00-13:00 | Lunch                                                                            |
| 13:00-13:45 | [Pod Connectivity](./300-pod-connectivity/README.md)                             |
| 14:00-14:45 | [Pod Connectivity Advanced](./400-pod-connectivity-advanced/README.md)           |
| 15:00-15:45 | [Kubernetes Services and Calico](./500-kubernetes-services-and-calico/README.md) |
| 15:00-16:00 | [Services and Calico Advanced](./600-services-and-calico-advanced/README.md)     |

## Lab Connectivity Instructions
Each student has their own hosted lab. The lab consists of 4 hosts, one of which is a jump server you will use to access the other hosts.

### Environment

**Calico Lab Hosts:**

| IP Address   |  Node/Purpose               |
|--------------|-----------------------------|
| 10.0.0.10    | Kubernetes Master           |
| 10.0.0.11    | Kubernetes Worker 01        |
| 10.0.0.12    | Kubernetes Worker 02        |
| 10.0.0.20    | BGP Peer and Jump Server    |

**Calico Lab Networks:**

| CIDR         |  Purpose                                                  |
|--------------|-----------------------------------------------------------|
| 10.48.0.0/16 | Kubernetes Pod Network (via kubeadm `--pod-network-cidr`) |
| 10.48.0.0/24 | Calico - Initial IP Pool                                  |
| 10.48.1.0/24 | Calico - Internal IP Pool (not externally routable)       |
| 10.48.2.0/24 | Calico - External IP Pool (externally routable)           |
| 10.49.0.0/16 | Kubernetes Service CIDR (via kubeadm `--service-cidr`)    |
| 10.50.0.0/24 | Calico - External Service CIDR                            |


## Help
If you are unable to complete an exercise, are seeing different results than expected, or have a broken lab environment; we can help. Raise your hand at any time and our onsite staff will stop by to help. If onsite staff unavailable busy helping others, you can also connect with our remote Calico developers on the Calico slack channel [#calicocon-2019](https://calicousers.slack.com/archives/CQD6DSX7B).

If you do not have an account on the CalicoUsers slack, you can sign up [here](https://www.projectcalico.org/community#slack).
