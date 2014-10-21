# Deploying Kubernetes on CoreOS with Fleet and Flannel

The goal of this tutorial is to build an elastic Kubernetes cluster on top of CoreOS using [Fleet](https://github.com/coreos/fleet) and [flannel](https://github.com/coreos/flannel).

The target audience for this tutorial understands how Kubernetes works at a basic level and has experience installing CoreOS and using [cloud-config](https://coreos.com/docs/cluster-management/setup/cloudinit-cloud-config).

## Overview

First we'll setup a dedicated etcd cluster, which will be used to bootstrap the other components. Then we'll build a CoreOS cluster where each machine will have flannel, Fleet, and Docker configured during boot via cloud-config. Next we'll deploy Kubernetes using Fleet and a collection of systemd unit files. Finally we'll wrap things up by deploying [kube-register](https://github.com/kelseyhightower/kube-register), which will auto register Kubernetes' Kubelets with the Kubernetes API server.

* [Setup a dedicated etcd cluster](#setup-a-dedicated-etcd-cluster)
* [Configure flannel](#configure-flannel)
* [Add machines to the cluster](#add-machines-to-the-cluster)
* [Deploy Kubernetes using Fleet](#deploying-kubernetes-with-fleet)
* [Auto Registering Kubernetes Kubelets with kube-register](#auto-registering-kubernetes-kubelets-with-kube-register)
* [Adding and removing machines](#adding-and-removing-machines)

### The Stack

* [CoreOS](https://coreos.com) (v440.0.0+)
* [etcd](https://github.com/coreos/etcd)
* [flannel](https://github.com/coreos/flannel)
* [Fleet](https://github.com/coreos/fleet) (v0.8.0+)
* [Kubernetes](https://github.com/GoogleCloudPlatform/kubernetes)
* [kube-register](https://github.com/kelseyhightower/kube-register)

### On the client machine

Install the following command line utilities so you can interact with the cluster remotely:

* [etcdctl](https://github.com/coreos/etcdctl)
* [fleetctl](https://github.com/coreos/fleet)
* [kubecfg](https://github.com/GoogleCloudPlatform/kubernetes)
```
$ go get github.com/coreos/etcdctl
$ go get github.com/coreos/fleet/fleetctl
$ gsutil mb gs://proppy-stuff-kubernetes
$ curl -L https://github.com/GoogleCloudPlatform/kubernetes/releases/download/v0.4.1/kubernetes.tar.gz | tar xvzf -
$ gsutil cp ./kubernetes/platforms/linux/amd64/* gs://proppy-stuff-kubernetes/v0.4.1/
$ gsutil acl ch -R -g AllUsers:R gs://proppy-stuff-kubernetes/v0.4.1

## Setup a Dedicated etcd Cluster

While etcd can run on every node in a CoreOS cluster it makes bootstrapping challenging. The other issue with running etcd on every machine has to do with resource contention, which can lead to unnecessary leader elections and poor performance.

Isolating etcd to a dedicated cluster, ideally with fixed IP addresses, means we can add and destroy any other machine without causing etcd downtime.

For simplicity, setup a single node etcd cluster using the following cloud-config file:

* [etcd.yml](configs/etcd.yml)

```
$ gcloud compute instances create etcd-node --image https://www.googleapis.com/compute/v1/projects/coreos-cloud/global/images/coreos-stable-444-5-0-v20141016 --zone us-central1-a --machine-type f1-micro --metadata-from-file user-data=configs/etcd.yml 

> Be sure to change the static IP address info in the etcd.yml cloud-config file to fit your environment.

I'm using a static IP address to make it easy to locate the etcd node later. Once you have the etcd IP, export the following environment variables used by the etcdctl and fleetctl clients:

```
$ gcloud compute ssh etcd-node --ssh-flag="-L 4001:localhost:4001" --ssh-flag="-fN" --zone us-central1-a
$ export ETCDCTL_PEERS=http://localhost:4001
$ export FLEETCTL_ENDPOINT=http://localhost:4001

```
Check that both the etcdctl and fleetctl clients are working:

```
$ etcdctl --no-sync ls /
```

```
$ fleetctl list-machines
MACHINE     IP              METADATA
9c1aa398... 192.168.12.10   role=etcd
```

## Configure flannel

flannel is used to setup and manage an overlay network, which will allow containers on different Kubernetes hosts to communicate. flannel reads it's runtime configuration from etcd, and requires a network block to be allocated for use by Kubernetes. Add the flannel configuration using etcdctl:

```
$ etcdctl --no-sync mk /coreos.com/network/config '{"Network":"10.0.0.0/16"}'
```

## Add Machines to the Cluster

At this point we are ready to start adding machines to the cluster. 

Pick your [favorite platform](https://coreos.com/docs/quickstart) and launch a couple of CoreOS machines using the `node.yml` cloud-config template.

### On GCE

$ gcloud compute instances create kube-nodeNN --image https://www.googleapis.com/compute/v1/projects/coreos-cloud/global/images/coreos-stable-444-5-0-v20141016 --zone us-central1-a --machine-type f1-micro --metadata-from-file user-data=configs/node.yml

> Be sure to adjust the IP address of your etcd server and SSH key in the node.yml cloud-config file.

Once the CoreOS machines have booted, you can use the fleetctl client to list the machine details:

```
$ fleetctl list-machines
MACHINE     IP              METADATA
9c1aa398... 192.168.12.10   role=etcd
a6681f2c... 192.168.12.229  role=kubernetes
fe36d443... 192.168.12.228  role=kubernetes
```

Notice all the "worker" machines have the `role=kubernetes` metadata field.

## Deploying Kubernetes with Fleet

Two of the Kubernetes components, the Kubelet and Proxy service, must run on every Kubernetes machine. Fleet can ensure this happens using [global units](https://github.com/coreos/fleet/blob/master/Documentation/unit-files-and-scheduling.md#unit-scheduling) and metadata filtering. The following unit files will run on every CoreOS machine where `role=kubernetes`:

```
$ fleetctl start units/kube-proxy.service
$ fleetctl start units/kube-kubelet.service
```

The other Kubernetes components make up the Kubernetes Master and only require a single instance to be running. The Kubernetes Master can run anywhere in the cluster, but we'll need to locate the IP address of the Kubernetes API server once it's up and running so we can configure the kubecfg Kubernetes client. 

```
$ fleetctl start units/kube-apiserver.service
$ fleetctl start units/kube-scheduler.service
$ fleetctl start units/kube-controller-manager.service
```

List the running units:

```
$ fleetctl list-units
UNIT                            MACHINE                     ACTIVE  SUB
kube-apiserver.service          a6681f2c.../192.168.12.229  active  running
kube-controller-manager.service a6681f2c.../192.168.12.229  active  running
kube-kubelet.service            a6681f2c.../192.168.12.229  active  running
kube-kubelet.service            fe36d443.../192.168.12.228  active  running
kube-proxy.service              a6681f2c.../192.168.12.229  active  running
kube-proxy.service              fe36d443.../192.168.12.228  active  running
kube-scheduler.service          a6681f2c.../192.168.12.229  active  running
```

Once you've located the kube-apiserver set the KUBERNETES_MASTER environment variable, which configures the kubecfg client use this API server:

```
$ gcloud compute ssh kube-nodeNN --ssh-flag="-L 8080:localhost:8080" --ssh-flag="-fN" --zone us-central1-a
$ export KUBERNETES_MASTER="http://localhost:8080"
```

## Auto Registering Kubernetes Kubelets with kube-register

Thanks to Fleet global units, new machines will get the Kubernetes Kubelet installed and configured during the boot process. However we'll still need some way of notifiying the Kubernetes API server about new machines. That's where kube-register comes in. kube-register syncs machine data from Fleet to the API server using the following workflow:

* Gather all machines from Fleet via the Fleet HTTP API
* Filter machines based on metadata
* Perform a health check on each Kubelet machine, then register healthy Kubelets with the API server. 

Start the registration service using fleet:

```
$ fleetctl start units/kube-register.service
```

The kube-register service needs to know where the Kubernetes API server is in order to register machines. This is accomplished using a Fleet requirement in the kube-register.service unit file:

```
[X-Fleet]
MachineOf=kube-apiserver.service
```

This will ensure the registration service will "follow" the Kubernetes API service around the cluster.

You can now list the registered Kubernetes Kubelets using the kubecfg client:

```
$ kubecfg list /minions
Minion identifier
----------
10.240.23.37
10.240.12.181
10.240.163.58
```

At this point you are ready to launch pods using the kubecfg command tool, or the Kubernetes API.

## Adding and removing machines

Adding more machines is as easy as starting up more nodes using the `node.yml` cloud-config file. The same is true for removing machines, simply destroy them and Fleet will reschedule the units.
