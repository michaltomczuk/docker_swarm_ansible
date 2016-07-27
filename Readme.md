## What Does This Button (playbook) Do? ##

This playbook assumes that we will create three node cluster with a Consul and Registrator as a autodiscover backend.

## Docker Swarm cluster ##

This playbook covers creating SWARM cluster using:
* Ubuntu 14.04 LTS
* Centos 7 (soon)
* vagrant (VirtualBox vm)
as a "hypervisor".

## Prepare environment ##

Before we start we need to install a few things:

1. [Vagrant](https://www.vagrantup.com/docs/installation/ "vagrant")
2. [VirtualBox](https://www.virtualbox.org/wiki/Downloads "virtualbox")
3. [VirtualBox  Extension Pack](http://download.virtualbox.org/virtualbox/5.1.0/Oracle_VM_VirtualBox_Extension_Pack-5.1.0-108711.vbox-extpack "virtualbox extension pack")
4. [Ansible](http://docs.ansible.com/ansible/intro_installation.html "ansible")
5. Clone this repo

```bash
git clone git@github.com:zerodowntime/docker_swarm_ansible.git
```

## How to create dev environment on local machine ##

First of all we need to create our hosts which will create the SWARM cluster. Our [Vagrantfile](https://github.com/zerodowntime/docker_swarm_ansible/blob/master/Vagrantfile "Vagrantfile") will create three nodes:

```bash
boxes = [
    {
        :name => "node1",
        :eth1 => "192.168.50.101",
        :mem => "2048",
        :cpu => "2",
        :os => "ubuntu/trusty64",
    },
    {
        :name => "node2",
        :eth1 => "192.168.50.102",
        :mem => "2048",
        :cpu => "2",
        :os => "ubuntu/trusty64",
    },
    {
        :name => "node3",
        :eth1 => "192.168.50.103",
        :mem => "2048",
        :cpu => "2",
        :os => "ubuntu/trusty64",
    }
]
```

and next will copy your SSH public key into each machine.

To deploy creating our three hosts we need to execute 

```bash
vagrant up
```

command. 

Next we can see the result

```bash
vagrant status
```

which should look simmilar to this

```bash
Current machine states:

node1                     running (virtualbox)
node2                     running (virtualbox)
node3                     running (virtualbox)
```

We can also try to SSH to those vms using ssh command

```bash

ssh root@192.168.50.101
```

If all went well we can now deploy our playbook by executing

```bash
ansible-playbook -i ./dev/hosts create_swarm_cluster.yml
```

## How to manage cluster? ##

First of all lets check cluster status

```bash
docker -H 192.168.50.101:4000 info
```
and we should see that our cluster has three nodes and available resources are 6 CPUs and 6GB of RAM total.

Next we can see what kind of containers we have running on the cluster

```bash
docker -H 192.168.50.101:4000 ps -a
```

also we can check Consul (cluster) UI which should be available under URL http://192.168.50.101:8500.








