---
title: Cloud Integrated Advanced Orchestrator (CIAO)
permalink: index.html
keywords: CIAO, overview
toc: false
---
The Cloud Integrated Advanced Orchestrator (CIAO) aims to provide an easy to
deploy, secure, scalable cloud orchestration system which handles virtual
machines, containers, and bare metal apps agnostically as generic workloads.

## Getting started

The easiest way to get to know the Cloud Integrated Advanced Orchestrator is to
install and and play around with it.  It comes with its own easy to setup
development environment that allows you to run an entire cluster inside a single
virtual machine.  This virtual machine can be created and configured by issuing
a single command.  For more information see the [The development
environment](developer.html).

## Setting up a Cluster

Although the Cloud Integrated Advanced Orchestrator development environment is
convenient and easy to set up, to unlock the true potential of this project
you'll need to install it on a real cluster of machines.  Installing on a real
cluster is relatively painless and is performed using a custom tool called
ciao-deploy.  For more information see [Deploying on a real
cluster](ciao-deploy.html).

## Using the Cluster

Cloud Integrated Advanced Orchestrator clusters can be manipulated and inspected
using the [ciao command line tool](ciao.html).  This tool can be used to build,
deploy and manage cloud applications running on a cluster.

A tool called [kubicle](kubicle.html) is also included that can be used to
deploy a Kubernetes (k8s) cluster on top of an existing Cloud Integrated Advanced
Orchestrator cluster.  It automatically creates workloads for the various k8s
roles (master and worker), creates instances from these workloads which
self-form into a k8s cluster, and extracts the configuration information needed
to control the new cluster from the host machine.
