# Table of Contents
This is a list of points that will be explained in this instructions file for the IKE project :

- [High-level Architecture](#high-level-architecture)
- [Prerequisites](#prerequisites)
- [Nodes Setup](#nodes-setup)
- [K8S Cluster Configuration](#k8s-cluster-configuration)
- [IKE Parameters](#ike-parameters)
- [Kubernetes deployment](#kubernetes-deployment)
- [Manage ETCD Cluster](./manage_etcd.md)
- [Create Pod](#create-pod)


# High-level Architecture

Below a diagram of the high-level architecture deployed by IKE :
![Architecture](../images/AgoraKube_diagram.png)

**Notes :** This distibution is aimed to be customizable so you can choose : 
 - Where the **etcd** will be deployed (with the master or not) 
 - The number of **master** nodes to deploy (from 1 to many - 5 nodes for production)
 - The number of **etcd** nodes to deploy (from 1 to many - 5 nodes for production)
 - The number of **worker** nodes to deploy (from 1 to many)
 - The number of **storage** nodes to deploy (from 0 to many - 3 nodes for production needs)
 
 # Prerequisites

This section explains what are the prerequisites to install IKE in your environment.

## OS

Below the OS currently supported on all the machines :
  - Ubuntu 18.04 & 20.04 - amd64
  - Centos 7 - amd64
  - Debian 10 - amd64

## Network

- Full network connectivity between all machines in the cluster (public or private network is fine)
- Full internet access
- Unique hostname, MAC address, and product_uuid for every node. See here for more [details](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#verify-the-mac-address-and-product-uuid-are-unique-for-every-node).
- Certain ports are open on your machines. See here for more [details](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports).

## Node Sizing

Node sizing indicated here is for production environment. You can custom it according to sweet your needs.

ETCD and MASTER are actually installed on the same Host, but it is a best-practice to install them on separate hosts.

| IKE Type | no HA or all-in-one | no-production | production |
| --- | --- | --- | --- |
| MASTER | 1 | 3 | 5 |
| ETCD | 1 | 3 | 5 |
| WORKER | 1 | X | X |
| STORAGE | 0 - 1 | 3 | 3+ |

We actually configure the proper VM size for your master depending on the number of nodes (Workers + Storage) in your cluster

| nodes | Master Size |
| --- | --- |
| 1-5 | 1 CPU - 3,75 Go RAM |
| 6-10 | 2 CPU - 7,50 Go RAM |
| 11-100 | 4 CPU - 15 Go RAM |
| 101-250 | 8 CPU - 30 Go RAM |
| 251-500 | 16 CPU - 60 Go RAM |
| more than 500 | 32 CPU - 120 Go RAM |

# Nodes Setup

This section explains how to setup notes before deploying Kubernetes Clusters with IKE.

## Deployment node

The deployment node is an Ansible server which contains all Ansible roles and variables used to deploy and configure Kubernetes Clusters with IKE distribution.

The prerequisites are:
- SSH Server (like openssh-server)
- Python3 & pip3
- Ansible
- netaddr
- git
- curl

Then clone or download the git branch / release you want to use.

You can run the following command to automatically install those packages and clone the latest stable IKE distribution:
```
bash <(curl -s https://raw.githubusercontent.com/ilkilab/ike-core/master/setup-deploy.sh)
```


## K8S nodes

The K8S nodes will host all the components needed for a Kubernetes cluster Control and Data planes.

The prerequisites are:
- SSH Server (like openssh-server)
- Python3
- curl

You can run the following command to automatically install those packages :
```
bash <(curl -s https://raw.githubusercontent.com/ilkilab/ike-core/master/setup-hosts.sh)
```

## SSH keys creation

IKE is using Ansible to deploy Kubernetes. You have to configure SSH keys to ensure the communication between the deploy machine and the others.

On the deploy machine, create the SSH keys :
```
ssh-keygen
```
You can let everything by default.

When your keys are created, you have to copy the public key in the other machine in the folder /home/yourUser/.ssh/authorized_keys, or you can use the following commands to copy the key :
```
ssh-copy-id -i .ssh/id_rsa.pub yourUser@IP_OF_THE_HOST
```
You have to execute this command for each node of your cluster

Once your ssh keys have been pushed to all nodes, modify the file "ike/hosts" to add the user/ssh-key (in section **SSH Connection settings**) that IKE will use to connect to all nodes

# K8S Cluster Configuration

IKE enables an easy way to deploy and manage customizable K8S clusters.

## Inventory file

The first file to modify is ["./hosts"](../hosts). This file contains all architecture information about your K8S Cluster.

**All K8S servers names must be filled in by their hostname**. You can run ```hostname -s``` to get it.

The next Sample deploys K8S components in HA mode on 6 nodes (3 **etcd/masters** nodes, 3 **workers** nodes) :

```
[deploy]
worker1 ansible_connection=local

[masters]
worker1  ansible_host=10.10.20.4

[etcd]
worker1  ansible_host=10.10.20.4

[workers]
worker2  ansible_host=10.10.20.5
worker3  ansible_host=10.10.20.6

[storage]
worker4 ansible_host=10.10.20.20

[all:vars]
advertise_masters=10.10.20.40
#advertise_masters=kubernetes.localcluster.lan

# SSH connection settings
ansible_ssh_extra_args='-o StrictHostKeyChecking=no'
ansible_user=vagrant
ansible_ssh_private_key_file=/home/vagrant/ssh-private-key.pem

[etc_hosts]
#kubernetes.localcluster.lan ansible_host=10.10.20.4
```

The **deploy** section contains information about how to connect to the deployment machine.

The **etcd** section contains information about the etcd machine(s) instances.

The **masters** section contains information about the masters nodes (K8S Control Plane).

The **workers** section contains information about the workers nodes (K8S Data Plane).

The **storage** section contains information about the storage nodes (K8S Storage Plane ).

The **etc_hosts** section contains a list of DNS entries that will be injected to /etc/hosts files of all hosts. Use it only if you don't have DNS server.

The **all:vars** section contains information about how to connect to K8S nodes.

The **advertise_masters** parameter configure the Advertising IP of control Plan. Actually it is the IP of a frontal LB that expose Master nodes on port TCP/6643. It can also be a Master's IP if you don't have LB. In this case, HA is not enabled even if you got multiple Masters...

The **SSH Connection settings** section contain information about the SSH connexion. You have to modify the variable **ansible_ssh_private_key_file** with the path where your public key is stored.
**ansible_user** User used as service account by Agorakube to connect to all nodes. **User must be sudoer**.

## Configuration file

The [../group_vars/all.yaml](../group_vars/all.yaml) file contains all configuration variables that you can customize to make your K8S Cluster fit your needs.

Sample file will deploy **containerd** as container runtime, **calico** as CNI plugin and enable all IKE features (storage, dashboard, monitoring, LB, ingress, ....).

```
---
ike:
  global:
    data_path: /var/ike

ike_pki:
  infos:
    state: "Ile-De-France"
    locality: "Paris"
    country: "FR"
    root_cn: "ILKI Kubernetes Engine"
    expirity: "+3650d"
  management:
    rotate_certificats: false

ike_base_components:
  etcd:
    release: v3.4.14
    update: false
    check: true
    data_path: /var/lib/etcd
  kubernetes:
    release: v1.19.4
    update: false
  container:
    engine: containerd
# release : Only Supported if container engine is set to docker
    release: ""
#    update: false

ike_network:
  cni_plugin: calico
  mtu: 0
  cidr:
    pod: 10.33.0.0/16
    service: 10.32.0.0/24
  service_ip:
    kubernetes: 10.32.0.1 
    coredns: 10.32.0.10
  nodeport:
    range: 30000-32000
  external_loadbalancing:
    enabled: True
    ip_range: 10.10.20.50-10.10.20.250
    secret_key: LGyt2l9XftOxEUIeFf2w0eCM7KjyQdkHform0gldYBKMORWkfQIsfXW0sQlo1VjJBB17shY5RtLg0klDNqNq4PAhNaub+olSka61LxV73KN2VaJY/snrZmHbdf/a7DfdzaeQ5pzP6D5O7zbUZwfb5ASOhNrG8aDMY3rkf4ZzHkc=
  kube_proxy:
    mode: ipvs
    algorithm: rr

ike_features:
  storage:
    enabled: true
    jiva:
      data_path: /var/openebs
      fs_type: ext4
    hostpath:
      data_path: /var/local-hostpath
  dashboard:
    enabled: true
    generate_admin_token: true
  metrics_server:
    enabled: true
  ingress:
    controller: nginx
    release: v0.41.2
  monitoring:
    enabled: true
    persistent: true
    admin:
      user: administrator
      password: P@ssw0rd

ike_populate_etc_hosts: True

# Security
ike_encrypt_etcd_keys:
# Warrning: If multiple keys are defined ONLY LAST KEY is used for encrypt and decrypt.
# Other keys are used only for decrypt purpose. Keys can be generated with command: head -c 32 /dev/urandom | base64
  key1:
    secret: 1fJcKt6vBxMt+AkBanoaxFF2O6ytHIkETNgQWv4b/+Q=

#restoration_snapshot_file: /path/snopshot/file Located on {{ etcd_data_directory }}

```

**Note :** You can also modify the IPs-CIDR if you want.

# IKE Parameters

Below  you can find all the parameters you can use in this file, section by section.

## Global Section

This section is used to custom global IKE settings.

| Parameter | Description | Values |
| --- | --- | --- |
| `ike.global.data_path` | Path where IKE saves all config/pik/service files on deploy machine | **/var/ike/** *(default)* |

## Certificates & PKI section

This section is used to custom the PKI used for your deployment and manage Certificates lifecycle.

| Parameter | Description | Values |
| --- | --- | --- |
| `ike_pki.infos.state` | State or province name added to PKI CSR | **Ile-De-France** *(default)* |
| `ike_pki.infos.locality` | Locality added to PKI CSR | **Paris** *(default)* |
| `ike_pki.infos.country` | Country added to PKI CSR | **FR** *(default)* |
| `ike_pki.infos.root_cn` | CommonName used for Root CA | **ILKI Kubernetes Engine** *(default)* |
| `ike_pki.infos.expirity` | Expirity for all PKI certificats | **+3650d** (default - 10 years)|
| `ike_pki.management.rotate_certificats` | Boolean used to rotate certificates | **False** (default)|

## Main K8S Components Section

This section is used to custom K8S main components that will be deployed.

### ETCD

This section allows you to configure your ETCD deployment.

| Parameter | Description | Values |
| --- | --- | --- |
| `ike_base_components.etcd.release` | ETCD release that will be installed on etcd hosts | **v3.4.14** *(default)* |
| `ike_base_components.etcd.update` | Update current ETCD release to `ike_base_components.etcd.release` | **False** *(default)* |
| `ike_base_components.etcd.check` | Check ETCD cluster Status/Size/Health/Leader when running ike run | **True** *(default)* |
| `ike_base_components.etcd.data_path` | Path where ETCD save data on ETCD hosts | **/var/lib/etcd** *(default)* |

### Kubernetes

This section allows you to configure your Kubernetes deployment.

| Parameter | Description | Values |
| --- | --- | --- |
| `ike_base_components.kubernetes.release` | Kubernetes release that will be installed on *Master/Worker/Storage* hosts |  **v1.19.4** *(default)* |
| `ike_base_components.kubernetes.update` | Update current Kubernetes release to `ike_base_components.kubernetes.release` | **False** *(default)* |

### Container Engine

This section allows you to configure your Container Engine taht will be deployed on all Master/Worker/Storage hosts.

| Parameter | Description | Values |
| --- | --- | --- |
| `ike_base_components.container.engine`  | Container Engine to install (Containerd or Docker) on all Master/Worker/Storage hosts |  **containerd** *(default)*, or docker |
| `ike_base_components.container.release` | Release of Container Engine to install - Supported only if `ike_base_components.container.engine` set to *docker*  | If **""** install latest release *(default)* |
| `ike_base_components.container.update` | Update current Container Engine release to `ike_base_components.container.release` | **Will be available soon** (No effect) |

## Network Settings

This section allows you to configure your K8S cluster network settings.

| Parameter | Description | Values |
| --- | --- | --- |
| `ike_network.cni_plugin` | CNI plugin used to enable K8S hosts Networking | **calico** *(default)*, kube-router |
| `ike_network.mtu` | MTU for CNI plugin. Auto-MTU if set to **0**. Only used if `ike_network.cni_plugin` is set to **calico** | **0** *(default)* |
| `ike_network.cidr.pod` | PODs CIDR network | **10.33.0.0/16** *(default)* |
| `ike_network.cidr.service` | Service CIDR network | **10.32.0.0/24** *(default)* |
| `ike_network.service_ip.kubernetes` | ClusterIP of *default.kubernetes* service. Should be the first IP available in `ike_network.cidr.service` | **10.32.0.1** *(default)* |
| `ike_network.service_ip.coredns` | ClusterIP of *kube-system.kube-dns* service. | **10.32.0.10** *(default)* |
| `ike_network.nodeport.range` | Range of allowed ports usable by NodePort services | **30000-32000** *(default)* |
| `ike_network.external_loadbalancing.enabled` | Enable External LoadBalancing in ARP mode. Working only if On-Prem deployments | **False** *(default)* |
| `ike_network.external_loadbalancing.ip_range` | IPs Range, or CIDR used by External LoadBalancer to assign External IPs  | **10.10.20.50-10.10.20.250** *(default range)* |
| `ike_network.external_loadbalancing.secret_key` | Security Key : Generate a custom key with : `openssl rand -base64 128` | **a default insecure key** *(Change it !)* |
| `ike_network.kube_proxy.mode` | Kube-Proxy mode. iptables/ipvs. IPVS > IPTABLES | **ipvs** *(default)* |
| `ike_network.kube_proxy.algorithm` | Default ClusterIP loadBalancing Algorithm : rr,lc,dh,sh,sed,nq. Only supported if IPVS | **rr** *(default Round-Robin)* |


## IKE features

This section allows you to configure your K8S features.

| Parameter | Description | Values |
| --- | --- | --- |
| `ike_features.storage.enabled` | Enable Storage feature - OpenEBS based | **False** *(default)* |
| `ike_features.storage.jiva.data_path` | Path where OpenEBS store Jiva volumes on Storage Nodes | **/var/openebse** *(default)* |
| `ike_features.storage.jiva.fs_type` | Jiva FS types | **ext4** *(default)* |
| `ike_features.storage.hostpath.data_path` | Path where OpenEBS store HostPath volumes on Pod node | **False** *(default)* |
| `ike_features.dashboard.enabled` | Enable Kubernetes dashboard | **False** *(default)* |
| `ike_features.dashboard.generate_admin_token` | Generate a default admin user + save token to /root/.kube/dashboardamin on Deploy node | **False** *(default)* |
| `ike_features.metrics_server.enabled` | Enable Metrics-Server | **False** *(default)* |
| `ike_features.ingress.controller` | Ingress Controller to install : nginx, ha-proxy, traefik | **nginx** *(default)* |
| `ike_features.ingress.release` | Ingress controller release to install. Only used if `ike_features.ingress.controller` set to "nginx" | **False** *(default)* |
| `ike_features.monitoring.enabled` | Enable Monitoring | **False** *(default)* |
| `ike_features.monitoring.persistent` | Persist Monitoring Data | **False** *(default)* |
| `ike_features.monitoring.admin.user` | Default Grafana admin user | **administrator** *(default)* |
| `ike_features.monitoring.admin.password` | Default grafana admin password | **P@ssw0rd** *(default)* |

## IKE other settings
This section allows you to configure some other settings

| Parameter | Description | Values |
| --- | --- | --- |
| `ike_populate_etc_hosts` | Add to all hostname/IPs of IKE Cluster to /etc/hosts file of all hosts. Use it only if you don't have DNS server. | **False** *(default)* |
| `ike_encrypt_etcd_keys` | Array of keys/algorith used to crypt/decrypt data in etcd? Generate with : `head -c 32 /dev/urandom | base64` | **changeME !** *(default)* |
| `restoration_snapshot_file` | ETCD backup path to be restored | **none** *(default)* |

# Kubernetes deployment

Once all configuration files are set, run the following command to launch the Ansible playbook that will deploy the pre-configured Kubernetes cluster :

```
sudo ansible-playbook ike-core.yaml
```

# Create pod<a name="create-pod" />

After the pre-configured Kubernetes cluster is deployed, run the following command to deploy a sample Kubernetes pod with the busybox image:

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep
  namespace: default
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000"
EOF
```

You should see an output similar to this:

```
pod/busybox-sleep created
```

Run the following command to verify if the deployed pod is running:

```
kubectl get pods
```


