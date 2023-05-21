# Installing a cluster with Platform External

> TODO describe steps to install a cluster, covering the official documentation teaching the required infra stacks (Network, DNS, Load Balancer, etc) to be deployed, then the steps to customize when creating the manifests prior the cluster deployment

Table of Contents:

- [Overview]()
- [Prerequisites]()
- [Create Infrastructure resources]()
    - [Identity]()
    - [Network]()
    - [DNS]()
    - [Load Balancers]()
- [Setup OpenShift installation]()
    - [Create the install-config.yaml]()
    - [Create the manifests]()
    - [Patch the manifests]()
    - [Create ignition files]()
- [Create compute nodes]()
    - [Bootstrap]()
    - [Control Plane]()
    - [Compute/workers]()
        - [Approve certificates]()
- [Review the installation]()
- [Next]()


## Overview

> ToDo

## Prerequisites

### Pull Secret

> Steps to login to the Console and download the pull secret file

### Clients

- installer client
- oc client

### Upload the RHCOS image

> ToDo: show how to retrieve existing formats of RHCOS images from Stream (openshift-install)

### Setup the Provider Account

> ToDo: Provide general instructions and best practices when preparing the account to install OCP

- Use least privileges
- Segregate by containers (Resource Group, Compartments, or similar for cloud specific)

## Create Infrastructure resources

### Identities

### Network

### DNS

### Load Balancers

## Prepare the installation

### Create the install-config.yaml


Create the file `install-config.yaml`:

```yaml
apiVersion: v1
baseDomain: myprovider.com

# Control Plane
controlPlane:
    architecture: amd64
    hyperthreading: 'Enabled'
    name: master
    replicas: 3

# Compute Pool
compute:
  - architecture: amd64
    hyperthreading: Enabled
    name: worker
    replicas: 3
    platform: {}

metadata:
  name: cluster-name

platform:
  external:
    platformName: "providerName"

publish: External
pullSecret: '{"auths":{"cloud.openshift.com":[...]'
sshKey: |
    ssh-rsa AAAAB[...]
```


### Create manifests

#### Create custom manifests

> TODO: The most important part: steps describing how to patch the custom manifests to install components like CCM, CSI, etc

### Create ignition files

## Create compute nodes

### Bootstrap

### Control Plane

### Compute/workers

#### Approve certificates

## Next

- [Run conformance tests on your custom installation](./use-case-test-with-opct.md)
- [Check the use case of deploying OpenShift cluster on Oracle Cloud Infrastructure using UPI method with Platform External](./use-case-oci-upi.md)
- [Check the Use case of deploying OpenShift cluster on Oracle Cloud Infrastructure using Platform External with Assisted Installer](./use-case-oci-assisted.md)