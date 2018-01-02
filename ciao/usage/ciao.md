---
title: The command line
permalink: ciao.html
keywords: CIAO, command line, tool
---

## Introduction

`ciao` is the command-line interface for the Cloud Integrated Advanced Orchestrator
(CIAO).  It sends HTTPS requests to the [Ciao controller](https://github.com/ciao-project/ciao/tree/master/ciao-controller)
compute API [endpoints](https://github.com/ciao-project/ciao/blob/master/ciao-controller/compute.go).

The general form is `ciao <verb> <noun>` you can find many examples below.

## Environment variables

`ciao` requires Cloud Integrated Advanced Orchestrator specific
environment variables to retrieve credentials and networking information:

* `CIAO_CONTROLLER` provides the controller URL
* `CIAO_CLIENT_CERT_FILE` provides the certificate to authenticate against the controller
* `CIAO_CA_CERT_FILE` (optional) use the supplied certificate as the CA

All those environment variables can be set through a sourced file.
For example:

```shell
$ cat ciao-example.sh

export CIAO_CONTROLLER=ciao-ctl.intel.com
export CIAO_CLIENT_CERT_FILE=$HOME/auth-testuser.pem
```

## Controller certificate

ciao interacts with the controller instance over HTTPS.  As such you
will need to have the controller CA certificate available in order to make
requests. You can either install the CA certificate system-wide:

* On Fedora:
```shell
$ sudo cp controller_ca_cert.pem /etc/pki/ca-trust/source/anchors/
$ sudo update-ca-trust
```

* On Ubuntu
```shell
$ sudo cp controller_ca_cert.pem /usr/local/share/ca-certificates/controller.crt
$ sudo update-ca-certificates
```

 Or, alternatively the CA certificate can be specified with the
 `CIAO_CA_CERT_FILE` environment variable.

## Priviledged versus non priviledged CIAO users

 Administrators of a Cloud Integrated Advanced Orchestrator cluster are
 privileged users. Some ciao commands are privileged and can only be run by
 administrators.

Non privileged commands can be run by all users. Administrators will have to specify
a tenant UUID through the `CIAO_TENANT_ID` environment variable :
```shell
$ CIAO_TENANT_ID=68a76514-5c8e-40a8-8c9e-0570a11d035b ciao list instances
```

  Non privileged users belonging to several tenants can specify which tenant to
  use in the same way. If no tenant is specified the first in the certificate is
  used.

Non privileged users belonging to only one single tenant do not need to
pass the tenant UUID:

```shell
$ ciao list instances
```


## Examples

 These examples assume you are running in an environment setup by ciao-deploy
 and all the required environment variables it reports are set.
### List all compute nodes (Privileged)

```shell
$ CIAO_CLIENT_CERT_FILE=$CIAO_ADMIN_CLIENT_CERT_FILE ciao list nodes --compute-only
```

### List all CNCIs (Privileged)

```shell
$ CIAO_CLIENT_CERT_FILE=$CIAO_ADMIN_CLIENT_CERT_FILE ciao list cncis
```

### List all tenants/projects (Privileged)

```shell
$ CIAO_CLIENT_CERT_FILE=$CIAO_ADMIN_CLIENT_CERT_FILE ciao list tenants
```

### List quotas

```shell
$ ciao list quotas
```


### List all instances

```shell
$ ciao list instances
```

### List all workloads

```shell
$ ciao list workloads
```

### Launch a new instance

```shell
$ ciao create instance 69e84267-ed01-4738-b15f-b47de06b62e7
```

### Launch 1000 new instances

```shell
$ ciao create instance 69e84267-ed01-4738-b15f-b47de06b62e7 --instances 1000
```

### Stop a running instance

```shell
$ ciao stop instance 4c46ace5-cf92-4ce5-a0ac-68f6d524f8aa
```

### Restart a stopped instance

```shell
$ ciao restart instance 4c46ace5-cf92-4ce5-a0ac-68f6d524f8aa
```

### Delete an instance

```shell
$ ciao delete instance 4c46ace5-cf92-4ce5-a0ac-68f6d524f8aa
```

### Delete all instances for a given tenant

```shell
$ ciao delete instance --all
```

### List all cluster events (Privileged)

```shell
$ CIAO_CLIENT_CERT_FILE=$CIAO_ADMIN_CLIENT_CERT_FILE ciao list events
```

### List all cluster events for a given tenant

```shell
$ ciao list events
```
