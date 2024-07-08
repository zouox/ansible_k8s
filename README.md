# Deploy a k8s cluster with Ansible

Maintainers: [zouox](https://gitlab.com/zouox), [ipefix ledruide](https://gitlab.com/ipefixledruide)

Supported distributions:

- Debian 11
- RHEL Family 7+ (CentOS, Almalinux, ...)

Supported architecture:

- x64

## Requirements

All managed nodes in inventory must have:

- Root access (or a user with equivalent permissions)

The kubernetes.core modules collection must be installed on the Ansible controller :
```bash
ansible-galaxy collection install kubernetes.core
```

The community.general modules collection must be installed on the Ansible controller :
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
|pod_network_cidr  |Sets the pod network CIDR for the cluster|10.10.0.0/16 |No  |
|kubevip_install |Install kube-vip on the cluser. Implies HA cluster installation. Set to **False** for single node installation.  |True |No |
|helm_install |Install Helm after the cluster installation.  |True |No  |
|path_to_sourcefile |Path to the file containing extra k8s install vars| /tmp/k8s-extras.  |No  |
|path_to_k8s_script |Path where the k8s install script will be downloaded to.  |/tmp/install-k8s.sh  |No |
|path_to_manifests |Path to the directory the templated manifests will be created, **without trailing slash**. |/var/lib/rancher/k8s/server/manifests  |No |
|vip_address  |The IP address kube-vip will advertise. |None |Yes, if *kubevip_install* var is set to **true** |
|vip_interface  |The interface kube-vip will use to advertise the virtual IP. |ansible_facts.default_ipv4.interface  |No |
|k8s_install_env_vars |Dictionnary of extra arguments to pass to k8s install script. |See explaination below.  |No |

### k8s Extra args

When using this role, you can set k8s installation environment variables in a dictionnary inside your Ansible variables.

The only exception is **INSTALL_k8s_EXEC**, as this env var is already set by the role when needed.

Here is an example of how to set up this dictionnary :

```yaml
k8s_install_env_vars:
  INSTALL_k8s_NAME: my-k8s
  INSTALL_k8s_BIN_DIR: /usr/bin
```

For a full list of these environment variables, see <https://docs.k8s.io/reference/env-variables>.

### Ansible tags

|Name |Description  |
|---|---|
|debug  |debug  |
|install  |Runs every tasks. Use this tag for a full installation |
|k8s  |Runs every k8s related tasks (such as joining members to cluster)  |
|main_server  |Runs every k8s tasks relatives to the installation of the first control plane node  |
|other servers  |Runs every k8s tasks relatives to the installation of the supplementary control plane nodes |
|agents |Runs every k8s tasks relatives to the installation of the worker nodes |
|helm |Runs every tasks to install Helm on the control plane nodes  |
|uninstall  |Runs the uninstall procedure (WIP) |

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
ansible-playbook -i /path/to/inventory --user <ansible_user> --private-key </path/to/ansible_user_private_key> -t other_servers <playbook_name>.yml
```

```yaml
# Add new worker nodes
ansible-playbook -i /path/to/inventory --user <ansible_user> --private-key </path/to/ansible_user_private_key> -t agents <playbook_name>.yml
```

#### In single node mode

```yaml
ansible-playbook -i /path/to/inventory --user <ansible_user> --private-key </path/to/ansible_user_private_key> -t install --skip-tags agents, other_servers <playbook_name>.yml
```

### Uninstallation

```yaml
ansible-playbook -i /path/to/inventory --user <ansible_user> --private-key </path/to/ansible_user_private_key> -t uninstall <playbook_name>.yml
```

### Testing

Create a symlink to the role in the tests/ directory:

```bash
ln -s ../.. tests/roles
```
