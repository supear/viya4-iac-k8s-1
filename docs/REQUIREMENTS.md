# Open Source Kubernetes Infrastructure Requirements for High Availability

The items listed below are required for a Highly Available (HA) infrastructure when creating your Kubernetes cluster on physical machines, on generic Linux VMs, or in a VMware vCenter/vSphere environment. All of these items are REQUIRED as listed.

Table of Contents

- [Open Source Kubernetes Infrastructure Requirements for High Availability](#open-source-kubernetes-infrastructure-requirements-for-high-availability)
  - [Operating System](#operating-system)
  - [Machines](#machines)
    - [VMware vSphere or vCenter](#vmware-vsphere-or-vcenter)
      - [Resources](#resources)
      - [Machine Template Requirements](#machine-template-requirements)
    - [Physical Machines or Linux VMs](#physical-machines-or-linux-vms)
  - [Network](#network)
    - [CIDR Block](#cidr-block)
    - [Static IP Addresses](#static-ip-addresses)
    - [Floating IP Addresses](#floating-ip-addresses)
  - [Examples](#examples)
    - [vCenter/vSphere Sample tfvars File](#vcentervsphere-sample-tfvars-file)
    - [Physical Machine or VM Sample Inventory File](#physical-machine-or-vm-sample-inventory-file)
  - [Deployment](#deployment)
  - [Third-Party Tools](#third-party-tools)

## Operating System

An Ubuntu Linux operating system is required for the machine that uses the tools in this repository to perform the tasks associated with standing up infrastructure for a SAS Viya deployment. 

| Operating System | Description |
| --- | --- |
| [Ubuntu 22.04 LTS](https://releases.ubuntu.com/22.04/) | You must have a user account and password with privileges that enable unprompted `sudo`. You must also have a shared "private/public" SSH key pair to use with each system. These are required for the Ansible tools that are described below. |

## Machines

The following table lists the minimum machine requirements that are needed to support a Kubernetes cluster and supporting elements for a SAS Viya 4 software installation. These machines must be running a recent version of Linux and must all be on the same network. Each machine should have a DNS entry associated with its assigned IP address; this is not required, but it makes setup easier.

**NOTE**: The PostgreSQL server that is described in the following table is listed as 1 machine. If you plan to use the internal PostgreSQL server for SAS Viya, this server is not required, but you will need to adjust the capacity of your compute nodes in order to ensure that they can handle the resource requirements of the PostgreSQL processes.

| Machine | CPU | Memory | Disk | Information | Minimum Count |
| ---: | ---: | ---: | ---: | --- | ---: |
| **Control Plane** | 2 | 4 GB | 100 GB | You must have an odd number of nodes, 3 or more, in order to provide high availability (HA) for the cluster. | 1 |
| **Nodes** | xx | xx GB | xx GB | Nodes in the Kubernetes cluster. The number of machines varies, depending on multiple factors. Suggested capacities and information can be found in the sample files. | 3 |
| **Jump Server** | 4 | 8 GB | 100 GB | Bastion box that is used to access NFS mounts, share data, etc. | 1 |
| **NFS Server** | 8 | 16 GB | 500 GB | Required server that is used to store persistent volumes for the cluster. Used for providing storage for the `default` storage class in the cluster. | 1 |
| **PostgreSQL Servers** | 8 | 16 GB | 250 GB | PostgreSQL servers for your SAS Viya deployment. | 1..n |

### VMware vSphere or vCenter

In order to leverage vSphere or vCenter, the following items are required for use in your tfvars file. You also need Administrator access on vSphere or vCenter.

#### Resources

| vSphere Item | Description |
| --- | ---: |
|vsphere_cluster | Name of the vSphere cluster |
|vsphere_datacenter | Name of the vSphere data center |
|vsphere_datastore | Name of the vSphere data store to use for the VMs |
|vsphere_resource_pool | Name of the vSphere resource pool to use for the VMs |
|vsphere_folder | Name of the vSphere folder to store the VMs |
|vsphere_template | Name of the VM template to use to create VMs for the cluster |
|vsphere_network | Name of the vSphere network |

#### Machine Template Requirements

The current repository supports the provisioning of vSphere VMs. The following table describes storage-related requirements:

| Requirement | Description |
| --- | --- |
| Disk | The `root` partition `/` must be on `/dev/sd2`. |
| Hard Disk | Specify `Thin Provision` to adjust the size of the disk to match the machine requirements that were listed previously. |

### Physical Machines or Linux VMs

In order to provision physical machines for a SAS Viya deployment, you must set up ALL systems with the required elements that were listed previously. These systems must have full network access to each other.

## Network

All systems require routable connectivity to each other.

### CIDR Block

The CIDR block for your infrastructure must be able to accommodate at least the number of machines described in [Machines](#machines). In addition, the CIDR block must have the following:

- the virtual IP address for the cluster entrypoint 
- the cloud provider IP address source range that is required to support the LoadBalancer services that are created

The following section outlines recommendations for IP address assignments. All machines can have static or floating IP addresses or a combination of these.

### Static IP Addresses

Static IP addresses must be part of your network and will be assigned to the following components that are deployed:

- Kubernetes cluster virtual IP address
- Load balancer IP address
- Jump server
- NFS server
- PostgreSQL server

### Floating IP Addresses

These IP addresses are part of your network but are not assigned. Floating IP addresses are required for the following components:

- Control Plane
- Cluster nodes
- Additional load balancers that are created

## Examples

This section provides an example configuration based on the physical-machine and vSphere inventory files that are provided in this repository. You are expected to modify the inventory files to match your environment and your requirements.

### vCenter/vSphere Sample tfvars File

If you are creating virtual machines with vCenter or vSphere, the terraform.tfvars file that you create will generate the required inventory and ansible-vars.yaml files for a SAS Viya deployment using the tools in the [viya4-deployment](https://github.com/sassoftware/viya4-deployment) repository.

For this example, the network setup is as follows:

```text
CIDR Range        : 10.18.0.0/16
Virtual IP        : 10.18.0.175
Load Balancer IPs : 10.18.0.100-10.18.0.125
```

Refer to the file [terraform.tfvars](../examples/vsphere/terraform.tfvars) for more information.

```yaml
# General items
ansible_user     = "ansadmin"
ansible_password = "!Th!sSh0uldNotBUrP@ssw0rd#"
prefix           = "vm-dev"    # Infra prefix
gateway          = "10.18.0.1" # Gateway for servers
netmask          = "16"        # Netmask providing network access to your gateway

# vSphere
vsphere_cluster       = "" # Name of the vSphere cluster
vsphere_datacenter    = "" # Name of the vSphere data center
vsphere_datastore     = "" # Name of the vSphere data store to use for the VMs
vsphere_resource_pool = "" # Name of the vSphere resource pool to use for the VMs
vsphere_folder        = "" # Name of the vSphere folder to store the VMs
vsphere_template      = "" # Name of the VM template to clone to create VMs for the cluster
vsphere_network       = "" # Name of the network to to use for the VMs

# Systems
system_ssh_keys_dir = "~/.ssh" # Directory holding public keys to be used on each machine

# Kubernetes - Cluster
cluster_version        = "1.23.8"                        # Kubernetes version
cluster_cni            = "calico"                        # Kubernetes Container Network Interface (CNI)
cluster_cri            = "containerd"                    # Kubernetes Container Runtime Interface (CRI)
cluster_service_subnet = "10.35.0.0/16"                  # Kubernetes service subnet
cluster_pod_subnet     = "10.36.0.0/16"                  # Kubernetes Pod subnet
cluster_domain         = "sample.domain.foo.com"         # Cluster domain suffix for DNS

# Kubernetes - Cluster Virtual IP Address and Cloud Provider
cluster_vip_version = "0.5.5"
cluster_vip_ip      = "10.18.0.175"
cluster_vip_fqdn    = "vm-dev-oss-vip.sample.domain.foo.com"

# Kubernetes - Load Balancer

# Load Balancer Type
cluster_lb_type = "kube_vip" # Load Balancer type [kube_vip,metallb]

# Load Balancer Addresses
#
# Examples for each provider can be found here:
#
#  kube-vip address format : https://kube-vip.io/docs/usage/cloud-provider/#the-kube-vip-cloud-provider-configmap
#  metallb address format  : https://metallb.universe.tf/configuration/#layer-2-configuration
#
#    kube-vip sample:
#
#      cluster_lb_addresses = [
#        cidr-default: 192.168.0.200/29                      # CIDR-based IP range for use in the default Namespace
#        range-development: 192.168.0.210-192.168.0.219      # Range-based IP range for use in the development Namespace
#        cidr-finance: 192.168.0.220/29,192.168.0.230/29     # Multiple CIDR-based ranges for use in the finance Namespace
#        cidr-global: 192.168.0.240/29                       # CIDR-based range which can be used in any Namespace
#      ]
#
#    metallb sample:
#
#      cluster_lb_addresses = [
#        192.168.10.0/24
#        192.168.9.1-192.168.9.5
#      ]
#
cluster_lb_addresses = [
  range-global: 10.18.0.100-10.18.0.125
]

# Control plane node shared ssh key name
control_plane_ssh_key_name = "cp_ssh"

# Cluster Node Pools config
#
#   Your node pools must contain at least 3 or more nodes.
#   The required node types are:
#
#   * control_plane - Having an odd number 3/5/7... ensures
#                     HA while using kube-vip
#   * system        - System node pool to run miscellaneous pods
#   * cas           - CAS Nodes
#   * <node type>   - Any number of node types with unique names.
#                     These are typically: compute, stateful, and
#                     stateless. 
#

# NOTE: For this example ALL node pools are using DHCP to obtain their
#       IP addresses. The assumption here is that these IPs are
#       addressable via the 10.18.0.0/16 network in this example
node_pools = {
  #
  # REQUIRED NODE TYPE - DO NOT REMOVE and DO NOT CHANGE THE NAME
  #                      Other variables may be altered
  control_plane = {
    count       = 3
    cpus        = 2
    memory      = 4096
    os_disk     = 100
    node_taints = []
    node_labels = {}
  },
  #
  # REQUIRED NODE TYPE - DO NOT REMOVE and DO NOT CHANGE THE NAME
  #                      Other variables may be altered
  system = {
    count       = 1
    cpus        = 8
    memory      = 16384
    os_disk     = 100
    node_taints = []
    node_labels = {
      "kubernetes.azure.com/mode" = "system" # REQUIRED LABEL - DO NOT REMOVE
    }
  },
  cas = {
    count       = 3
    cpus        = 16
    memory      = 131072
    os_disk     = 350
    misc_disks  = [
      150,
      150,
    ]
    node_taints = ["workload.sas.com/class=cas:NoSchedule"]
    node_labels = {
      "workload.sas.com/class" = "cas"
    }
  },
  compute = {
    count       = 1
    cpus        = 16
    memory      = 131072
    os_disk     = 100
    node_taints = ["workload.sas.com/class=compute:NoSchedule"]
    node_labels = {
      "workload.sas.com/class"        = "compute"
      "launcher.sas.com/prepullImage" = "sas-programming-environment"
    }
  },
  stateful = {
    count       = 1
    cpus        = 8
    memory      = 32768
    os_disk     = 100
    misc_disks  = [
      150,
    ]
    node_taints = ["workload.sas.com/class=stateful:NoSchedule"]
    node_labels = {
      "workload.sas.com/class" = "stateful"
    }
  },
  stateless = {
    count       = 2
    cpus        = 8
    memory      = 32768
    os_disk     = 100
    misc_disks  = [
      150,
    ]
    node_taints = ["workload.sas.com/class=stateless:NoSchedule"]
    node_labels = {
      "workload.sas.com/class" = "stateless"
    }
  },
  singlestore = {
    cpus    = 16
    memory  = 131072
    os_disk = 100
    misc_disks = [
      250,
      250,
      500,
      500,
    ]
    count       = 3
    node_taints = ["workload.sas.com/class=singlestore:NoSchedule"]
    node_labels = {
      "workload.sas.com/class" = "singlestore"
    }
  },
}

# Jump server
#
#   Suggested server specs are shown below:
#
create_jump    = true         # Creation flag
jump_num_cpu   = 4            # 4 CPUs
jump_memory    = 8092         # 8 GB
jump_disk_size = 100          # 100 GB
jump_ip        = "10.18.0.11" # Assigned values for static IPs

# NFS server
#
#   Suggested server specs shown below.
#
create_nfs    = true         # Creation flag
nfs_num_cpu   = 8            # 8 CPUs
nfs_memory    = 16384        # 16 GB
nfs_disk_size = 500          # 500 GB
nfs_ip        = "10.18.0.12" # Assigned values for static IPs

# Container Registry
create_cr    = false         # Creation flag
cr_num_cpu   = 4             # 4 CPUs
cr_memory    = 8092          # 8 GB
cr_disk_size = 250           # 250 GB
cr_ip        = "10.18.0.13"  # Assigned values for static IPs

# PostgreSQL server
#
#   Suggested server specs shown below.
#
postgres_servers = {
  default = {
    server_num_cpu         = 8                       # 8 CPUs
    server_memory          = 16384                   # 16 GB
    server_disk_size       = 250                     # 256 GB
    server_ip              = "10.18.0.14"            # Assigned values for static IPs
    server_version         = 13                      # PostgreSQL version
    server_ssl             = "off"                   # SSL flag
    administrator_login    = "postgres"              # PostgreSQL admin user - CANNOT BE CHANGED
    administrator_password = "my$up3rS3cretPassw0rd" # PostgreSQL admin user password
  }
}
```

### Physical Machine or VM Sample Inventory File

With this example, because you are using physical (bare-metal) machines or generic Linux VMs, you must update the inventory file and the ansible-vars.yaml file for your environment, using the settings provided below as examples.

This example is using the `192.168.0.0/16` CIDR block for the cluster. The cluster prefix is `viya4-oss`. The cluster virtual IP address is `192.168.0.1`.

Refer to the [inventory file](../examples/bare-metal/inventory) for more information.

```yaml
#
# Kubernetes - Control Plane nodes
#
# This list is the FQDN/IP of the nodes used for the control plane
#
# NOTE: For HA/kube-vip to work you need at least 3 nodes
#
[k8s_control_plane]
192.168.1.0
192.168.1.1
192.168.1.2

#
# Kubernetes - Compute nodes
#
# This list is the FQDN/IP of the nodes used for the compute nodes
#
# NOTE: For HA to work you need at least 3 nodes
#
[k8s_node]
192.168.2.0
192.168.2.1
192.168.2.2
192.168.2.3
192.168.2.4
192.168.2.5

#
# Kubernetes Nodes - alias - DO NOT MODIFY
#
[k8s:children]
k8s_control_plane
k8s_node

#
# Jump Server
#
[jump_server]
192.168.3.0

#
# Jump Server - alias - DO NOT MODIFY
#
[jump:children]
jump_server

#
# NFS Server
#
[nfs_server]
192.168.4.0

#
# NFS Server - alias - DO NOT MODIFY
#
[nfs:children]
nfs_server

#
# PostgreSQL Servers
#
# NOTE: You MUST have an entry for each PostgreSQL server
#
[viya4_oss_default_pgsql]
192.168.5.0
[viya4_oss_default_pgsql:vars]
postgres_server_version=12
postgres_server_ssl=off                 # NOTE: Values - [on,off]
postgres_administrator_login="postgres" # NOTE: Do not change this value at this time
postgres_administrator_password="Un33d2ChgM3n0W!"

# NOTE: Add entries here for each PostgreSQL server listed previously
[postgres:children]
viya4_oss_default_pgsql

#
# All systems
#
[all:children]
k8s
jump
nfs
postgres
```

Refer to the [ansible-vars.yaml](../examples/bare-metal/sample-ansible-vars.yaml) file for more information.

```yaml
# Ansible items
ansible_user     : ""
ansible_password : ""

# VM items
vm_os   : "ubuntu" # Choices : [ubuntu|rhel] - Ubuntu 22.04 LTS / Red Hat Enterprise Linux ???
vm_arch : "amd64"  # Choices : [amd64] - 64-bit OS / ???

# System items
enable_cgroup_v2    : true     # TODO - If needed hookup or remove flag
system_ssh_keys_dir : "~/.ssh" # Directory holding public keys to be used on each system

# Generic items
prefix          : ""
deployment_type : ""

# Kubernetes - Common
#
# TODO: kubernetes_upgrade_allowed needs to be implemented to either
#       add or remove locks on the kubeadm, kubelet, kubectl packages
#
kubernetes_cluster_name    : "{{ prefix }}-oss" # NOTE: only change the prefix value above
kubernetes_version         : ""
kubernetes_upgrade_allowed : true
kubernetes_arch            : "{{ vm_arch }}"
kubernetes_cni             : "calico"           # Choices : [calico]
kubernetes_cri             : "containerd"       # Choices : [containerd|docker|cri-o] NOTE: cri-o is not currently functional
kubernetes_service_subnet  : ""
kubernetes_pod_subnet      : ""

#
# Kubernetes - VIP : https://kube-vip.io
# 
# Useful links:
#
#   VIP IP : https://kube-vip.io/docs/installation/static/
#   VIP Cloud Provider IP Range : https://kube-vip.io/docs/usage/cloud-provider/#the-kube-vip-cloud-provider-configmap
#
kubernetes_vip_version              : "0.5.5"
kubernetes_vip_ip                   : ""
kubernetes_vip_fqdn                 : ""

# Kubernetes - Load Balancer

#
# Load Balancer Type
#
kubernetes_loadbalancer : "kube_vip" # Load Balancer accepted values [kube_vip,metallb]

#
# Load Balancer Addresses
#
# Examples for each load balancer type can be found here:
#
#  kube-vip address format : https://kube-vip.io/docs/usage/cloud-provider/#the-kube-vip-cloud-provider-configmap
#  metallb address format  : https://metallb.universe.tf/configuration/#layer-2-configuration
#
#    kube-vip sample:
#
#      kubernetes_loadbalancer_addresses :
#        - "cidr-default: 192.168.0.200/29"                  # CIDR-based IP range for use in the default Namespace
#        - "range-development: 192.168.0.210-192.168.0.219"  # Range-based IP range for use in the development Namespace
#        - "cidr-finance: 192.168.0.220/29,192.168.0.230/29" # Multiple CIDR-based ranges for use in the finance Namespace
#        - "cidr-global: 192.168.0.240/29"                   # CIDR-based range which can be used in any Namespace
#
#    metallb sample:
#
#      kubernetes_loadbalancer_addresses :
#        - "192.168.10.0/24"
#        - "192.168.9.1-192.168.9.5"
#
#  NOTE: If you are assigning a static IP using the loadBalancerIP value for your 
#        load balancer controller service when using `metallb` that IP must fall
#        within the address range you provide below. If you are using `kube_vip`
#        you do not have this limitation.
#
kubernetes_loadbalancer_addresses : []

# Kubernetes - Control Plane
control_plane_ssh_key_name : "cp_ssh"

# Labels/Taints
#
#   The label names match the host names to apply these items.
#   If the node names do not match, you'll have to apply these
#   taints/labels manually.
#
#   The format for the label block is:
#
#       node_labels:
#         <node name pattern>:
#           - <label>
#           - <label>
#
#       The format for the taint block is:
#
#       node_taints:
#         <node name pattern>:
#           - <taint>
#           - <taint>
#
#   NOTE: There are no quotes around the label and taint elements
#         These are literal converted to strings when applying
#         into the cluster
#   

## Labels
node_labels:
  cas:
    - workload.sas.com/class=cas
  compute:
    - launcher.sas.com/prepullImage=sas-programming-environment
    - workload.sas.com/class=compute
  singlestore:
    - workload.sas.com/class=singlestore
  stateful:
    - workload.sas.com/class=stateful
  stateless:
    - workload.sas.com/class=stateless
  system:
    - kubernetes.azure.com/mode=system

## Taints
node_taints:
  cas:
    - workload.sas.com/class=cas:NoSchedule
  compute:
    - workload.sas.com/class=compute:NoSchedule
  singlestore:
    - workload.sas.com/class=singlestore:NoSchedule
  stateful:
    - workload.sas.com/class=stateful:NoSchedule
  stateless:
    - workload.sas.com/class=stateless:NoSchedule

# Jump Server
jump_ip : ""

# NFS Server
nfs_ip  : ""

# Container Registry
cr_ip   : ""

# PostgreSQL Servers
```

## Deployment

The following items **MUST** be added to your ansible-vars.yaml file if you are using the [viya4-deployment](https://github.com/sassoftware/viya4-deployment.git) repository to deploy SAS Viya:

```yaml
## 3rd Party

### Ingress Controller
INGRESS_NGINX_CONFIG:
  controller:
    service:
      externalTrafficPolicy: Cluster
      # loadBalancerIP: <your static ip> # Assigns a specific IP for your loadBalancer
      loadBalancerSourceRanges: [] # Not supported on open source kubernetes - https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/
      annotations:

### Metrics Server
METRICS_SERVER_CHART_VERSION: 5.10.14
METRICS_SERVER_CONFIG:
  apiService:
    create: true
  extraArgs:
    kubelet-insecure-tls: true
    kubelet-preferred-address-types: InternalIP,ExternalIP,Hostname,InternalDNS,ExternalDNS
    kubelet-use-node-status-port: true
    requestheader-allowed-names: aggregator
    metric-resolution: 15s
    cert-dir: /tmp
  service:
    labels:
      kubernetes.io/cluster-service: "true"
      kubernetes.io/name: "Metrics-server"

### NFS Subdir External Provisioner - SAS default storage class
# Updates to support open source Kubernetes 
NFS_CLIENT_NAME: nfs-subdir-external-provisioner-sas
NFS_CLIENT_CHART_VERSION: 4.0.17
```

## Third-Party Tools

The third-party applications that are listed in the following table are supported for the deployment of SAS Viya into a cluster configured by viya4-iac-kubernetes:

| Application | Minimum Version |
| ---: | ---: |
| [Ansible](https://www.ansible.com/) | Core 2.13.4 |
| [Terraform](https://www.terraform.io/) |1.3.2 |
| [Docker](https://www.docker.com/) | 20.10.17 |
| [Helm](https://helm.sh/) | 3.10.0 |
