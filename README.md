# Kubernetes cluster
A vagrant script for setting up a Kubernetes cluster using Kubeadm

## Pre-requisites

 * **[Vagrant 2.1.4+](https://www.vagrantup.com)**
 * **[Virtualbox 5.2.18+](https://www.virtualbox.org)**

## How to Run

Execute the following vagrant command to start a new Kubernetes cluster, this will start one master and two nodes:

```
vagrant up
```

You can also start invidual machines by vagrant up k8s-master, vagrant up k8s-node1 and vagrant up k8s-node2

If more than two nodes are required, you can edit the servers array in the Vagrantfile

```
servers = [
{
        :name => "k8s-node3",
        :type => "node",
        :box => "ubuntu/bionic64",
        :eth1 => "192.168.50.13",
        :mem => "4096",
        :cpu => "2"
    }
]
 ```

As you can see above, you can also configure IP address, memory and CPU in the servers array. 

## Clean-up

Execute the following command to remove the virtual machines created for the Kubernetes cluster.
```
vagrant destroy -f
```

You can destroy individual machines by vagrant destroy k8s-node1 -f

## Licensing

[Apache License, Version 2.0](http://opensource.org/licenses/Apache-2.0).
