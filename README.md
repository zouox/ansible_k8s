# Deploy a k8s cluster with Ansible

Maintainers: [zouox](https://gitlab.com/zouox), [ipefix ledruide](https://gitlab.com/ipefixledruide)

Supported distributions:

- Debian 12
- RHEL Family 8+ (Almalinux, Rocky Linux, ...)

Supported architecture:

- x64

This role automates the setup of Kubernetes with integration of essential components such as an ingress controller (Traefik or Nginx), a storage class (local or NFS), and a Virtual IP (kube-vip) for high availability and load balancing.

> ⚠️ All issues must be created on the [project main repository](https://gitlab.com/zouox-projects/ansible/zouox.ansible_k8s).

## Requirements

- Root access (or a user that can sudo) on all managed nodes in inventory

- The kubernetes.core modules collection must be installed on the Ansible controller :

```bash
ansible-galaxy collection install kubernetes.core
```

- The community.general modules collection must be installed on the Ansible controller :

```bash
ansible-galaxy collection install community.general
```

## Usage

### Preparation

Create an inventory for the cluster. **Don't change the groups names as this role use fixed groups names** (for now, as soon as we find a suitable solution)

```yaml
all:
  children:
    k8s_main_master:
      hosts:
        k8s-cp-1:
          ansible_host: 192.168.10.51
    k8s_other_masters:
      hosts:
        k8s-cp-2:
          ansible_host: 192.168.10.52
        k8s-cp-3:
          ansible_host: 192.168.10.53
    k8s_workers:
      hosts:
        k8s-worker-1:
          ansible_host: 192.168.10.61
```

### Variables

|Name |Description  |Default  |Required |
|---|---|---|---|
|set_hostnames  |Automatically sets hostnames for target hosts, based on Ansible inventory name. |True |No  |
|kubernetes_version  |Sets Kubernetes version. |v1.30 |No  |
|pod_network_cidr  |Sets the pod network CIDR for the cluster|10.244.0.0/16 |No  |
|kubernetes_cni  |Sets the installed CNI. Options are: flannel, calico, false |flannel |No  |
|calico_version  |Sets the Calico version to download. |v3.28.0 |No |
|controlplane_endpoint  |Sets the control plane endpoint. Can be a domain or an IP address. |ansible_hostname |No  |
|kubevip_install |Install kube-vip on the cluser. Implies HA cluster installation. Set to **False** for single node installation.  |True |No |
|kubernetes_ingress |Sets the installed Ingress Controller. Options are: traefik, nginx, false |traefik |No  |
|kubernetes_storageclass |Sets the installed Storage Class. Options are: local, nfs, false |local |No  |
|nfs_server_address  |The IP address of the NFS server for the NFS StorageClass. |None |Yes, if *kubernetes_storageclass* var is set to **nfs** |
|nfs_server_path  |The path on the NFS server for the NFS StorageClass. |None |Yes, if *kubernetes_storageclass* var is set to **nfs** |
|vip_address  |The IP address kube-vip will advertise. |None |Yes, if *kubevip_install* var is set to **true** |
|vip_interface  |The interface kube-vip will use to advertise the virtual IP. |ansible_facts.default_ipv4.interface  |No |

### Ansible tags

|Name |Description  |
|---|---|
|install  |Runs every tasks. Use this tag for a full installation |
|prepare  |Runs tasks related to nodes preparations. |
|k8s  |Runs every k8s related tasks (such as joining members to cluster)  |
|main_master  |Runs every k8s tasks relatives to the installation of the first control plane node  |
|other_master  |Runs every k8s tasks relatives to the installation of the supplementary control plane nodes |
|worker |Runs every k8s tasks relatives to the installation of the worker nodes |

### Installation

```bash
# For your own sake
export ANSIBLE_HOST_KEY_CHECKING=False
```

#### In High Availability mode

##### Full installation

```yaml
ansible-playbook -i /path/to/inventory --user <ansible_user> --private-key </path/to/ansible_user_private_key> -t install <playbook_name>.yml
```

##### Join new members to cluster

```yaml
# Add new control planes
ansible-playbook -i /path/to/inventory --user <ansible_user> --private-key </path/to/ansible_user_private_key> -t other_master <playbook_name>.yml
```

```yaml
# Add new worker nodes
ansible-playbook -i /path/to/inventory --user <ansible_user> --private-key </path/to/ansible_user_private_key> -t worker <playbook_name>.yml
```

#### In single node mode

```yaml
ansible-playbook -i /path/to/inventory --user <ansible_user> --private-key </path/to/ansible_user_private_key> -t install --skip-tags worker, other_master <playbook_name>.yml
```
