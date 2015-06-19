# Getting started with Calico on Docker

Calico provides IP connectivity between Docker containers on different hosts (as well as on the same host).

This example shows Calico with Docker's new, [experimental network driver support](https://github.com/docker/libnetwork).  This is an early prototype and not considered stable.  For a more stable version of Calico's integration with Docker, see [the main project page](https://github.com/Metaswitch/calico-docker) for examples based on Powerstrip.

### A note about names & addresses
In this example, we will use the following server names and IP addresses.

| hostname   | IP address   |
|------------|--------------|
| ubuntu-0  | 172.17.8.100 |
| ubuntu-1  | 172.17.8.101 |

## Set up your cluster

1) Install dependencies

* [VirtualBox][virtualbox] 4.3.10 or greater.
* [Vagrant][vagrant] 1.6 or greater.
* [Git][git]

2) Clone this project and get it running!

    git clone https://github.com/Metaswitch/calico-ubuntu-vagrant.git
    cd calico-ubuntu-vagrant

3) Startup and SSH

Run this command to bring up the VMs:

    vagrant up
    
Note for VMWare users: if you want to use VMWare instead of the default VirtualBox provider, you should pass `--provider vmware_workstation` or `--provider vmware_fusion` at this stage.

Note for Windows users: the instructions below assume that you're running on OS X or Linux.  If you're running on Windows, follow [these instructions](https://github.com/nickryand/vagrant-multi-putty) and then substitute `vagrant putty` for `vagrant ssh`.

4) Verify environment

You should now have two Ubuntu servers, each running etcd in a cluster. The servers are named ubuntu-0 and ubuntu-1.

At this point, it's worth checking that your servers can ping each other.

Connect to ubuntu-0 and then check it can ping ubuntu-1:

    vagrant ssh ubuntu-0
    ping 172.17.8.101

and vice-versa:

    vagrant ssh ubuntu-1
    ping 172.17.8.100

If you see ping failures, the likely culprit is a problem with the VirtualBox/VMWare network between the VMs.  You should check that each host is connected to the same virtual network adapter in VirtualBox/VMWare.  Rebooting the host may also help.  Remember to gracefully shut down the VMs with `vagrant halt` before you reboot.

## Starting Calico services<a id="calico-services"></a>

Once you have your cluster up and running, start calico on all the nodes

On ubuntu-0

    sudo ./calicoctl node --ip=172.17.8.100 --node-image=calico/node:libnetwork

On ubuntu-1

    sudo ./calicoctl node --ip=172.17.8.101 --node-image=calico/node:libnetwork

This will start the calico-node container. Check it is running:

    docker ps

You should see output like this on each node:

    vagrant@ubuntu-1:~$ docker ps -a
    CONTAINER ID        IMAGE                    COMMAND                CREATED             STATUS              PORTS                                            NAMES
    39de206f7499        calico/node:libnetwork   "/sbin/my_init"        2 minutes ago       Up 2 minutes                                                         calico-node
    5e36a7c6b7f0        quay.io/coreos/etcd      "/etcd --name calico   30 minutes ago      Up 30 minutes       0.0.0.0:4001->4001/tcp, 0.0.0.0:7001->7001/tcp   quay.io-coreos-etcd

## Creating networked endpoints

This pre-release version of Docker introduces a new flag to `docker run` to network containers:  `--publish-service <service>.<network>.<driver>`.

 * `<service>` is the name by which you want the container to be known on the network.
 * `<network>` is the name of the network to join.  Containers on different networks cannot communicate.
 * `<driver>` is the name of the network driver to use.  Calico's driver is called `calico`.

So let's go ahead and start a few of containers on each host.

On ubuntu-0

    docker run --publish-service srvA.net1.calico --name workload-A -tid busybox
    docker run --publish-service srvB.net2.calico --name workload-B -tid busybox
    docker run --publish-service srvC.net1.calico --name workload-C -tid busybox

On ubuntu-1

    docker run --publish-service srvD.net3.calico --name workload-D -tid busybox
    docker run --publish-service srvE.net1.calico --name workload-E -tid busybox

Now, check that A can ping C and E. You can get a containers IP by running

    docker inspect --format "{{ .NetworkSettings.IPAddress }}" <container name>

    docker exec workload-A ping -c 4 192.168.0.3
    docker exec workload-A ping -c 4 192.168.0.5

Also check that A cannot ping B or D:

    docker exec workload-A ping -c 4 192.168.0.2
    docker exec workload-A ping -c 4 192.168.0.4

By default, networks are configured so that their members can communicate with one another, but workloads in other networks cannot reach them.  B and D are in their own networks so shouldn't be able to ping anyone else.

You list the networks using

    docker network ls

Finally, to clean everything up (without doing a `vagrant destroy`), you can run

    sudo ./calicoctl reset


## IPv6
To connect your containers with IPv6, first make sure your Docker hosts each have an IPv6 address assigned.

On ubuntu-0

    sudo ip addr add fd80:24e2:f998:72d6::1/112 dev eth1

On ubuntu-1

    sudo ip addr add fd80:24e2:f998:72d6::2/112 dev eth1

Verify connectivity by pinging.

On ubuntu-01

    ping6 fd80:24e2:f998:72d6::2

Then restart your calico-node processes with the `--ip6` parameter to enable v6 routing.

On ubuntu-0

    sudo ./calicoctl node --ip=172.17.8.100 --ip6=fd80:24e2:f998:72d6::1 --node-image=calico/node:libnetwork

On ubuntu-1

    sudo ./calicoctl node --ip=172.17.8.101 --ip6=fd80:24e2:f998:72d6::2 --node-image=calico/node:libnetwork

Then, containers you start will be assigned IPv6 addresses in addition to IPv4.

NOTE: the `busybox` container in the examples above does not support IPv6.  To check IPv6 connectivity between nodes we recommend you use the `ubuntu` container.

[calico-ubuntu-vagrant]: https://github.com/Metaswitch/calico-ubuntu-vagrant-example
[virtualbox]: https://www.virtualbox.org/
[vagrant]: https://www.vagrantup.com/downloads.html
[using-calico]: https://github.com/Metaswitch/calico-docker/blob/master/docs/GettingStarted.md
[git]: http://git-scm.com/
