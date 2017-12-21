---
title: Basic Acceptance Tests
permalink: bat.html
keywords: bat, system tests, BAT
---
## BAT tests

The
[_release/bat](https://github.com/ciao-project/ciao/tree/master/_release/bat)
folder contains a set of BAT tests.  Each set of tests validates a specific part
of the Cloud Integrated Advanced Orchestrator, such as storage, and is
implemented by a separate go package in one of the following sub folders.

```
.
├── base_bat     - Basic tests that verify that the cluster is functional
├── image_bat    - BAT tests for the image service
├── quotas_bat   - Quota related BAT tests
├── storage_bat  - Storage related BAT tests
├── workload_bat - Workload related BAT tests
```

The tests are implemented using the go testing framework.  This is convenient as
this framework is used for Cloud Integrated Advanced Orchestrator's unit tests
and so is already familiar to its developers, it requires no additional
dependencies and it works with the project's existing test case runner, test-cases.

## Set up

The BAT tests require a running Cloud Integrated Advanced Orchestrator cluster
to execute.  This can be a full cluster running on hundreds of nodes or a
Single VM cluster running on a single machine.  For more information about
Single VM see [here](developer.html).

The BAT tests have some dependencies. The device on which they are run must have
qemu-img installed and must also have access to the ceph cluster. The controller
node in a Cloud Integrated Advanced Orchestrator cluster fulfills both of these
requirements so it is often easiest to run the BAT tests from this node. There
is only one node in Single VM clusters, so when using Single VM, simply run the
BAT tests from the device on which you ran setup.sh.

The BAT tests require that certain environment variables have been set before they
can be run:

* `CIAO_CONTROLLER` exports the controller URL
* `CIAO_CLIENT_CERT_FILE` provides the certificate to authenticate against the controller
* `CIAO_ADMIN_CLIENT_CERT_FILE` provides the admin certificate to authenticate against the controller
* `CIAO_CA_CERT_FILE` (optional) use the supplied certificate as the CA

Note if you are using Single VM a script will be created for you called
~/local/demo.sh that initialises these variables to their correct
values for the Single VM cluster.  You just need to source this file
before running the tests, e.g.,

```shell
$ . ~/local/demo.sh
```

## Run the BAT Tests and Generate a Pretty Report

```shell
$ cd $GOPATH/src/github.com/ciao-project/ciao/_release/bat
$ test-cases ./...
```

## DON'T use go test to run the BAT tests!

You might be forgiven for thinking that the easiest way to run all the
BAT tests would be to do the following.

```shell
$ cd $GOPATH/src/github.com/ciao-project/ciao/_release/bat
$ go test -v ./...
```

This does not work.  The reason is that when go test is run with a
wildcard and that wildcard matches multiple packages the tests for all of these
packages are run in parallel.  As all tests are run in the same tenant and some
tests call "ciao delete instance --all" the tests from different packages can
interfere with each other.  The introduction of BAT tests for the evacuate
command complicates things further as tests that evacuate a node are likely to
interfere with other tests as they will cause the instances of those tests to
exit.  As we will always want our tests to run inside singlevm, where there is
only one node, it's unlikely that we will ever be able to run our tests
concurrently.

## Run the BAT Tests and Generate TAP report

```shell
$ cd $GOPATH/src/github.com/ciao-project/ciao/_release/bat
$ test-cases -format tap ./...
```

## Run the BAT Tests and Generate a Test Plan

```shell
$ cd $GOPATH/src/github.com/ciao-project/ciao/_release/bat
$ test-cases -format html ./...
```

## Run a Single Set of Tests

```shell
$ cd $GOPATH/src/github.com/ciao-project/ciao/_release/bat
$ go test -v github.com/ciao-project/ciao/_release/bat/base_bat
```

## Run a Single Test

```shell
$ cd $GOPATH/src/github.com/ciao-project/ciao/_release/bat
$ go test -v -run TestGetAllInstances ./...
```
