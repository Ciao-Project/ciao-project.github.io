---
title: Working with workloads
permalink: workloads.html
keywords: CIAO, command line, tool
---

## Workload examples

A workload can be uploaded into the Cloud Integrated Advanced Orchestrator
datastore via the `ciao` command.

```shell
$ ciao create workload my_workload.yaml
```

A workload consists of two yaml files.  The first
defines the properties and resources the workload needs while the second contains
the cloud-init user data file used to initialise the instance on first boot.
The first yaml file is passed to the `ciao create workload` command as an
argument, e.g., my_workload.yaml above.  It contains a reference to the second file.


## Workload properties yaml file

The workload properties yaml file contains information which define the
workload. This configuration will be used for each instance that is started
with the workload ID.

The first section of the file contains some basic information about the
workload.

```shell
description: "My awesome container workload"
vm_type: docker
image_name: "ubuntu:latest"
```

`description` is just a human readable string which will be
displayed when you list workloads with `ciao list workloads`.

There are two types of workloads - container based, and vm based. The `vm_type`
string specifies whether this workload is a container or a VM. Currently the
Cloud Integrated Advanced Orchestrator supports only docker based containers,
and qemu for vms. Valid values for vm_type are `docker` or `qemu`.

The above example is for a container workload. The `vm_type` is set to `docker`
and the docker hub image name is provided by the `image_name` value.

For VMs, the configuration would be slightly different. `description` is still
provided, however, `vm_type` is set to `qemu`. Additionally, a `fw_type` is
required to specify whether the image that is provided has a UEFI based
firmware, or a legacy firmware. If the workload should be booted from an
image, the image must have already been added to the image service.

```yaml
description: "My favorite pet VM"
vm_type: qemu
fw_type: legacy
disks:
  - source:
       type: image
       source: "73a86d7e-93c0-480e-9c41-ab42f69b7799"
    ephemeral: true
    bootable: true
```

Workloads may also be specified to boot from volumes created in the volume
service. To create a workload which boots from an existing volume include a
definition for a bootable disk.

```yaml
description: "My favorite pet VM"
vm_type: qemu
fw_type: legacy
disks:
- volume_id: "9c858de5-fdd3-42d8-925e-2fdc60768d24"
  bootable: true
```

Disks for the workload may be bootable, or attached. To create a new volume
to be attached to your workload for persistent storage, specify the size
of your new disk in GigaBytes. Newly created disks may be marked as ephemeral
if you wish to not persist the disk after the instance has been destroyed.

```yaml
disks:
- size: 20
  ephemeral: true
```

To attach a volume that has already been created in the volume service,
specify the volume_id of the volume to be attached.

```yaml
disks:
- volume_id: "9c858de5-fdd3-42d8-925e-2fdc60768d24"
```

You can specify in the workload definition whether to create a volume
to either attach or boot from that is cloned from an existing volume or
image by including a `source` definition.

```yaml
disks:
- bootable: true
  source:
     type: volume
     source: "9c858de5-fdd3-42d8-925e-2fdc60768d24"
```

Valid values for the source `type` field are `image` or `volume`. The `source`
field for images can refer to either the image ID or name.

Workload definitions must also contain default values for resources
that the workload will need to use when it runs. There are two resources
which must be specified:

```yaml
requirements:
    vcpus: 2
    mem_mb: 512
```

Finally, the filename for the cloud config file must be included in the
workload definition. This file must be readable by `ciao`.

```yaml
cloud_init: "fedora_vm.yaml"
```

## Specifying the requirements for a workload

As well as being able to specify the amount of memory and number of virtual
CPUs that should be allocated to a workload the requirements section can also
be used to control how an instance of the workload is scheduled.

```yaml
requirements:
    node_id: 1c555de5-fee2-42d8-925e-2fdc47839d24
```

Requires that an instance created from this workload must be scheduled on the
node which has a matching ID.

```yaml
requirements:
    hostname: my-special-node
```

Here rather than using the node ID for a specific node the node's hostname is
used to require that the instance is only started on the node that matches that
hostname.

## Cloud config yaml file

Below is an example cloud config file. Cloud Integrated Advanced Orchestrator
cloud config files have a cloud-config section, followed by a user data section.
These files use the cloud-init userdata format as described
[here](http://cloudinit.readthedocs.io/en/latest/index.html).  The cloud-config
section must be prefaced by the `---` separator.

```yaml
---
#cloud-config
users:
  - name: demouser
    gecos: CIAO Demo User
    lock-passwd: false
    passwd: $6$rounds=4096$w9I3hR4g/hu$AnYjaC2DfznbPSG3vxsgtgAS4mJwWBkcR74Y/KHNB5OsfAlA4gpU5j6CHWMOkkt9j.9d7OYJXJ4icXHzKXTAO.
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh-authorized-keys:
    - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDerQfD+qkb0V0XdQs8SBWqy4sQmqYFP96n/kI4Cq162w4UE8pTxy0ozAPldOvBJjljMvgaNKSAddknkhGcrNUvvJsUcZFm2qkafi32WyBdGFvIc45A+8O7vsxPXgHEsS9E3ylEALXAC3D0eX7pPtRiAbasLlY+VcACRqr3bPDSZTfpCmIkV2334uZD9iwOvTVeR+FjGDqsfju4DyzoAIqpPasE0+wk4Vbog7osP+qvn1gj5kQyusmr62+t0wx+bs2dF5QemksnFOswUrv9PGLhZgSMmDQrRYuvEfIAC7IdN/hfjTn0OokzljBiuWQ4WIIba/7xTYLVujJV65qH3heaSMxJJD7eH9QZs9RdbbdTXMFuJFsHV2OF6wZRp18tTNZZJMqiHZZSndC5WP1WrUo3Au/9a+ighSaOiVddHsPG07C/TOEnr3IrwU7c9yIHeeRFHmcQs9K0+n9XtrmrQxDQ9/mLkfje80Ko25VJ/QpAQPzCKh2KfQ4RD+/PxBUScx/lHIHOIhTSCh57ic629zWgk0coSQDi4MKSa5guDr3cuDvt4RihGviDM6V68ewsl0gh6Z9c0Hw7hU0vky4oxak5AiySiPz0FtsOnAzIL0UON+yMuKzrJgLjTKodwLQ0wlBXu43cD+P8VXwQYeqNSzfrhBnHqsrMf4lTLtc7kDDTcw== ciao@ciao
...
```

The above example shows that the user data section is blank, but the `...`
separator must be present to indicate the end of the cloud-config section.
Be sure to use a cloud-config that is recognized by the host OS you are using
for your workload.
