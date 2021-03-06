# kubernetes-exoscale

*This is now obsolete, you should check this ansible [recipe](https://github.com/runseb/ansible-kubernetes) instead.*

A tutorial to run Kubernetes on [CloudStack](http://cloudstack.apache.org) using [exoscale](http://www.exoscale.ch) as a Cloud provider. It is based on Kelsey Hightower [tutorial](https://github.com/kelseyhightower/kubernetes-fleet-tutorial). This tutorial walks you through creating an [etcd](https://github.com/coreos/etcd) cluster and a [kubernetes](https://github.com/GoogleCloudPlatform/kubernetes) cluster. Both clusters run within their own security group.

# Introduction

Kubernetes is a [docker](http://docker.io) management platform open sourced by Google. It aims at making application management scalable, fault tolerant and easy. It provides replication and proxy services. Kubernetes can be deployed on any set of machines that supports docker. CoreOS is a new linux distribution that offers rolling upgrade and is customized for running docker applications and managing distributed systemd services. The availibility of a CoreOS template on exoscale makes it extremely simple to deploy Kubernetes there and shows how Kubernetes can run on CloudStack.

## Exoscale

[exoscale](http://www.exoscale.ch) is a swiss cloud provider based on [Apache CloudStack](http://cloudstack.apache.org), it is setup in a similar fashion than Amazon EC2 and provides virtual machine instances in various flavors including [CoreOS](http://coreos.com). Access to the instances is via ssh using key pairs and network isolation is done through security groups. To go through this tutorial you will need to [sign-up](https://portal.exoscale.ch/register) and go through some of the getting started [guides](https://community.exoscale.ch/compute/tutorials/getting-started/).

## Apache CloudStack

While this tutorial is specific to exoscale it can be applied to any other CloudStack clouds, as long as a CoreOS template is made available to the users. There are other ways to install Kubernetes on CloudStack, via [Vagrant](http://www.vagrantup.com) using the vagrant-cloudstack plugin, using the salt recipes available in Kubernetes or writing your own recipes for Ansible, Chef etc. 

## CoreOS

CoreOS is a new linux distribution aimed at scalable deployments and docker support. It offers rolling upgrade and a minimal install, with the particularity of having etcd and [fleet](https://coreos.com/using-coreos/clustering/) installed. Fleet allows you to manage systemd based services on a cluster of machines, in this tutorial we will install Kubernetes through systemd services and launch them with fleet. Etcd is a distributed key value store that can be used for service discovery.

# Getting started

This tutorial is composed of four steps:

* Creating the security groups on exoscale
* Creating an ETCD cluster
* Creating a KUBERNETES cluster
* Launching the KUBERNETES services

## Creating the security groups and a key pair on exoscale

We will launch one ETCD cluster (three nodes) and one KUBERNETES cluster (five nodes). Each cluster will be in its own security group: etcd and kubernetes.

Follow the [guide](https://community.exoscale.ch/compute/tutorials/firewall/introduction-to-security-groups/) and create the two security groups. They should have the following rules

Etcd security group:

![Etcd security group](images/etcd.png)

Kubernetes security group:

![Kubernetes security group](images/kubernetes.png)

[Create](https://community.exoscale.ch/compute/tutorials/getting-started/ssh-keypairs/) a keypair and save the private key locally.

## Bootstrapping the etcd cluster

To ease the bootstrapping of our etcd cluster we are going to use the CoreOS discovery service (really an existing etcd cluster :) ).
You can learn more about the [discovery protocol](https://github.com/coreos/etcd/blob/master/Documentation/discovery-protocol.md) including setting up your own discovery service.

Start by grabbing a new discovery token:

```
curl -w "\n" https://discovery.etcd.io/new
```

And edit the `nodes/etcd.yml` to replace the `<token>` with it.

Our deployment process will use two [cloud config](https://coreos.com/docs/cluster-management/setup/cloudinit-cloud-config/) scripts located in `nodes/` and very similar to the EC2 [scripts](https://coreos.com/docs/running-coreos/cloud-providers/ec2/). This is not suprising as exoscale aims to mimic EC2 and supports [userdata](https://community.exoscale.ch/compute/tutorials/automation/bootstrap-your-instances-with-cloudinit-and-user-data/).

The `deploy` directory contains two python scripts that use libcloud to start the instances.
Install [libcloud](http://libcloud.apache.org):

```
sudo pip install apache-libcloud
```

Store your API keys from exoscale as environment variables:

```
export EXOSCALE_API_KEY=<api key>
export EXOSCALE_SECRET_KEY=<secret key>
```

Then execute the deployment script:

```
python ./deploy/etcd.py
```

Once the script finished you should see three CoreOS instances in the exoscale dashboard. SSH to one of the instance using your ssh keypair and the core user.
Then execute `fleetctl list-machines` and you should see something similar to:

```
$ fleetctl list-machines
MACHINE		IP		METADATA
1745a36f...	185.19.28.164	role=etcd
34399153...	185.19.28.129	role=etcd
3cd468ed...	185.19.28.191	role=etcd
```

## Creating the Kubernetes cluster

Take one of the IP addresses of your etcd cluster nodes and edit the `nodes/knode.yml` file to replace `<ip>` with it.

Execute the deployment script to start 5 kubernetes nodes:

```
python ./deploy/kube.py
```

And on one of your etcd or kubernetes machines you should now see:

```
$ fleetctl list-machines
MACHINE		IP		METADATA
11deb0be...	185.19.28.224	role=kubernetes
1745a36f...	185.19.28.164	role=etcd
3424fd8d...	185.19.28.207	role=kubernetes
34399153...	185.19.28.129	role=etcd
3cd468ed...	185.19.28.191	role=etcd
6ce6b544...	185.19.28.211	role=kubernetes
d9803976...	185.19.28.201	role=kubernetes
db72da3e...	185.19.28.229	role=kubernetes
```

At this stage, there might be an issue with FLEETCTL_ENDPOINT not being set on the kubernetes nodes. This prevents you from running `fleetctl` on those nodes.
This is under investigation. To solve this temporarily, ssh into one of your `kubernetes` node and do:

```
export FLEETCTL_ENDPOINT=http://<ip_of_one_etcd_node>:4001
```

### Sanity check

[Flannel](https://github.com/coreos/flannel) formerly known as Rudder, is used to create a networking overlay between all Kubernetes pods.
In the `nodes/knode.yml` file you will notice:

```
ExecStartPre=-/usr/bin/etcdctl --peers 185.19.28.207:4001 mk /coreos.com/network/config '{"Network":"10.0.0.0/16"}
``` 

This lines store a specific key `/coreos.com/networking/config` in your etcd cluster and puts in the value `{"Network":"10.0.0.0/16"}`. When Kubernetes nodes start they will lease a /24 subnet from this network and create a UDP based overlay between all nodes. The network interfaces will look something like this:

```
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 10.0.2.1  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 56:84:7a:fe:97:99  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 185.19.28.229  netmask 255.255.252.0  broadcast 185.19.31.255
        inet6 fe80::4fa:dcff:fe00:122  prefixlen 64  scopeid 0x20<link>
        ether 06:fa:dc:00:01:22  txqueuelen 1000  (Ethernet)
        RX packets 24737  bytes 30864276 (29.4 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 15098  bytes 1497685 (1.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

flannel0: flags=81<UP,POINTOPOINT,RUNNING>  mtu 1472
        inet 10.0.2.0  netmask 255.255.0.0  destination 10.0.2.0
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 500  (UNSPEC)
        RX packets 8  bytes 448 (448.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 8  bytes 2696 (2.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 0  (Local Loopback)
        RX packets 80  bytes 8208 (8.0 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 80  bytes 8208 (8.0 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Check that docker is running properly:

```
$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
```

## Launching the Kubernetes services

All the systemd services needed to run Kubernetes are located in the `./units` directory. To get the service units on one of the Kubernetes nodes, ssh into one of them and clone this very repo:

```
$ git clone https://github.com/runseb/kubernetes-exoscale.git
```

A convenience configuration script is provided `./kubernetes-exoscale/deploy/config.sh` to set the proper IP addresses in the services script and launch them with fleet:

```
$ chmod +x ./kubernetes-exoscale/deploy/config.sh
$ cd kubernetes-exoscale/units
$ find .
.
./kube-apiserver.service
./kube-controller-manager.service
./kube-kubelet.service
./kube-proxy.service
./kube-register.service
./kube-scheduler.service
$ ../deploy/config.sh
```

Fleet will schedule these services on the kubernetes cluster, you can check their status with:

```
$ fleetctl list-units
UNIT				MACHINE				ACTIVE	SUB
kube-apiserver.service		59d0740a.../185.19.29.94	active	running
kube-controller-manager.service	59d0740a.../185.19.29.94	active	running
kube-kubelet.service		072512c3.../185.19.29.64	active	running
kube-kubelet.service		0f645055.../185.19.29.95	active	running
kube-kubelet.service		59d0740a.../185.19.29.94	active	running
kube-kubelet.service		ab856ccb.../185.19.29.51	active	running
kube-kubelet.service		f13220c3.../185.19.29.9		active	running
kube-proxy.service		072512c3.../185.19.29.64	active	running
kube-proxy.service		0f645055.../185.19.29.95	active	running
kube-proxy.service		59d0740a.../185.19.29.94	active	running
kube-proxy.service		ab856ccb.../185.19.29.51	active	running
kube-proxy.service		f13220c3.../185.19.29.9		active	running
kube-register.service		59d0740a.../185.19.29.94	active	running
kube-scheduler.service		59d0740a.../185.19.29.94	active	running
```

Grab the IP address of the kubernetes api server and set it as the KUBERNETES_MASTER environment variable:

```
export KUBERNETES_MASTER=http://185.19.28.116:8080
```

The convenience script downloaded `kubecfg` from Google storage, set it to executable. Use it to list your minions:

```
$ cd ../
$ ./kubecfg list /minions
Minion identifier
----------
185.19.28.211
185.19.28.224
185.19.28.229
185.19.28.230
185.19.28.244
```

You now have a functioning Kubernetes cluster running on a [CloudStack](http://cloudstack.apache.org) cloud, thanks to [exoscale](http://exoscale.ch).
On the Kubernetes website you can go through some of the [examples](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/examples/guestbook/README.md)

# Using Kubernetes

Start a replicated Nginx pod:

```
$  ./kubecfg -p 8088:80 run dockerfile/nginx 3 myNginx
```

This will start three containers running Nginx, with port 80 forwarded to port 8088 of the host. Be careful not to forward to 8080 as the KUBERNETES_MASTER is running on one of the host on port 8080 and this may cause conflicts (probably an aera of improvement in this setup).

```
./kubecfg list /pods
ID                                     Image(s)            Host                Labels                          Status
----------                             ----------          ----------          ----------                      ----------
43e78486-44a5-11e4-9079-060b8200019b   dockerfile/nginx    185.19.29.9/        replicationController=myNginx   Running
43e7f6af-44a5-11e4-9079-060b8200019b   dockerfile/nginx    185.19.29.95/       replicationController=myNginx   Running
43e805f4-44a5-11e4-9079-060b8200019b   dockerfile/nginx    185.19.29.94/       replicationController=myNginx   Running
```

Take your browser and open `http://185.19.29.9:8088` or the IP that you got and enjoy Nginx:
