---
title: Docker in Environments
description: Learn how to enable support for secure Docker inside Environments.
---

If you're a site admin or a site manager, you can enable [container-based
virtual machines (CVMs)](../../environments/cvms.md) as an environment
deployment option. CVMs allow users to run system-level programs, such as Docker
and systemd, in their environments.

## Infrastructure Requirements

- The Kubernetes Nodes must have a minimum kernel version of **5.4** (released
  Nov 24th, 2019).
- The Kubernetes Nodes must be running Ubuntu.
- The cluster must allow privileged containers and `hostPath` mounts. Read more
  about why this is still secure [here](#security).

**Note:** Coder doesn't support legacy versions of cluster-wide proxy services
such as Istio.

## Enabling CVMs in Coder

1. Go to **Manage > Admin > Infrastructure**.
2. Toggle the **Enable Container-Based Virtual Machines** option to **Enable**.

## Setting Up Your Cluster

The following sections show how you can set up your K8 clusters hosted by Google
and Amazon to support CVMs.

### Google Cloud Platform w/ GKE

If your cluster is configured as follows

- GKE Master version `>= 1.17`
- `node-version >= 1.17` and `image-type = "UBUNTU"`

Then run this snippet to create your node pool:

```bash
gcloud beta container clusters create "coder-cluster" \
    --image-type "UBUNTU" \
    --node-version "1.17.14-gke.1600" \
    ...
```

### Amazon Web Services w/ EKS

If your config file defines a `nodeGroup` where `amiFamily >= "Ubuntu1804"`,
then update your `eksctl` config spec with the following to create your node
pool:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata: 
  version: "1.17"
  ...
nodeGroups:
  - name: coder-node-group
    amiFamily: Ubuntu1804
    ...
```

## Security

The [Container-based Virtual Machine](../../environments/cvms.md) deployment
option leverages the [sysbox container
runtime](https://github.com/nestybox/sysbox) to offer a VM-like user experience
while retaining the footprint of a typical container.

Coder first launches a supervising container with additional privileges. This
container is standard and included with the Coder release package. During the
environment build process, the supervising container launches an inner container
using the [sysbox container runtime](https://github.com/nestybox/sysbox). This
inner container is the user’s [environment](../../environments/index.md).

The user cannot gain access to the supervising container at any point. The
isolation between the user's environment container and its outer, supervising
container is what provides [strong
isolation](https://github.com/nestybox/sysbox/blob/master/docs/user-guide/security.md).

## Image Configuration

The following sections show how you can configure your image to include systemd
and Docker for use in CVMs.

### systemd

If your image's OS distribution doesn't link the `systemd` init to
`/sbin/init`, you'll need to do this manually in your Dockerfile.

The following snippet shows how you can specify `systemd` as the init in your
image:

```Dockerfile
FROM ubuntu:20.04
RUN apt-get update && apt-get install -y \
    build-essential \
    systemd

# use systemd as the init
RUN ln -s /lib/systemd/systemd /sbin/init
```

When you start up an environment, Coder checks for the presence of `/sbin/init`
in your image. If it exists, then Coder uses it as the container entrypoint with
a `PID` of 1.

### Docker

To add Docker, install the `docker` packages into your image. For a
seamless experience, use [systemd](#systemd) and register the `docker` service
so `dockerd` runs automatically during initialization.

The following snippet shows how your image can register the `docker` services in
its Dockerfile.

```Dockerfile
FROM ubuntu:20.04
RUN apt-get update && apt-get install -y \
    build-essential \
    git \
    bash \
    docker.io \
    curl \
    sudo \
    systemd

# Enables Docker starting with systemd
RUN systemctl enable docker

# use systemd as the init
RUN ln -s /lib/systemd/systemd /sbin/init
```
