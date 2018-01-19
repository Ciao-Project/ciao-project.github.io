---
title: Installing Kubernetes
permalink: kubicle.html
keywords: kubicle, kubernetes
---

## Introduction

The Cloud Integrated Advanced Orchestrator is a suite of Go programs designed to make it easy to set up and configure a private cloud.  It can orchestrate both containers and VMs in multiple tenants across multiple nodes.  It is designed to be fast and scalable and it enables security by default.  All control plane communication within the cluster is encrypted and each tenant gets its own private network.  All of these traits make the Cloud Integrated Advanced Orchestrator an ideal private undercloud for Kubernetes and the good news is, once you have a running Cloud Integrated Advanced Orchestrator cluster, you can set up a complete k8s cluster with a single command.

In this article we will take a look at the simplest way of installing k8s on top of a Cloud Integrated Advanced Orchestrator cluster which requires only a single machine.  We will do this by

- Installing Go
- Configuring proxies [OPTIONAL] 
- Downloading and building Configurable Cloud VM (ccloudvm)
- Creating a VM in which to run Cloud Integrated Advanced Orchestrator
- Logging into the VM and starting the cluster
- Creating our k8s cluster
- Shutting down our k8s and Cloud Integrated Advanced Orchestrator clusters

## Prerequisites

To follow the instructions below you will need a Linux system with at least 8GB of RAM.  Ideally, running Ubuntu 16.04 as that is what we use to develop the Cloud Integrated Advanced Orchestrator and is the environment in which it is most heavily tested.

The Cloud Integrated Advanced Orchestrator development environment runs inside a VM managed by a tool called ccloudvm which we will introduce shortly.  It will launch some VMs inside the ccloudvm created VM for its own internal use and to host our k8s cluster.  To launch VMs the Cloud Integrated Advanced Orchestrator requires KVM to be enabled, hence KVM must be available inside the ccloudvm VM.  In order for this to work our host machine needs to have nested KVM enabled.  This is enabled by default in Ubuntu 16.04 but may not be enabled on other distributions.  If nested KVM is not enabled you will get an error when running ccloudvm in step 4.

On systems with Intel based CPUs you can verify that nested KVM is enabled by typing

```shell
$ cat /sys/module/kvm_intel/parameters/nested
Y
```

If all is well you should see 'Y' printed out on the console.  If you see 'N' you’ll need to enable nested KVM.  This can be done as follows

```shell
$ echo "options kvm-intel nested=1" | sudo tee /etc/modprobe.d/kvm-intel.conf
```

The code we are going to run will be downloaded using the Go tool chain.  Go has a dependency on git, therefore you must install git if it is not already present on your machine.  This can be done on Ubuntu as follows

```shell
$ sudo apt-get install git
```

## Installing Go

Download the latest stable version of Go from [here]( https://golang.org/dl/  ) and follow the installation instructions.  Go 1.8 or later is required.

You should also ensure that the directory into which Go installs its binaries is present in your path.  You can do this by executing the following command:

```shell
$ export PATH=$PATH:$(go env GOPATH)/bin
```

## Configuring Proxies

{% include important.html content="Skip this section if you do not access the Internet through a proxy." %}

If your computer accesses the Internet through a proxy, you should make sure that the proxy environment variables are correctly configured in your shell.  This is important for two reasons:

The Go command used to download the Cloud Integrated Advanced Orchestrator will fail if it cannot find the proxy.
ccloudvm will replicate your local proxy settings in all the VMs it creates.  It will ensure that proxy environment variables are correctly initialised for all users, and that both docker and the package managers, such as APT, are made aware of the appropriate proxy settings.  If the proxy environment variables are not correctly configured, ccloudvm cannot do this.

So, assuming that you are using a corporate proxy you should enter the following commands, replacing the URLs and the domain names with values that are appropriate to your environment.

```shell
$ export http_proxy=http://my-proxy.my-company.com:port 
$ export https_proxy=http://my-proxy.my-company.com:port
$ export no_proxy=.my-company.com
```

## Downloading and Building ccloudvm

Once Go is installed downloading and installing ccloudvm is easy.  Simply type

```shell
$ go get github.com/intel/ccloudvm
```

## Creating a VM in which to run the Cloud Integrated Advanced Orchestrator

The next step is to create a custom VM for running the Cloud Integrated Advanced Orchestrator.  This might sound complicated, and it is, but luckily the entire process is automated for us by a tool called ccloudvm.  Ccloudvm is a small utility designed to create and manage custom VMs built from cloud images.  To create a VM we need to provide ccloudvm with a set of instructions called a workload.  A workload has already been created, so to make a new VM designed to run the Cloud Integrated Advanced Orchestrator you simply need to type.


```shell
$ ccloudvm create -mem=6000 -cpus=2 ciao
```

This will create a new VM with 6GBs of memory and 2 VCPUs, which is the minimum needed for hosting a k8s cluster inside ccloudvm.

ccloudvm has dependencies on a number of other components, such as qemu.  The first thing it will do when executed is to check to see whether these packages are present on your host computer.  If they are not, it will install them, asking you for your password if necessary. 

The ccloudvm create command has a lot of work to do so it can take some time to run.  In short, it performs the following tasks.

- Installs the dependencies ccloudvm needs on the host machine
- Downloads an Ubuntu 16.04 cloud image
- Creates and boots a new VM based on this image
- Installs all of the dependencies needed by the Cloud Integrated Advanced Orchestrator inside this VM
- Updates the guest OS
- Creates a user account inside the VM with SSH enabled

An edited example ccloudvm output is shown below

```shell
$ ccloudvm create -mem=6000 -cpus=2 ciao
Installing host dependencies
OS Detected: ubuntu
Missing packages detected: [xorriso]
[sudo] password for user: 
Reading package lists...
Building dependency tree...
Reading state information...
The following NEW packages will be installed
  xorriso
….
[SNIP]
….
Missing packages installed.
Downloading Ubuntu 16.04
Downloaded 10 MB of 287
….
[SNIP]
….
Downloaded 287 MB of 287
Booting VM with 6 GB RAM and 2 cpus
Booting VM : [OK]
Adding singlevm to /etc/hosts : [OK]
Mounting /home/markus/go-fork : [OK]
Downloading Go : [OK]
Unpacking Go : [OK]
Installing apt-transport-https and ca-certificates : [OK]
Add docker GPG key : [OK]
Add Google GPG key : [OK]
Retrieving updated list of packages : [OK]
Installing Docker : [OK]
Installing kubectl : [OK]
Installing GCC : [OK]
Installing Make : [OK]
Installing QEMU : [OK]
Installing xorriso : [OK]
Installing ceph-common : [OK]
Installing Openstack client : [OK]

Auto removing unused components : [OK]
Building ciao : [OK]
Installing Go development utils : [OK]
Pulling ceph/demo : [OK]
Downloading Fedora-Cloud-Base-24-1.2.x86_64.qcow2 : [OK]
Downloading xenial-server-cloudimg-amd64-disk1.img : [OK]
Downloading CNCI image : [OK]
Downloading latest clear cloud image : [OK]
Setting git user.name : [OK]
Setting git user.email : [OK]
VM successfully created!
Type ccloudvm connect to start using it.
```

Please see the [Troubleshooting](kubicle.html#trouble) section near the bottom of this document if this command fails.

## Starting the Cloud Integrated Advanced Orchestrator Cluster

Now our VM has been created we need to log into it and start our cluster.  This is easily done.  To log into the VM simply type

```shell
$ ccloudvm connect
```

This will connect to the VM via SSH using a private key created specifically for this VM when we ran ccloudvm create.  You will be presented with a set of instructions explaining how to start the ciao cluster.

```
Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.4.0-83-generic x86_64)

To run Single VM:

cd /home/<user>/go-fork/src/github.com/ciao-project/ciao/testutil/singlevm
./setup.sh
```

Follow the instructions for running Single VM, replace <user> with your user name, e.g.,

```shell
$ cd /home/<user>/go-fork/src/github.com/ciao-project/ciao/testutil/singlevm
$ ./setup.sh
```

If everything works okay the script should terminate with output that looks similar to the following.

```
Your ciao development environment has been initialised.
To get started run:

. ~/local/demo.sh

Verify the cluster is working correctly by running

~/local/verify.sh

Use ciao to manipulate and inspect the cluster, e.g., 

ciao create instance ab68111c-03a6-11e6-87de-001320fb6e31 --instances=1

When you're finished run the following command to cleanup

~/local/cleanup.sh
```

To communicate with the newly created Cloud Integrated Advanced Orchestrator cluster we need to initialise some environment variables.  This can be done by sourcing the file ~/local/demo.sh, e.g.,

```shell
$ . ~/local/demo.sh
```

We can then run a few simple commands to check that everything is working correctly.  To get started let’s enumerate the list of workloads.  A workload is a set of instructions for creating an instance (VM or a container).  Our new cluster comes with some predefined workloads, which you can see if you execute

```shell
$ ciao list workloads
ID                                   Name                         CPUs    Mem     
73250276-5f2d-4d22-840a-b8faec63ba0d Clear Linux test VM          2       128     
9e562fbc-4c26-4ed9-a2c8-8e2b9806f1af Ubuntu latest test container 2       128     
ae3d16e7-3b7a-47f9-a27a-713a4c4af6d0 Debian latest test container 2       128     
3afb3866-ef88-4289-9369-d550af67a2ea Ubuntu test VM               2       256    
```

There are two workloads for creating container instances and one for creating a VM instance.  We’ll be creating some new VM workloads later on for our k8s master and worker nodes.

Now let’s enumerate the list of instances.  This can be done as follows

```shell
$ ciao list instances
```

You should see that there are no instances.  This will change when we set up our k8s cluster.

Finally, should you get an error when running the setup.sh script to start ciao you could try tearing the cluster down using, ~/local/cleanup.sh and then re-running setup.sh.

## Creating a k8s Cluster

We’re going to set up our k8s cluster using another tool called kubicle.  Kubicle is a command line tool for creating Kubernetes clusters on top of an existing Cloud Integrated Advanced Orchestrator cluster. It automatically creates Cloud Integrated Advanced Orchestrator workloads for the various k8s roles (master and worker), creates instances from these workloads which self-form into a k8s cluster, and extracts the configuration information needed to control the new cluster from the master node.  Kubicle is installed by the setup.sh script we’ve just run.

Creating a Kubernetes cluster is easy. Kubicle only needs one piece of information, the name of the image to use for the k8s nodes. Currently, this name must refer to an Ubuntu server image, as the workloads created by kubicle for the k8s nodes assume Ubuntu.   Luckily the setup.sh script we ran earlier uploads an Ubuntu server image into ciao’s image service for us. 

Now we simply need to run the kubicle create command specifying the name of the above image, e.g.,

```shell
$ kubicle create --external-ip=198.51.100.2 ubuntu-server-16.04
Creating master
Creating workers
Mapping external-ip
Instances launched.  Waiting for k8s cluster to start

k8s cluster successfully created
--------------------------------
Created master:
 - f6c494d1-e4b4-4c26-a663-0dab4dfb15db
Created 1 workers:
 - aaed1545-2125-4c59-8711-583c26c3299b
Created external-ips:
- 198.51.100.2
Created pools:
- k8s-pool-46959cfe-f584-45f1-9218-50ea3549a0ee
To access k8s cluster:
- export KUBECONFIG=$GOPATH/src/github.com/ciao-project/ciao/testutil/singlevm/admin.conf
- If you use proxies, set
  - export no_proxy=$no_proxy,198.51.100.2
  - export NO_PROXY=$NO_PROXY,198.51.100.2
```

You shouldn’t change anything else, i.e., include the --external-ip=198.51.100.2 option verbatim.  The --external-ip option provides an IP address that can be used to administer the k8s cluster and to access services running within it.  The address 198.51.100.2 is safe to use inside a ccloudvm VM created to run the Cloud Integrated Advanced Orchestrator.

Looking at the output of the kubicle command we can see that it has created a number of objects for us.  It has created two new workloads, one for the master and one for the workers.  From these workloads it has created two VM instances, one master node and one worker node.  Finally, it has created an external ip address for us which it has associated with the master node.  We’ll use this address to access the k8s cluster a little later.  Let’s inspect these new objects using the ciao tool.  If you execute `ciao list workloads`, you should now see five workloads, the final two of which have just been created by kubicle.

```shell
$ ciao list workloads
ID                                   Name                         CPUs    Mem     
73250276-5f2d-4d22-840a-b8faec63ba0d Clear Linux test VM          2       128     
9e562fbc-4c26-4ed9-a2c8-8e2b9806f1af Ubuntu latest test container 2       128     
ae3d16e7-3b7a-47f9-a27a-713a4c4af6d0 Debian latest test container 2       128     
3afb3866-ef88-4289-9369-d550af67a2ea Ubuntu test VM               2       256     
345568aa-a14b-4e01-8ed6-af42a7b220e9 k8s master                   1       1024    
ca3eaf77-2542-4e3c-8cc9-bf8202196d4e k8s worker                   1       2048    

```

Now let’s enumerate our instances, i.e. our running VMs and containers using the `ciao list instances` command.  You may remember that the last time we ran this command there was no output.  Running it again should show that we have two VM instances running.

```shell
$ ciao list instances
ID                                   Name    Status  SSHIP         SSHPort 
6db9f5ac-3406-4e74-9a42-18bb2c13cb08         active  198.51.100.91 33003   
aba48378-be3f-4499-abf7-9e7cb6b2511d         active  198.51.100.91 33002  
```

We can manipulate our newly formed k8s cluster using the kubectl tool.  Kubectl expects an environment variable, KUBECONFIG, to be set to the path of a configuration file which contains the configuration settings for the cluster.  Luckily for us, kubicle creates a configuration file for our new cluster and even provides us with the command needed to initialise KUBECONFIG.  If you scroll back up to where you executed the kubicle command you should see the following.

```shell
To access k8s cluster:
- export KUBECONFIG=$GOPATH/src/github.com/ciao-project/ciao/testutil/singlevm/admin.conf
```

Execute this command.  If your ccloudvm instance is running behind a proxy, you will also need to add the external-ip address we specified earlier to your no_proxy settings.  The reason for this is that kubectl will access the k8s cluster via this external ip address and we don’t want this access to go through a proxy.  Again the status message printed by the kubicle create command provides us with the commands we need to execute.

```shell
- If you use proxies, set
  - export no_proxy=$no_proxy,198.51.100.2
  - export NO_PROXY=$NO_PROXY,198.51.100.2
```

We’re now ready contact our k8s cluster.  First let’s check the nodes have successfully joined the cluster.

```shell
$ kubectl get nodes
NAME                                   STATUS    AGE       VERSION
0d112fc1-8b4b-427c-b258-e93c1ad989e6   Ready     5m        v1.6.7
38ec8bd5-3c50-477f-9b59-b8b332609551   Ready     6m        v1.6.7
```

They have and they’re ready to use.  Note it can take up to a minute for worker nodes to join the k8s cluster and transition to the Ready state.   If you only see one node listed, wait half a minute or so and try again.

Now we've got our kubectl tool running let's create a deployment

```shell
$ kubectl run nginx --image=nginx --port=80 --replicas=2
deployment "nginx" created
$ kubectl get pods
NAME                    READY     STATUS              RESTARTS   AGE
nginx-158599303-0p5dj   0/1       ContainerCreating   0          6s
nginx-158599303-th0fc   0/1       ContainerCreating   0          6s
```

So far so good. We've created a deployment of nginx with two pods. Let's now expose that deployment via an external IP address. There is a slight oddity here. We don't actually specify the external IP we passed to the kubicle create command. Instead we need to specify the internal IP address of the master node, which is associated with the external IP address we passed to the create command. The reason for this is due to the way the Cloud Integrated Advanced Orchestrator implements external IP addresses. Its networking translates external IP addresses into internal IP addresses and for this reason our k8s services need to be exposed using the internal addresses. To find out which address to use, execute the `ciao list external-ips` command, e.g.,

```shell
$ ciao list external-ips
# ExternalIP   InternalIP InstanceID
1 198.51.100.2 172.16.0.2 38ec8bd5-3c50-477f-9b59-b8b332609551
```

You can see from the above command that the internal IP address associated with the external IP address we specified earlier is 172.16.0.2. So to expose the deployment we simply need to type.

Note: Here 172.16.0.2 is an IP address that is routable within the tenant network created by the Cloud Integrated Advanced Orchestrator for the Kubernetes cluster. The IP 198.51.100.2 is the IP address outside of the isolated tenant network with which, the internal IP 172.16.0.2 can be accessed. In a normal setup the IP 198.51.100.2 is either routable at the data center level, or in some cases may be exposed to the internet.

```shell
$ kubectl expose deployment nginx --external-ip=172.16.0.2
service "nginx" exposed
```

The nginx service should now be accessible outside the k8s cluster via the external-ip. You can verify this using curl, e.g.,

```shell
$ curl 198.51.100.2
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
```

## Tearing Down the Clusters

Kubicle created k8s clusters can be deleted with the kubicle delete command. This could of course be done manually, but it would be tedious to do so, particularly with large clusters. Instead use kubicle delete which deletes all the objects (instances, volumes, workloads, pools and external-ips) created to support the k8s cluster in the correct order. For example, to delete the cluster created above simply type.

```shell
$ kubicle delete
External-ips deleted:
198.51.100.2

Pools Deleted:
K8s-pool-46959cfe-f584-45f1-9218-50ea3549a0ee

Workloads deleted:
a92ce5be-922d-445a-ab3c-8f8a42e043ac
ead96d3d-b23f-4c43-9921-4756d274e51d

Instances deleted:
f6c494d1-e4b4-4c26-a663-0dab4dfb15db
38ec8bd5-3c50-477f-9b59-b8b332609551
```

To delete the entire ccloudvm VM simply log out of the VM and type

```shell
$ ccloudvm delete
```

on your host.

## Troubleshooting {#trouble}

The ccloudvm create command can sometimes fail.  The most common causes of this failure are discussed below.

### Failure to Access /dev/kvm

One cause of failure is that the user running ccloudvm does not have the required permissions
to access /dev/kvm.  If this is the case you will see and error message similar to the one
below.

```shell
Booting VM with 6 GB RAM and 2 cpus
Failed to launch qemu : exit status 1, Could not access KVM kernel module: Permission denied
failed to initialize KVM: Permission denied
```

You can resolve this problem by adding yourself to the group of users permitted to access this
device.  Assuming that this group is called kvm you would execute

```shell
$ sudo gpasswd -a $USER kvm
```

### Port Conflict

 By default, ccloudvm maps a port on your computer's network interface to a port
 on the VM it creates. One of these ports, 10022, is used for SSH access. This
 port mapping is necessary to access these services in the VM from the host. For
 example, the ccloudvm connect command is implementing by executing

```shell
$ ssh -q -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i /home/$USER/.ccloudvm/id_rsa 127.0.0.1 -p 10022
```

on the host.  There's a potential problem here.  If either of these ports are already taken by some other service on
your computer, ccloudvm create will fail, e.g.,

```shell
Installing host dependencies
OS Detected: ubuntu
Downloading Ubuntu 16.04
Booting VM with 6 GB RAM and 2 cpus
Failed to launch qemu : exit status 1, qemu-system-x86_64: -net user,hostfwd=tcp::10022-:22: could not set up host forwarding rule 'tcp::10022-:22'
qemu-system-x86_64: -net user,hostfwd=tcp::10022-:22: Device 'user' could not be initialized

```

Here we can see that port 10022 is already taken.  Going forward we will modify ccloudvm to dynamically select available host ports.  In the meantime however, we can work around this problem by overriding the default ports on the command line, as follows:

```shell
$ ccloudvm create -mem=6000 -cpus=2 -port "10023-22" ciao
```

