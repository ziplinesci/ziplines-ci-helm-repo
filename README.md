Welcome to the official Helm repository for NextGen CI. This repository contains Helm charts for deploying and managing the core components of the NextGen CI platform.

## Table of Contents

1. [Installation](#installation)
2. [Local Testing](#local-testing)
   - [Prepare your system](#prepare-your-system)
   - [Create a kind cluster](#create-a-kind-cluster)
3. [Deploying the Helm chart](#deploying-the-helm-chart)
4. [Verify Deployment](#verify-deployment)

## Installation

To start using the Helm charts from this repository, add it to your Helm client with the following command:

```shell
helm repo add ziplinesci https://helm-ziplinesci.malsharbaji.com
```

To update and view all available charts in the repository, run:

```shell
helm repo update
helm search repo ziplinesci
```

You can then install or upgrade Helm charts using:

```shell
helm upgrade --install ziplinesci-ci ziplinesci/ziplinesci-ci --namespace ziplinesci --wait
```

## Local Testing

To test these Helm charts on your local machine, you can use [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/), a tool for running local Kubernetes clusters using Docker container nodes. Follow the steps below to set up a local testing environment.


### Prepare Your System

To set up a local Kubernetes cluster with Kind, create a configuration file named `cluster.yaml`. This file defines the cluster configuration, including port mappings and registry settings.

Here is an example of a Kind configuration file:
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = "/etc/containerd/certs.d"
name: tooling
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"

    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
 ```
   Save this content to a file named `cluster.yaml` in your current directory.


### Create a Kind Cluster

Download the [Makefile](kind-cluster/Makefile) in your current directory to create a Kind cluster and set up the local environment.
```shell
make setup-env
```

## Deploying the Helm chart

With the Kind cluster running, you can now deploy the NextGen CI components using Helm. First, ensure your repository is cloned to your local machine.

```shell
helm install ziplinesci-ci ./ziplinesci-ci --namespace ziplinesci --wait`
```
This command will deploy the NextGen CI components into the ziplinesci namespace within your Kind cluster. The --wait flag ensures that Helm waits until all components are successfully deployed.

## Verify Deployment

To verify that the deployment was successful, check the status of the Helm release:

```shell
helm list -n ziplinesci
```
This command lists all Helm releases in the ziplinesci namespace, allowing you to verify the status of your NextGen CI deployment.
