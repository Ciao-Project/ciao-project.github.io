---
title: Creating your first cloud app
permalink: tutorial.html
keywords: tutorial, guestbook, redis, workloads, instances, external IP
---

## Introduction

This tutorial will demonstrate how to create a simple cloud application with the
Cloud Integrated Advanced Orchestrator.  The application in question is the
Guestbook application as used in the Kubernetes tutorial [Example: Deploying PHP
Guestbook application with
redis](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/).
The reader will be introduced to many of the Cloud Integrated Advanced
Orchestrator's concepts and constructs by deploying the Guestbook application on
a Cloud Integrated Advanced Orchestrator cluster.

## The Guestbook application

The Guestbook application is a very simple application.  It consists of two
redis instances, a master and a slave and one PHP application.  The PHP
application presents a very simple web page to the user with a single text box.
Information typed into the text box by the user is stored in the redis master
instance using a specific key.  The application then queries the contents of the
key from the slave instance and displays the value underneath the text box.  If
everything works okay the application demonstrates that the Guestbook PHP
application can talk to both redis instances and that both redis instances can
talk to each other.

For this example we are going to use the Cloud Integrated Advanced
Orchestrator's [development environment](developer.html).  Please familiarize
yourself with this environment before reading any further.

## Setup

In order to view the Guestbook web application from our PC we will need to set up
a port mapping when we launch Configurable Cloud VM (ccloudvm).  This can be done
using the -port parameter.  If you already have a ccloudvm instance running, you
need to stop it and restart it specifying the port mapping, e.g.,

```
$ ccloudvm stop
$ ccloudvm restart -port 8080-80
```

if you have not yet created a ccloudvm instance type

```
$ ccloudvm create ciao -port 8080-80
```

Once your ccloudvm VM has been restarted or created, connect to it using the

```
$ ccloudvm connect
```

## Creating the workloads

Our cloud application will consist of two virtual machines and one container.
The redis master and slave will be deployed as VM instances and the Guestbook
application as a container.  Collectively, containers and VMs are referred to as
instances in the Cloud Integrated Advanced Orchestrator.  Instances are created
from specifications called workloads.  The Cloud Integrated Advanced
Orchestrator development environment ships with some default workloads that are
mainly used for testing.

We're going to need to create some new workloads for our Guestbook
application.  To create a workload you first need to create two separate YAML
files.  The first file contains information about the workload's backing images
and resources.  The second, which is referenced by the first, is a
[cloud-init](http://cloudinit.readthedocs.io/en/latest/) file that describes how
instances created from the workload should be configured.  We'll need to create
three new workloads, one for each of our instances, amounting to 6 YAML files in
total.

The workloads for our VM instances require a backing image from which they will
create their rootfs.  This backing image must be stored in the image service.
The Cloud Integrated Advanced Orchestrator development environment provides a
suitable backing image for us.  However, we need to discover what its UUID is so
that we can refer to it in our workload definitions.  You can discover this UUID
by executing the following command.

{% raw %}
```shell
$ ciao-cli image list -f '{{range (filterContains . "Name" "Ubuntu Server")}}{{println .ID}}{{end}}'
046aa079-7614-494b-9044-06587510213d
```
{% endraw %}

The reported UUID of the Ubuntu Server backing image is 046aa079-7614-494b-9044-06587510213d.
A different UUID will be reported on your own cluster.  You will need to note it down
for future use.

Now we're ready to create our workload definition files.  Create a new directory somewhere inside your ccloudvm VM, e.g., ~/examples.  Enter this directory and create the following files,
being careful to specify the correct UUID for disks->source->id in the redis-master.yaml
and redis-slave.yaml files.

*~/example/redis-master.yaml*

```yaml
description: "Redis Master"
vm_type: qemu
fw_type: legacy
requirements:
    vcpus: 2
    mem_mb: 512
cloud_init: redis-master-cloud.yaml
disks:
  - source:
       type: image
       source: "046aa079-7614-494b-9044-06587510213d"
    ephemeral: true
    bootable: true
```

*~/example/redis-master-cloud.yaml*

```yaml
---
#cloud-config
runcmd:
  - systemctl restart networking
  - apt-get update
  - DEBIAN_FRONTEND=noninteractive DEBCONF_NONINTERACTIVE_SEEN=true apt-get install redis-server -y 
  - sed -i "s/bind 127.0.0.1/bind `hostname -i`/" /etc/redis/redis.conf
  - systemctl restart redis-server
...
```

*~/example/redis-slave.yaml*

```yaml
description: "Redis Slave"
vm_type: qemu
fw_type: legacy
requirements:
    vcpus: 2
    mem_mb: 512
cloud_init: redis-slave-cloud.yaml
disks:
  - source:
       type: image
       source: "046aa079-7614-494b-9044-06587510213d"
    ephemeral: true
    bootable: true
```

*~/example/redis-slave-cloud.yaml*

```yaml
---
#cloud-config
runcmd:
  - systemctl restart networking
  - apt-get update
  - DEBIAN_FRONTEND=noninteractive DEBCONF_NONINTERACTIVE_SEEN=true apt-get install redis-server -y 
  - sed -i "s/bind 127.0.0.1/bind `hostname -i`/" /etc/redis/redis.conf
  - echo "slaveof redis-master 6379" >> /etc/redis/redis.conf
  - systemctl restart redis-server
...
```

Note that if you are behind a corporate proxy you will need to prefix the calls to
apt-get in the cloud.yaml files with `http_proxy=<MY-PROXY-URL>`, replacing
`<MY-PROXY-URL>` with the appropriate values.

*~/example/guestbook.yaml*

```yaml
description: "Guestbook container"
vm_type: docker
image_name: "gcr.io/google-samples/gb-frontend:v4"
requirements:
  vcpus: 2
  mem_mb: 100
cloud_init: "guestbook-cloud.yaml"
```

*~/example/guestbook-cloud.yaml*

```yaml
---
#cloud-config
runcmd:
...
```

Now we need to create the workloads themselves.  This can be done using the
ciao-cli workload create command, as shown below.

```shell
$ ciao-cli workload create --yaml redis-master.yaml
Created new workload: 0996f1fa-17f8-46cf-afea-1c5bf6797a63
$ ciao-cli workload create --yaml redis-slave.yaml
Created new workload: 2757da4f-54d1-446c-99a6-d6644eac927c
$ ciao-cli workload create --yaml guestbook.yaml
Created new workload: c9037a59-939b-435b-b19e-66d044add570
```

We can check everything has worked correctly by enumerating the defined workloads
using the ciao-cli workload list command.  You should see the default workloads
provided with the SingleVM setup in addition to our newly created workloads.

```shell
# ciao-cli workload list
Workload 1
	Name: Ubuntu latest test container
	UUID:dbb0dc45-7d3d-4a11-89fe-9784e52c2e5f
	CPUs: 2
	Memory: 128 MB
Workload 2
	Name: Debian latest test container
	UUID:90b7b092-6b3c-43ed-aef9-22f8c7545b0a
	CPUs: 2
	Memory: 128 MB
Workload 3
	Name: Ubuntu test VM
	UUID:181cae48-aabb-4e02-9586-64e750d5d57c
	CPUs: 2
	Memory: 256 MB
Workload 4
	Name: Redis Master
	UUID:0996f1fa-17f8-46cf-afea-1c5bf6797a63
	CPUs: 2
	Memory: 512 MB
Workload 5
	Name: Redis Slave
	UUID:2757da4f-54d1-446c-99a6-d6644eac927c
	CPUs: 2
	Memory: 512 MB
Workload 6
	Name: Guestbook container
	UUID:c9037a59-939b-435b-b19e-66d044add570
	CPUs: 2
	Memory: 100 MB
```

## Creating the instances

Now we have our workloads we can create our instances.  Instances are created
using the ciao-cli create command.  When we create an instance we need to specify
the UUID of the workload that contains the instance specification.  Our three
Guestbook instances can be created as follows.  Note that the workload UUIDs are
randomly generated so these commands will need to be modified slightly to reflect the
workload UUIDs used in your cluster.

```shell
$ ciao-cli instance add --workload 0996f1fa-17f8-46cf-afea-1c5bf6797a63 -name 'redis-master'
Created new (pending) instance: 2111f605-d02c-489d-acf0-05a10d24833b
$ ciao-cli instance add --workload 2757da4f-54d1-446c-99a6-d6644eac927c -name 'redis-slave'
Created new (pending) instance: 6bcf6caa-a77e-4fe2-b227-c9bd4122340a
$ ciao-cli instance add --workload c9037a59-939b-435b-b19e-66d044add570
Created new (pending) instance: b40f3aab-908f-4377-a6bf-e9b7703a0684
```

When creating instances we have the option of specifying a name.  The
instance will be discoverable via DNS by other instances in the same tenant
using that name.  Note that in the example above we have provided names for the
first two instances.  This is because the Guestbook container refers to the
redis instances by name.  If we did not supply the correct names for the redis
instances when creating them our Guestbook container would not be able to
store or retrieve data.

We can check to see that our instances have started correctly using the
ciao-cli instance list command.

```shell
$ ciao-cli instance list
# UUID                                 Status Private IP SSH IP        SSH PORT
1 2111f605-d02c-489d-acf0-05a10d24833b active 172.16.0.2 198.51.100.84 33002
2 6bcf6caa-a77e-4fe2-b227-c9bd4122340a active 172.16.0.3 198.51.100.84 33003
3 b40f3aab-908f-4377-a6bf-e9b7703a0684 active 172.16.0.4 198.51.100.84 33004
```

You should see that you have three active instances running.

## Creating an external IP

Our Guestbook application is now up and running.  There's only one small problem.
We cannot actually access the application.  This is because our three instances
are running on their own private network.  To expose our application to the
outside world, i.e., our SingleVM development environment, we're going to need
to create an external IP and assign it to the Guestbook container.  There are
three steps involved in creating an external IP and assigning it to an instance.
Firstly, we need to create a new pool for external IPs, secondly we need to add
an IP address to this pool, and finally we need to map that IP to our instance.
This can be done as follows.

```shell
$ CIAO_CLIENT_CERT_FILE=$CIAO_ADMIN_CLIENT_CERT_FILE ciao-cli pool create --name redis
Created new pool: redis
$ CIAO_CLIENT_CERT_FILE=$CIAO_ADMIN_CLIENT_CERT_FILE ciao-cli pool add --name redis 198.51.100.2
Added new address to: redis
$ ciao-cli external-ip map --instance b40f3aab-908f-4377-a6bf-e9b7703a0684 --pool redis
Requested external IP for: b40f3aab-908f-4377-a6bf-e9b7703a0684
```
Don't forget to replace the instance id, b40f3aab-908f-4377-a6bf-e9b7703a0684, with the
ID of the container instance in your cluster.

There are a couple of interesting things to note about these three
commands.

Firstly, the first two commands require administrator privileges.
To execute a command with administrator privileges we need to prove that we are
an administrator.  To do this we need to provide an administrator certificate.
In SingleVM an administrator certificate is automatically created and its
location is stored in the CIAO_ADMIN_CLIENT_CERT_FILE environment variable.
In order to execute a command with administrator privileges we simply need
to set the value of the CIAO_CLIENT_CERT_FILE to point to the admin cert, i.e.,
to the PATH stored in CIAO_ADMIN_CLIENT_CERT_FILE.

Secondly, the IP address we have associated with our Guestbook container instance,
is routable within the SingleVM environment (which is why it was chosen).  It can
be used to access the private IP address of the container, 172.16.0.4, from outside
of the instances' private network, e.g., from the terminal of our SingleVM.

We can see the mapping we've just established as follows.

```shell
$ ciao-cli external-ip list
# ExternalIP   InternalIP InstanceID
1 198.51.100.2 172.16.0.4 b40f3aab-908f-4377-a6bf-e9b7703a0684
```

Now let's verify that our external IP address is working.  We can do this quickly via
the curl command.

{% raw %}
```html
$ curl 198.51.100.2
<html ng-app="redis">
  <head>
    <title>Guestbook</title>
    <link rel="stylesheet" href="//netdna.bootstrapcdn.com/bootstrap/3.1.1/css/bootstrap.min.css">
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.2.12/angular.min.js"></script>
    <script src="controllers.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/angular-ui-bootstrap/0.13.0/ui-bootstrap-tpls.js"></script>
  </head>
  <body ng-controller="RedisCtrl">
    <div style="width: 50%; margin-left: 20px">
      <h2>Guestbook</h2>
    <form>
    <fieldset>
    <input ng-model="msg" placeholder="Messages" class="form-control" type="text" name="input"><br>
    <button type="button" class="btn btn-primary" ng-click="controller.onRedis()">Submit</button>
    </fieldset>
    </form>
    <div>
      <div ng-repeat="msg in messages track by $index">
        {{msg}}
      </div>
    </div>
    </div>
  </body>
</html>
```
{% endraw %}

If you're behind a proxy you will need to add the external IP address 198.51.100.2
to your no_proxy environment variable for this to work.

## Publishing the Guestbook application

Guestbook is now up and running but to really test it we need to view the
application in a browser.  The easiest way to do this is to use the browser
running on your host computer.  At the start of the tutorial we set up a port
mapping for our ccloudvm VM.  We will access the Guestbook application via this
port using the localhost interface on your host computer.  However, before we do
this we need to first set up some IP table rules in the SingleVM to redirect
traffic on port 80 to our container.  This can be done as follows.

```shell
$ sudo iptables -A FORWARD --in-interface ens2 -j ACCEPT
$ sudo iptables -t nat -A PREROUTING -p tcp -i ens2 -m tcp --dport 80 -j DNAT --to-destination 198.51.100.2:80
```

Finally, point your host machine's browser at http://127.0.0.1:8080.  You should
see a simple text box on the screen.  Type some text into the text box and press
submit.  The guestbook application will then set a key on the redis-master with
the value typed into the textbox and will then read that key back from the
slave, outputting the contents to the web page.  You should therefore see the
value you entered in the textbox echoed back onto the page.
