---
title: The ciao project
permalink: index.html
toc: false
---
## Ciao

Ciao is the "Cloud Integrated Advanced Orchestrator".  Its goal is
to provide an easy to deploy, secure, scalable cloud orchestration
system which handles virtual machines, containers, and bare metal apps
agnostically as generic workloads.

## Getting started

The easiest way to get to know ciao is to install and and play
around with it.  Ciao comes with its own easy to setup development environment
that allows you to run an entire ciao cluster inside a single virtual machine.
This virtual machine can be created and configured by issuing a single command.
For more information see the [The ciao development environment](developer.html).

## Deploying ciao

Although the ciao development environment is convenient and easy to set up, to
unlock the true potential of ciao you'll need to install it on a real cluster of
machines.  Installing ciao on a real cluster is relatively painless and is
performed using a custom tool called ciao-deploy.  For more information see
[Deploying ciao on a real cluster](ciao-deploy.html).

## Using ciao

Ciao clusters can be manipulated and inspected using the
[ciao command line tool](ciao.html).  This tool can be used to build, deploy
and manage cloud applications on a ciao cluster.

Ciao also includes a tool called [kubicle](kubicle.html) that can be used to
deploy a Kubernetes cluster on top of an existing ciao cluster.