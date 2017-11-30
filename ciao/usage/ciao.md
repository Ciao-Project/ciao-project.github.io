---
title: The command line
permalink: ciao.html
keywords: CIAO, command line, tool
---

## Introduction

ciao-cli is the command-line interface for the Cloud Integrated Advanced Orchestrator
(CIAO).  It sends HTTPS requests to the [Ciao controller](https://github.com/ciao-project/ciao/tree/master/ciao-controller)
compute API [endpoints](https://github.com/ciao-project/ciao/blob/master/ciao-controller/compute.go).

## Usage

```shell
ciao-cli: Command-line interface for the Cloud Integrated Advanced Orchestrator (CIAO)

Usage:

	ciao-cli [options] command sub-command [flags]

The options are:

  -alsologtostderr
    	log to standard error as well as files
  -ca-file string
    	CA Certificate
  -ciaoport int
    	ciao API port (default 8889)
  -client-cert-file string
    	Path to certificate for authenticating with controller
  -controller string
    	Controller URL
  -log_backtrace_at value
    	when logging hits line file:N, emit a stack trace
  -log_dir string
    	If non-empty, write log files in this directory
  -logtostderr
    	log to standard error instead of files
  -stderrthreshold value
    	logs at or above this threshold go to stderr
  -tenant-id string
    	Tenant UUID
  -v value
    	log level for V logs
  -vmodule value
    	comma-separated list of pattern=N settings for file-filtered logging


The commands are:

	event
	external-ip
	image
	instance
	node
	pool
	quotas
	tenant
	trace
	volume
	workload

Use "ciao-cli command -help" for more information about that command.
```

## Environment variables

ciao-cli first looks for Cloud Integrated Advanced Orchestrator specific
environment variables to retrieve credentials and networking information:

* `CIAO_CONTROLLER` exports the controller URL
* `CIAO_CLIENT_CERT_FILE` provides the certificate to authenticate against the controller
* `CIAO_CA_CERT_FILE` (optional) use the supplied certificate as the CA

All those environment variables can be set through an rc file.
For example:

```shell
$ cat ciao-cli-example.sh

export CIAO_CONTROLLER=ciao-ctl.intel.com
export CIAO_CLIENT_CERT_FILE=$HOME/auth-testuser.pem
```

Exporting those variables is not compulsory and they can be defined
or overridden from the `ciao-cli` command line.

## Controller certificate

ciao-cli interacts with the controller instance over HTTPS.  As such you
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

Or, alternatively the CA certificate may be specified with the `-ca-file`
command line or with the `CIAO_CA_CERT_FILE` environment variable.

## Priviledged versus non priviledged CIAO users

Administrators of a Cloud Integrated Advanced Orchestrator cluster are
privileged users. They are allowed to run each and every ciao-cli commands. Some
ciao-cli commands are privileged and can only be run by administrators.

Non privileged commands can be run by all users. Administrators will have to specify
a tenant/project UUID through the -tenant-id option in order to specify against which
tenant/project they're running the command:
```shell
$ ciao-cli -tenant-id 68a76514-5c8e-40a8-8c9e-0570a11d035b instance list 
```

 Non privileged users belonging to several tenants/projects will also have to
 specify a tenant/project UUID through the -tenant-id option in order to specify
 against which tenant/project they're running the command:

```shell
$ ciao-cli -tenant-id 68a76514-5c8e-40a8-8c9e-0570a11d035b instance list
```

Non privileged users belonging to only one single tenant/project do not need to
pass the tenant/project UUID or name when running non privileged commands:

```shell
$ ciao-cli instance list
```


## Examples

Let's assume we're running a cluster with the following settings:

* The controller is running at `ciao-ctl.intel.com`
* The admin certificate is stored in `/etc/pki/ciao/auth-admin.pem`
* A user certificate with access to one tenant is in `$HOME/auth-user.pem`

This can be defined through the following rc file:

```shell
$ cat ciao-cli-example.sh

export CIAO_CONTROLLER=ciao-ctl.intel.com
export CIAO_CLIENT_CERT_FILE=$HOME/auth-user.pem
```

### Cluster status (Privileged)

```shell
$ CIAO_CLIENT_CERT_FILE=/etc/pki/ciao/auth-admin.pem ciao-cli node status
```

### List all compute nodes (Privileged)

```shell
$ CIAO_CLIENT_CERT_FILE=/etc/pki/ciao/auth-admin.pem ciao-cli node list -compute
```

### List all CNCIs (Privileged)

```shell
$ CIAO_CLIENT_CERT_FILE=/etc/pki/ciao/auth-admin.pem ciao-cli node list -cnci
```

### List all tenants/projects (Privileged)

```shell
$ CIAO_CLIENT_CERT_FILE=/etc/pki/ciao/auth-admin.pem ciao-cli tenant list -all
```

### List quotas

```shell
$ ciao-cli tenant list -quotas
```

### List consumed resources

```shell
$ ciao-cli tenant list -resources
```

### List all instances

```shell
$ ciao-cli instance list
```

### List all workloads

```shell
$ ciao-cli workload list
```

### Launch a new instance

```shell
$ ciao-cli instance add -workload 69e84267-ed01-4738-b15f-b47de06b62e7
```

### Launch 1000 new instances

```shell
$ ciao-cli instance add -workload 69e84267-ed01-4738-b15f-b47de06b62e7 -instances 1000
```

### Stop a running instance

```shell
$ ciao-cli instance stop -instance 4c46ace5-cf92-4ce5-a0ac-68f6d524f8aa
```

### Restart a stopped instance

```shell
$ ciao-cli instance restart -instance 4c46ace5-cf92-4ce5-a0ac-68f6d524f8aa
```

### Delete an instance

```shell
$ ciao-cli instance delete -instance 4c46ace5-cf92-4ce5-a0ac-68f6d524f8aa
```

### Delete all instances for a given tenant

```shell
$ ciao-cli instance delete -all
```

### List all cluster events (Privileged)

```shell
$ CIAO_CLIENT_CERT_FILE=/etc/pki/ciao/auth-admin.pem ciao-cli event list -all
```

### List all cluster events for a given tenant

```shell
$ ciao-cli event list
```
