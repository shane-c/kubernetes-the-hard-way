# Cloud Infrastructure Provisioning

Kubernetes can be installed just about anywhere physical or virtual machines can be run. In this lab we are going to focus on [Google Cloud Platform](https://cloud.google.com/) (IaaS).

This lab will walk you through provisioning the compute instances required for running a H/A Kubernetes cluster. A total of 9 virtual machines will be created.

After completing this guide you should have the following compute instances:

```
gcloud compute instances list
```

````
NAME         ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP      STATUS
controller0  us-central1-f  n1-standard-1               10.240.0.20  XXX.XXX.XXX.XXX  RUNNING
controller1  us-central1-f  n1-standard-1               10.240.0.21  XXX.XXX.XXX.XXX  RUNNING
controller2  us-central1-f  n1-standard-1               10.240.0.22  XXX.XXX.XXX.XXX  RUNNING
etcd0        us-central1-f  n1-standard-1               10.240.0.10  XXX.XXX.XXX.XXX  RUNNING
etcd1        us-central1-f  n1-standard-1               10.240.0.11  XXX.XXX.XXX.XXX  RUNNING
etcd2        us-central1-f  n1-standard-1               10.240.0.12  XXX.XXX.XXX.XXX  RUNNING
worker0      us-central1-f  n1-standard-1               10.240.0.30  XXX.XXX.XXX.XXX  RUNNING
worker1      us-central1-f  n1-standard-1               10.240.0.31  XXX.XXX.XXX.XXX  RUNNING
worker2      us-central1-f  n1-standard-1               10.240.0.32  XXX.XXX.XXX.XXX  RUNNING
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
