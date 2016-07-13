# Cloud Infrastructure Provisioning

Kubernetes can be installed just about anywhere physical or virtual machines can be run. In this lab we are going to focus on [Google Cloud Platform](https://cloud.google.com/) (IaaS).

This lab will walk you through provisioning the compute instances required for running a H/A Kubernetes cluster. A total of 9 virtual machines will be created.

After completing this guide you should have the following compute instances:

```
gcloud compute instances list
```

````
+--------------------------------------+------------------------------+--------+------------+-------------+------------------------------------------------------------------------------------------------------------------+
| ID                                   | Name                         | Status | Task State | Power State | Networks                                                                                                         |
+--------------------------------------+------------------------------+--------+------------+-------------+------------------------------------------------------------------------------------------------------------------+
| 21598d90-91c9-48e6-9f49-f01e0165112a | shane-kubernetes-controller0 | ACTIVE | -          | Running     | public=104.130.155.17; private=10.209.8.207; dfw-overlay-net=10.0.2.4    |
| f49129a5-1a7f-4758-8742-842748798213 | shane-kubernetes-controller1 | ACTIVE | -          | Running     | public=104.130.155.23; private=10.209.8.211; dfw-overlay-net=10.0.2.5    |
| a6bf33d0-9825-4500-9557-5967a098b54c | shane-kubernetes-controller2 | ACTIVE | -          | Running     | public=23.253.246.81; private=10.208.233.106; dfw-overlay-net=10.0.2.6    |
| de5195d7-8bb9-412a-a28e-b2735e90aa1f | shane-kubernetes-etcd0       | ACTIVE | -          | Running     | public=104.130.124.192; private=10.209.32.30; dfw-overlay-net=10.0.2.1   |
| 4b4f8f2a-898e-4980-8db7-7ef2cf71f966 | shane-kubernetes-etcd1       | ACTIVE | -          | Running     | public=104.239.143.115; private=10.209.33.248; dfw-overlay-net=10.0.2.2  |
| 9d0d9b28-43d6-4829-9e41-b5d13cb24c69 | shane-kubernetes-etcd2       | ACTIVE | -          | Running     | public=104.130.155.5; private=10.209.8.142; dfw-overlay-net=10.0.2.3     |
| 2bb97e7a-51a2-41d1-b8d9-0d13ebe652dc | shane-kubernetes-worker0     | ACTIVE | -          | Running     | public=104.130.143.37; private=10.208.234.107; dfw-overlay-net=10.0.2.7  |
| fbe04f2a-cddc-4a9b-9deb-e5d61aac21b6 | shane-kubernetes-worker1     | ACTIVE | -          | Running     | public=104.130.143.98; private=10.208.234.119; dfw-overlay-net=10.0.2.8  |
| 542ab073-16a3-4bc7-a9ae-3bb445a1e9c4 | shane-kubernetes-worker2     | ACTIVE | -          | Running     | public=104.130.143.102; private=10.208.235.125; dfw-overlay-net=10.0.2.9 |
+--------------------------------------+------------------------------+--------+------------+-------------+------------------------------------------------------------------------------------------------------------------+
````

> All machines will be provisioned with fixed private IP addresses to simplify the bootstrap process.

To make our Kubernetes control plane remotely accessible, a public IP address will be provisioned and assigned to a Load Balancer that will sit in front of the 3 Kubernetes controllers.

## Create a Custom Network

```
gcloud compute networks create kubernetes --mode custom
```

```
NAME        MODE    IPV4_RANGE  GATEWAY_IPV4
kubernetes  custom
```

Create a subnet for the Kubernetes cluster:

```
gcloud compute networks subnets create kubernetes \
  --network kubernetes \
  --range 10.240.0.0/24 \
  --region us-central1
```

```
NAME        REGION       NETWORK     RANGE
kubernetes  us-central1  kubernetes  10.240.0.0/24
```

### Firewall Rules

```
gcloud compute firewall-rules create kubernetes-allow-icmp \
  --allow icmp \
  --network kubernetes \
  --source-ranges 0.0.0.0/0 
```

```
gcloud compute firewall-rules create kubernetes-allow-internal \
  --allow tcp:0-65535,udp:0-65535,icmp \
  --network kubernetes \
  --source-ranges 10.240.0.0/24
```

```
gcloud compute firewall-rules create kubernetes-allow-rdp \
  --allow tcp:3389 \
  --network kubernetes \
  --source-ranges 0.0.0.0/0
```

```
gcloud compute firewall-rules create kubernetes-allow-ssh \
  --allow tcp:22 \
  --network kubernetes \
  --source-ranges 0.0.0.0/0
```

```
gcloud compute firewall-rules create kubernetes-allow-healthz \
  --allow tcp:8080 \
  --network kubernetes \
  --source-ranges 130.211.0.0/22
```

```
gcloud compute firewall-rules create kubernetes-allow-api-server \
  --allow tcp:6443 \
  --network kubernetes \
  --source-ranges 0.0.0.0/0
```


```
gcloud compute firewall-rules list --filter "network=kubernetes"
```

```
NAME                         NETWORK     SRC_RANGES      RULES                         SRC_TAGS  TARGET_TAGS
kubernetes-allow-api-server  kubernetes  0.0.0.0/0       tcp:6443
kubernetes-allow-healthz     kubernetes  130.211.0.0/22  tcp:8080
kubernetes-allow-icmp        kubernetes  0.0.0.0/0       icmp
kubernetes-allow-internal    kubernetes  10.240.0.0/24   tcp:0-65535,udp:0-65535,icmp
kubernetes-allow-rdp         kubernetes  0.0.0.0/0       tcp:3389
kubernetes-allow-ssh         kubernetes  0.0.0.0/0       tcp:22
```

## Create the Kubernetes Public IP Address

Create a public IP address that will be used by remote clients to connect to the Kubernetes control plane:

```
gcloud compute addresses create kubernetes
```

```
gcloud compute addresses list kubernetes
```
```
NAME        REGION       ADDRESS          STATUS
kubernetes  us-central1  XXX.XXX.XXX.XXX  RESERVED
```

## Provision Virtual Machines

All the VMs in this lab will be provisioned using Ubuntu 16.04 mainly because it runs a newish Linux Kernel that has good support for Docker.


### etcd

```
nova boot --flavor general1-2 \
  --image 1d3ea64f-1ead-4042-8cb6-8ceb523b6149 \
  --key-name shane-dfw \
  --nic net-id=a95acf2f-4ca7-49d8-a60a-3e57613a13f0 \
  --nic net-id=00000000-0000-0000-0000-000000000000 \
  --nic net-id=11111111-1111-1111-1111-111111111111 \
  shane-kubernetes-etcd0
```

```
nova boot --flavor general1-2 \
  --image 1d3ea64f-1ead-4042-8cb6-8ceb523b6149 \
  --key-name shane-dfw \
  --nic net-id=a95acf2f-4ca7-49d8-a60a-3e57613a13f0 \
  --nic net-id=00000000-0000-0000-0000-000000000000 \
  --nic net-id=11111111-1111-1111-1111-111111111111 \
  shane-kubernetes-etcd1
```

```
nova boot --flavor general1-2 \
  --image 1d3ea64f-1ead-4042-8cb6-8ceb523b6149 \
  --key-name shane-dfw \
  --nic net-id=a95acf2f-4ca7-49d8-a60a-3e57613a13f0 \
  --nic net-id=00000000-0000-0000-0000-000000000000 \
  --nic net-id=11111111-1111-1111-1111-111111111111 \
  shane-kubernetes-etcd2
```

### Kubernetes Controllers

```
nova boot --flavor general1-2 \
  --image 1d3ea64f-1ead-4042-8cb6-8ceb523b6149 \
  --key-name shane-dfw \
  --nic net-id=a95acf2f-4ca7-49d8-a60a-3e57613a13f0 \
  --nic net-id=00000000-0000-0000-0000-000000000000 \
  --nic net-id=11111111-1111-1111-1111-111111111111 \
  shane-kubernetes-controller0
```

```
nova boot --flavor general1-2 \
  --image 1d3ea64f-1ead-4042-8cb6-8ceb523b6149 \
  --key-name shane-dfw \
  --nic net-id=a95acf2f-4ca7-49d8-a60a-3e57613a13f0 \
  --nic net-id=00000000-0000-0000-0000-000000000000 \
  --nic net-id=11111111-1111-1111-1111-111111111111 \
  shane-kubernetes-controller1
```

```
nova boot --flavor general1-2 \
  --image 1d3ea64f-1ead-4042-8cb6-8ceb523b6149 \
  --key-name shane-dfw \
  --nic net-id=a95acf2f-4ca7-49d8-a60a-3e57613a13f0 \
  --nic net-id=00000000-0000-0000-0000-000000000000 \
  --nic net-id=11111111-1111-1111-1111-111111111111 \
  shane-kubernetes-controller2
```

### Kubernetes Workers

```
nova boot --flavor general1-2 \
  --image 1d3ea64f-1ead-4042-8cb6-8ceb523b6149 \
  --key-name shane-dfw \
  --nic net-id=a95acf2f-4ca7-49d8-a60a-3e57613a13f0 \
  --nic net-id=00000000-0000-0000-0000-000000000000 \
  --nic net-id=11111111-1111-1111-1111-111111111111 \
  shane-kubernetes-worker0
```

```
nova boot --flavor general1-2 \
  --image 1d3ea64f-1ead-4042-8cb6-8ceb523b6149 \
  --key-name shane-dfw \
  --nic net-id=a95acf2f-4ca7-49d8-a60a-3e57613a13f0 \
  --nic net-id=00000000-0000-0000-0000-000000000000 \
  --nic net-id=11111111-1111-1111-1111-111111111111 \
  shane-kubernetes-worker1
```

```
nova boot --flavor general1-2 \
  --image 1d3ea64f-1ead-4042-8cb6-8ceb523b6149 \
  --key-name shane-dfw \
  --nic net-id=a95acf2f-4ca7-49d8-a60a-3e57613a13f0 \
  --nic net-id=00000000-0000-0000-0000-000000000000 \
  --nic net-id=11111111-1111-1111-1111-111111111111 \
  shane-kubernetes-worker2
```
