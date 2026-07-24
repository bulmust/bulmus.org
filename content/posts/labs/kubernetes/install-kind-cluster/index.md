---
title: "How to Install Kind Cluster with Gateway?"
date: 2026-07-24T19:10:57+03:00
description:
menu:
  sidebar:
    name: Install Kind Cluster
    identifier: install-kind-cluster
    parent: kubernetes
    weight: 10
hero: images/hero.png
tags:
  - kubernetes
  - kubernetes-kind
  - kubernetes-gateway
categories:
  - Basic
created: 2026-07-24 18:26:28
updated: 2026-07-24 19:11:07
---

1. Install kind cluster if it is not installed
2. Install cloud-provider-kind

## Install Kind Cluster

Before installation, check the latest version from [official kind repository](https://github.com/kubernetes-sigs/kind/releases).

Install binary.

```bash
# ======================
# CHECK THIS VERSION
KIND_VERSION=v0.32.0
# ======================

# Download
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/${KIND_VERSION}/kind-linux-amd64
# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/${KIND_VERSION}/kind-linux-arm64

# Make it executable
chmod +x ./kind
# Move it to common binary
sudo mv ./kind /usr/local/bin/kind
```

Create kubernetes kind cluster for fast testing purpose.

```bash
# Determine cluster name
export CLUSTER_NAME=testing
# Delete that named kind cluster if it is exists
kind delete cluster --name $CLUSTER_NAME

# Create Kind Cluster with only one control-plane
cat << EOF | kind create cluster --name $CLUSTER_NAME --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
EOF
```

## Install Cloud Provider Kind

_KIND has demonstrated to be a very versatile, efficient, cheap and very useful tool for Kubernetes testing. However, KIND doesn't offer capabilities for testing all the features that depend on cloud-providers, specifically the Load Balancers, causing a gap on testing and a bad user experience, since is not easy to connect to the applications running on the cluster._ [^1]

`LoadBalancer` is not strictly required for Kubernetes `Gateway` for local testing such as kind but in cloud environments it is usually recommended.

You can install it using `go install sigs.k8s.io/cloud-provider-kind@latest`, `brew install cloud-provider-kind` or docker install which I prefer.

Get the final version number of cloud-provider-kind from [official github repo](https://github.com/kubernetes-sigs/cloud-provider-kind/releases/latest).

```bash
# ======================
# CHECK THIS VERSION
VERSION="v0.11.1"
# ======================
# Run docker container
docker run -d --name cloud-provider-kind --rm --network host -v /var/run/docker.sock:/var/run/docker.sock registry.k8s.io/cloud-provider-kind/cloud-controller-manager:${VERSION}
```

You can check gateway related crds.

```bash
kubectl get crds|grep gateway.networking
```

## References

[^1] : https://github.com/kubernetes-sigs/cloud-provider-kind#kubernetes-cloud-provider-for-kind
