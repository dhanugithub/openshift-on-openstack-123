# General
The purpose of this project is to provide a simple, yet flexible deployment of OpenShift on OpenStack using a three step process. This guide assumes you are familiar with OpenStack.

# Contribution
If you want to provide additional features, please feel free to contribute via pull requests or any other means.
We are happy to track and discuss ideas, topics and requests via 'Issues'.

In addition I would like to metion I borrowed a lot of ideas from two other projects.
* [OpenShift setup for Hetzner from RH SSA team](https://github.com/RedHat-EMEA-SSA-Team/hetzner-ocp)
* [OpenShift on OpenStack](https://github.com/redhat-openstack/openshift-on-openstack)

# Pre-requisites
* Working OpenStack deployment. Tested is OpenStack Pike (12) using RDO.
* RHEL 7 image. Tested is RHEL 7.4.
* An openstack ssh key for accessing instances.
* A pre-configured provider (public) network with at least three available floating ips.
* Flavors configured for OpenShift.
  * ocp.master  (2 vCPU, 4GB RAM, 30 GB Root Disk)
  * ocp.infra   (4 vCPU, 16GB RAM, 30 GB Root Disk)
  * ocp.node    (2 vCPU, 4GB RAM, 30 GB Root Disk)
  * ocp.bastion (1 vCPU, 4GB RAM, 30 GB Root Disk)
* A router that has the provider network configured as a gateway.
* Properly configured cinder and nova storage.
  * Make sure you aren't using default loop back and have disabled disk zeroing in cinder/nova for LVM.

More information on setting up proper OpenStack environment can be found [here](https://keithtenzer.com/2018/02/05/openstack-12-pike-lab-installation-and-configuration-guide-with-hetzner-root-servers/).

# Tested Deployments
```Single Master - Non HA```

Single Master deployment is 1 Master, 1 Infra node and X number of App nodes. This configuration is a non-HA setup, ideal for test environments.
![](images/openshift_on_openstack_non_ha.PNG)

```Multiple Master - HA```

Multiple Master deployment is 3 Master, 2 Infra node and X number of App nodes. This configuration is an HA setup. By default etcd and registry are not using persistent storage. This would need to be configured post-install manually at this time if those should be persisted.
![](images/openshift_on_openstack_ha.PNG)

# Install
![](images/one.png)

```[OpenStack Controller]```

Clone Git Repository
```
# git clone https://github.com/dhanugithub/openshift-on-openstack-123
```

Change dir to repository
```
# cd openshift-on-openstack-123
```

Checkout release branch 1.0
```
# git checkout release-1.0
```

Checkout master
```
# git checkout master
```

Configure Parameters
```
# cp sample_vars.yml vars.yml
```
```
# vi vars.yml
---
### OpenStack Setting ###
domain_name: berlin.x-ion.de
dns_forwarders: [8.8.8.8,8.8.4.4]
public_network: public
service_subnet_cidr: 192.168.1.0/24
router_id: <router id from 'openstack router list'>
image: rhel74
ssh_user: cloud-user
ssh_key_name: admin
stack_name: openshift
openstack_version: 3.15.0
contact: admin@berlin.x-ion.de
heat_template_path: /home/dhanashree/openshift-on-openstack-123/heat/openshift.yaml

### OpenShift Settings ###
openshift_version: 3.7
docker_version: 1.12.6
openshift_ha: true
registry_replicas: 2
openshift_user: admin
openshift_passwd: <password>

### Red Hat Subscription ###
rhn_username: <user>
rhn_password: <password>
rhn_pool: <pool>

### OpenStack Instance Count ###
master_count: 3
infra_count: 2
node_count: 2

### OpenStack Instance Group Policies ###
### Set to 'affinity' if only one compute node ###
master_server_group_policies: "['anti-affinity']"
infra_server_group_policies: "['anti-affinity']"
node_server_group_policies: "['anti-affinity']"

### OpenStack Instance Flavors ###
bastion_flavor: ocp.bastion
master_flavor: ocp.master
infra_flavor: ocp.infra
node_flavor: ocp.node
```

Authenticate OpenStack Credentials
```
# source /root/keystonerc_admin
```

Disable host key checking
```
# export ANSIBLE_HOST_KEY_CHECKING=False
```

Deploy OpenStack Infrastructure for OpenShift
```
# ansible-playbook deploy-openstack-infra.yml --private-key=/root/admin.pem -e @vars.yml
```

![](images/two.png)

Get ip address of the bastion host.
```
# openstack stack output show -f value -c output_value openshift ip_address

{
  "masters": [
    {
      "name": "master0",
      "address": "192.168.1.19"
    },
    {
      "name": "master1",
      "address": "192.168.1.16"
    },
    {
      "name": "master2",
      "address": "192.168.1.15"
    }
  ],
  "lb_master": {
    "name": "lb_master",
    "address": "144.76.134.230"
  },
  "infras": [
    {
      "name": "infra0",
      "address": "192.168.1.10"
    },
    {
      "name": "infra1",
      "address": "192.168.1.11"
    }
  ],
  "lb_infra": {
    "name": "lb_infra",
    "address": "144.76.134.229"
  },
  "bastion": {
    "name": "bastion",
    "address": "144.76.134.228"
  },
  "nodes": [
    {
      "name": "node0",
      "address": "192.168.1.6"
    },
    {
      "name": "node1",
      "address": "192.168.1.13"
    }
  ]
}
```

SSH to the bastion host using cloud-user and key.
```
ssh -i /root/admin.pem cloud-user@144.76.134.228
```

```[Bastion Host]```

Change dir to repository
```
# cd openshift-on-openstack-123
```

Authenticate OpenStack Credentials
```
[cloud-user@bastion ~]$ source /home/cloud-user/keystonerc_admin
```

Disable host key checking
```
[cloud-user@bastion ~]$ export ANSIBLE_HOST_KEY_CHECKING=False
```

Prepare the nodes for deployment of OpenShift.
```
[cloud-user@bastion ~]$ ansible-playbook prepare-openshift.yml --private-key=/home/cloud-user/admin.pem -e @vars.yml

PLAY RECAP *****************************************************************************************
bastion                    : ok=15   changed=7    unreachable=0    failed=0
infra0                     : ok=18   changed=13   unreachable=0    failed=0
infra1                     : ok=18   changed=13   unreachable=0    failed=0
localhost                  : ok=7    changed=6    unreachable=0    failed=0
master0                    : ok=18   changed=13   unreachable=0    failed=0
master1                    : ok=18   changed=13   unreachable=0    failed=0
master2                    : ok=18   changed=13   unreachable=0    failed=0
node0                      : ok=18   changed=13   unreachable=0    failed=0
node1                      : ok=18   changed=13   unreachable=0    failed=0
```

![](images/three.png)

```[Bastion Host]```

Deploy OpenShift.
```
[cloud-user@bastion ~]$ ansible-playbook -i /home/cloud-user/openshift-inventory --private-key=/home/cloud-user/admin.pem -vv /usr/share/ansible/openshift-ansible/playbooks/byo/config.yml
PLAY RECAP *****************************************************************************************
infra0.ocp3.lab            : ok=183  changed=59   unreachable=0    failed=0
infra1.ocp3.lab            : ok=183  changed=59   unreachable=0    failed=0
localhost                  : ok=12   changed=0    unreachable=0    failed=0
master0.ocp3.lab           : ok=635  changed=265  unreachable=0    failed=0
master1.ocp3.lab           : ok=635  changed=265  unreachable=0    failed=0
master2.ocp3.lab           : ok=635  changed=265  unreachable=0    failed=0
node0.ocp3.lab             : ok=183  changed=59   unreachable=0    failed=0
node1.ocp3.lab             : ok=183  changed=59   unreachable=0    failed=0


INSTALLER STATUS ***********************************************************************************
Initialization             : Complete
Health Check               : Complete
etcd Install               : Complete
Master Install             : Complete
Master Additional Install  : Complete
Node Install               : Complete
Hosted Install             : Complete
Service Catalog Install    : Complete
```

Run post install playbook
```
[cloud-user@bastion ~]$ ansible-playbook post-openshift.yml --private-key=/home/cloud-user/admin.pem -e @vars.yml

PLAY RECAP **************************************************************************************************************************
infra0                     : ok=4    changed=2    unreachable=0    failed=0
infra1                     : ok=4    changed=2    unreachable=0    failed=0
localhost                  : ok=7    changed=6    unreachable=0    failed=0
master0                    : ok=6    changed=4    unreachable=0    failed=0
master1                    : ok=6    changed=4    unreachable=0    failed=0
master2                    : ok=6    changed=4    unreachable=0    failed=0
node0                      : ok=4    changed=2    unreachable=0    failed=0
node1                      : ok=4    changed=2    unreachable=0    failed=0
```

Login in to UI.
```
https://openshift.144.76.134.226.xip.io:8443
```

# Optional
Configure admin user
```
[cloud-user@bastion ~]$ ssh -i /home/cloud-user/admin.pem cloud-user@master0
```

Authenticate as system:admin user.
```
[cloud-user@master0 ~]$ oc login -u system:admin -n default
```

Make user OpenShift Cluster Administrator
```
[cloud-user@master0 ~]$ oadm policy add-cluster-role-to-user cluster-admin admin
```

Install Metrics
Set metrics to true in inventory
```
[cloud-user@bastion ~]$ vi openshift_inventory
...
openshift_hosted_metrics_deploy=true
...
```

Run playbook for metrics
```
[cloud-user@bastion ~]$ ansible-playbook -i /home/cloud-user/openshift-inventory --private-key=/home/cloud-user/admin.pem -vv /usr/share/ansible/openshift-ansible/playbooks/byo/openshift-cluster/openshift-metrics.yml
PLAY RECAP **************************************************************************************************************************
infra0.ocp3.lab            : ok=45   changed=4    unreachable=0    failed=0
infra1.ocp3.lab            : ok=45   changed=4    unreachable=0    failed=0
localhost                  : ok=11   changed=0    unreachable=0    failed=0
master0.ocp3.lab           : ok=48   changed=4    unreachable=0    failed=0
master1.ocp3.lab           : ok=48   changed=4    unreachable=0    failed=0
master2.ocp3.lab           : ok=205  changed=48   unreachable=0    failed=0
node0.ocp3.lab             : ok=45   changed=4    unreachable=0    failed=0
node1.ocp3.lab             : ok=45   changed=4    unreachable=0    failed=0


INSTALLER STATUS ********************************************************************************************************************
Initialization             : Complete
Metrics Install            : Complete
```

Install Logging
Set logging to true in inventory
```
[cloud-user@bastion ~]$ vi openshift_inventory
...
openshift_hosted_logging_deploy=true
...
```

Run playbook for logging
```
[cloud-user@bastion ~]$ ansible-playbook -i /home/cloud-user/openshift-inventory --private-key=/home/cloud-user/admin.pem -vv /usr/share/ansible/openshift-ansible/playbooks/byo/openshift-cluster/openshift-logging.yml
```

# Issues
## Issue 1: Dynamic storage provisioning using cinder not working
Currently using the OpenStack cloud provider requires using Cinder v2 API. Most current OpenStack deployments will default to v3.
```
Error creating cinder volume: BS API version autodetection failed.
```
If you provision OpenShift volume and it is pending check /var/log/messages on master. If you see this error you need to add following in /etc/origin/cloudprovider/openstack.conf on masters and all nodes then restart node service on node and controller service on master.
```
...
[BlockStorage]
bs-version=v2
...
```

The post-openshift.yml playbook takes care of setting v2 for cinder automatically.

## Issue 2: Service Catalog Install Fails

This seems to be general issue with OpenShift 3.7 installer, somtimes API timeout's occur, it can be ignored or you can re-run playbook to install just service catalog.

## Issue 3: Hosted Install Fails

The registry sometimes fails to complete install due to host resolution of xip.io. Not sure if this is issue in OpenShift 3.7 or environment. Simply re-running hosted playbook resolved the issue and resulted in successful installation.
