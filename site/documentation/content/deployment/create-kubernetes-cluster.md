+++
title = "Setting up a Kubernetes Cluster"
weight = 480
+++

This guide describes how to set up a Kubernetes cluster which can be used to run Eclipse Hono.

<!--more-->

Hono can be deployed to any Kubernetes cluster running version 1.11 or newer. This includes [OpenShift (Origin)](https://www.okd.io/) which is built on top of Kubernetes.

The [Kubernetes web site](https://kubernetes.io/docs/setup/) provides instructions for setting up a (local) cluster on bare metal and/or virtual infrastructure from scratch or for provisioning a managed Kubernetes cluster from one of the popular cloud providers.

<a name="Local Development"></a>

## Setting up a local Development Environment

The easiest option is to set up a single-node cluster running on a local VM using the [Minikube](https://github.com/kubernetes/minikube) project.
This kind of setup is sufficient for evaluation and development purposes.
Please refer to Kubernetes' [Minikube setup guide](https://kubernetes.io/docs/setup/minikube/) for instructions on how to set up a cluster locally.

The recommended settings for a Minikube VM used for running Hono's example setup are as follows:

- **cpus**: 2
- **memory**: 8192 (MB)

The command to start the VM will look something like this:

```sh
minikube start --cpus 2 --memory 8192
```

After the Minikube VM has started successfully, the `minikube tunnel` command should be run in order to support Hono's services being deployed using the _LoadBalancer_ type. Please refer to the [Minikube Networking docs](https://github.com/kubernetes/minikube/blob/master/docs/networking.md#access-to-loadbalancer-services-using-minikube-tunnel) for details.

{{% note title="Supported Kubernetes Versions" %}}
Minikube will use the most recent Kubernetes version that was available when it has been compiled by default. Hono _should_ run on any version of Kubernetes starting with 1.13.6. However, it has been tested with several specific versions only so if you experience any issues with running Hono on a more recent version, please try to deploy to 1.13.6 before
raising an issue. You can use Minikube's `--kubernetes-version` command line switch to set a particular version.
{{% /note %}}

## Setting up a Production Environment

Setting up a multi-node Kubernetes cluster is a more advanced topic. Please follow the corresponding links provided in the [Kubernetes documentation](https://kubernetes.io/docs/setup/#production-environment).
