# Ovirt-deploy-ocp-cluster
The goal of this playbook is to deloy an Openshift cluster on Openshift Virtualization using agent-based installation.

## Prereqs

### Infrastructure
- An Openshift cluster with Openshift Virtualization
- A load balancer from the cluster to deploy
- a DNS resolving the cluster to deploy (nodes, api etc... openshift standard)
  - **be sure that the workstation running this playbook resolves the cluster elements - you may need to create a resolver**

### On  the workstation running the playbook
- this workstation should resolves (DNS) the Openshist Virtualization cluster and the Openshift cluster to deploy
- a pullsecret (by defaut in the user home directory)

- Ansible collection for the playbook
  - kubernetes.core                          
  - kubevirt.core 
  - community.crypto 
  - community.general
  - ansible.posix

```bash
ansible-galaxy install -r requirements.yaml
```

## Deploy a new cluster

### Prepare a host vars file
Using the template *host_vars/host.sample*, create a new host vars file *mycluster* for your cluster to deploy.

### Add the cluster to the inventory
Add the cluster to deploy the inventory file *inventory/hosts* in the group *cluster_to_deploy*

### Run the playbook
```bash
ansible-playbook deploy.yaml
```
...and wait 30 to 40min.

## Add nodes to a cluster
This playbook will add nodes to an existing cluster using [https://github.com/openshift/installer/blob/master/docs/user/agent/add-node/node-joiner-monitor.sh](https://github.com/openshift/installer/blob/master/docs/user/agent/add-node/node-joiner-monitor.sh).

**Only worker and infra nodes can be added**

### Add nodes to the cluster host vars file
In the host vars file (located in *host_vars* directory), add the new nodes to `node.worker` or `node.infra`, just like any other nodes.
```
  worker:
    ...    
    - { ip: 10.3.59.109, cpu: 24, ram_gb: 48, disk_size_gb: 300 }
    - { ip: 10.3.59.110, cpu: 24, ram_gb: 48, disk_size_gb: 300 }
  infra:
    ...
    - { ip: 10.3.59.106, cpu: 16, ram_gb: 34, disk_size_gb: 100 }
```

### Add cluster authentication token
If cluster authentication kubeconfig  created during initial deployment (located in *clusters_data/gauthocp/install-dir/auth/kubeconfig*) doesn't exist, you need to provide an authentication token
