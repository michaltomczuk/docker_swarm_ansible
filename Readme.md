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

## Features Walk-through ##

1. Make sure that there is an `index.html` file on every node with some content identifying the node:

    ```bash
    vagrant ssh node2 -- cat /opt/web/index.html
    <p>This website was deployed on: <b><h2>192.168.50.102</h2></b></p>
    ```

1. Spawn a container that will serve this file on some node:
   ```bash
      docker -H 192.168.50.102:4000 run --name nginx-single1 -v /opt/web:/usr/share/nginx/html:ro -d -p 8090:80 nginx
   ```
   We're using Swarm API here (192.168.50.101:4000) that will choose a node for the container according to [the strategy](https://docs.docker.com/swarm/scheduler/strategy/).

1. Check that the container is running
   * use Swarm API
     ```bash
     docker -H 192.168.50.101:4000 ps -a
     ```

    * use Consul
     ```bash
     http://192.168.50.101:8500
     ```

    It should appear as `nginx-80` - that is the service name.

    > It might happened that the container was started on a different node that the API we hit.
    > It is due to the load balancing and the aforementioned strategies.

1. Spawn another container and look where it was started:

   ```bash
   docker -H 192.168.50.102:4000 run --name nginx-single2 -v /opt/web:/usr/share/nginx/html:ro -d -p 8090:80 nginx
   ```
3. Hit HTTP interface of the containers:

    If your containers where started on node1 and node3:

    http://192.168.50.101:8090
    http://192.168.50.103:8090

4. Get information about the service:
   via consul REST API exposed by every node in the cluster:
     ```bash
     curl 192.168.50.102:8500/v1/catalog/service/nginx-80
     ```

    The result is something like the following:
     ```json
     [
        {
          "ServicePort": 8090,
          "ServiceAddress": "192.168.50.101",
          "ServiceTags": null,
          "ServiceName": "nginx-80",
          "ServiceID": "379f63b6cb5e:nginx-single1:80",
          "Address": "192.168.50.101",
          "Node": "node1"
        },
        {
          "ServicePort": 8090,
          "ServiceAddress": "192.168.50.103",
          "ServiceTags": null,
          "ServiceName": "nginx-80",
          "ServiceID": "6a38ca419203:nginx-single2:80",
          "Address": "192.168.50.103",
          "Node": "node3"
        }
     ]
     ```
   * via Consul DNS service from within a special container with network tools:
     ```bash
     docker -H tcp://192.168.50.101:4000 run --rm -it --name dnt joffotron/docker-net-tools
     ```

    That will drop you into the shell of the `dnt` container. It gets attached to the docker network
     and can access the DNS service provided by Consul:

    ```bash
     dig nginx-80.service.consul

     ; <<>> DiG 9.10.2-P4 <<>> nginx-80.service.consul
     ;; global options: +cmd
     ;; Got answer:
     ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 48529
     ;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 0

     ;; QUESTION SECTION:
     ;nginx-80.service.consul.	IN	A

     ;; ANSWER SECTION:
     nginx-80.service.consul. 0	IN	A	192.168.50.101
     nginx-80.service.consul. 0	IN	A	192.168.50.103

     ;; Query time: 3 msec
     ;; SERVER: 172.17.0.1#53(172.17.0.1)
     ;; WHEN: Fri Jul 29 16:12:45 UTC 2016
     ;; MSG SIZE  rcvd: 119
     ```

     Note, that the service is available on 2 hosts. What's more Consul will provide automatic load
     balancing between the instances of the service:

    ```bash
     $ ping -c 2 nginx-80.service.consul
     PING nginx-80.service.consul (192.168.50.101): 56 data bytes
     64 bytes from 192.168.50.101: seq=0 ttl=63 time=0.531 ms

     $ ping -c 2 nginx-80.service.consul
     PING nginx-80.service.consul (192.168.50.103): 56 data bytes
     64 bytes from 192.168.50.103: seq=0 ttl=63 time=0.322 ms
     ```

1. Add another nginx container and check that it is automatically registered under nginx-80:
   ```bash
      docker -H 192.168.50.102:4000 run --name nginx-single3 -v /opt/web:/usr/share/nginx/html:ro -d -p 8090:80 nginx
   ```

    > Note that the registry recognizes the same service based on the image name and the port (nginx-80).
