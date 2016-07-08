---
layout: post
title:  "Provision Azure infrastructure using Python Fabric and Azure-cli"
date:   2016-07-07 19:15:00
categories: azure fabric
author: ripusingla
---


### Some Background

Uninstall.io was part of Summer 2015 batch of Microsoft Accelerator program. After completion of the program in Jun 2015, we received bunch of
Azure credits to try out Azure Cloud ecosystem. Earlier all our machines were hosted on Amazon EC2. 
With free credits at our disposal, we started migrating our less critical machines to Azure to save on costs. As of today we are fully hosted
on Azure Cloud.


To start with, Azure portal was good enough for a single machine deployment and network configuration etc. But soon it became too time consuming to 
manage more machines. Azure templates were too cumbersome and repetitive for our taste. So we decided to build an inhouse `python-fabric & azure-cli` combo - a
suit of scripts to automate the most common tasks.


### Code

Basic structure is -

  1. fab_common.py
  2. fabfile.py
  3. fab_xyz_deploy.py - where xyz is any deployment need e.g. redis cluster, couch cluster, Mysql, Mongo, Load balanced API servers etc.

`fab_common.py` implements all the building blocks using `azure-cli` module. For example, creating vm, disk, vnet, network security group, 
resource group, availability set, attaching/mounting disk, ssh reset, firewal modification and many more. Also it contains code for installing 
base packages which are needed on every server e.g. monitoring packages like collectd, filebeat etc. This file has grown to 1000+ line of 
python code containing basic building blocks over time. Few examples are -

{% highlight python %}
def azure_vm_create(resource_group, vm_name, location, vnet, vm_size, storage, diag_storage, avail_set):
    local("""
        azure vm create \
        --resource-group {0} \
        --name {1} \
        --nic-id /subscriptions/{5}/resourceGroups/{0}/providers/Microsoft.Network/networkInterfaces/{1} \
        --subnet-id /subscriptions/{5}/resourceGroups/{0}/providers/Microsoft.Network/virtualNetworks/{3}/subnets/default \
        --nic-name {1} \
        --vnet-name {3} \
        --vnet-subnet-name default \
        --location {2} \
        --os-type Linux \
        --image-urn {6} \
        --admin-username {7} \
        --ssh-publickey-file {8} \
        --vm-size {4} \
        --public-ip-id /subscriptions/{5}/resourceGroups/{0}/providers/Microsoft.Network/publicIPAddresses/{1} \
        --public-ip-name {1} \
        --public-ip-domain-name {1} \
        --public-ip-allocation-method Dynamic \
        --storage-account-name {9} \
        --storage-account-container-name vhds \
        --os-disk-vhd https://{9}.blob.core.windows.net/vhds/{1}.vhd \
        --subscription {5} \
        --availset-name {11} \
        --boot-diagnostics-storage-uri https://{10}.blob.core.windows.net/
        """.format(resource_group, vm_name, location, vnet, vm_size, subscription, IMAGE, env.user, azure_ssh_pem_file, storage, diag_storage, avail_set)
    )
{% endhighlight %}

{% highlight python %}
def azure_create_disk(resource_group, vm_name, size_in_gb, vhd_name, host_caching, storage_account_name, 
                        storage_account_container_name, lun):
    '''
    Azure - execute create new disk command
    '''
    local("""
        azure vm disk attach-new \
        --resource-group {0} \
        --vm-name {1} \
        --size-in-gb {2} \
        --vhd-name {3} \
        --host-caching {4} \
        --storage-account-name {5} \
        --storage-account-container-name {6} \
        --lun {7} \
        --subscription {8}
        """.format(resource_group, vm_name, size_in_gb, vhd_name, host_caching, storage_account_name, storage_account_container_name, lun, subscription)
    )
{% endhighlight %}


`fabfile.py` is just a dummy file which imports everything from all fab_* file and is used to execute commands. e.g 

{% highlight python %}
fab ask redis_cluster_create_vms
fab ask redis_cluster_attach_data_disk
fab ask redis_install
fab ask redis_configure
{% endhighlight %}

would deploy a fully configured 8 node redis cluster for us in matter of minutes.

`fab_xyz_deploy.py` are the files where custom logic for a specific deployment goes, all common logic is imported from fab_common.py to adhere to
DRY approach. For example, the installation parameters will be defined here

{% highlight python %}
resource_group = "redis-cluster"
location = "westus"
avail_set = "abc"
vnet = "abc"
nsg = "abc"
prem_storage = "abc"  # premium redis storage - ssd
diag_storage = "xyz" # standard redis storage
host_caching = "ReadOnly"
storage_account_container_name = "vhds"
mount_point = "/redisdisk"
{% endhighlight %}

Below would initialize the vms


{% highlight python %} 
@task
def redis_cluster_create_azure_components():
	'''
	create azure components like resource group, vnet, nsg, storage accounts etc.
	'''
    azure_initialize_components(resource_group, location, storage_name, vnet, nsg, avail_set, subnet_name)
{% endhighlight %}

{% highlight python %} 
@task
def redis_cluster_create_vms():
	'''
	create vms for deploying redis cluster
	'''
    vm_size = "Standard_DS13" #56 GB Ram, 8 cores

    # create 4 master servers and 4 slave servers for redis cluster
    for i in range(1, 5):
        for j in ("master", "slave"):
        vm_name = "redis-cluster-{0}{1}".format(j, i)

        # create network interface
        azure_network_nic_create(resource_group, vm_name, location, vnet, nsg)

        # create vm
        azure_vm_create(resource_group, vm_name, location, vnet, vm_size, prem_storage, diag_storage, avail_set)

        # show vm
        azure_vm_show(resource_group, vm_name)
{% endhighlight %}


### Conclusion

Once the basic structure is in place, we can build as much as we want on top of it and automate whole lot of things with ease and 
minimal code.