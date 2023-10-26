# Overview

The Platform External path allows providers to self-serve the integration of
Kubernetes components in OpenShift/OKD without the need to modify any core payload
and without the need for direct involvement of OpenShift engineering.

What are external cloud providers in Kubernetes?

Historically, all cloud providers in Kubernetes were found in the main Kubernetes repository.
However, Kubernetes aims to be ubiquitous and this means supporting a great many infrastructure
providers. Doing this all from a single monolithic repository (and a single monolithic kube-controller-manager
binary) was deemed something that wouldn’t scale, and so in 2017, the Kubernetes community began working
on support for out-of-tree cloud providers. These out-of-tree providers were initially aimed at
allowing the community to develop cloud providers for new, previously unsupported infrastructure providers,
but as the functionality matured, the community decided to migrate all of the current in-tree cloud providers
to external cloud providers too. You can read more about this in the
[Kubernetes blog](https://kubernetes.io/blog/2019/04/17/the-future-of-cloud-providers-in-kubernetes/)
and the [Kubernetes documentation](https://kubernetes.io/docs/concepts/architecture/cloud-controller/).

The external platform in OpenShift sets the `--cloud-provider` flag to `external` on Kubernetes components
(Kubelet and Kube Controller Manager) to signalize the use of external cloud providers,
allowing partners to extend providers' components like Cloud Controller Manager to the OpenShift platform.

To learn more about the feature, we encourage you to read the OpenShift Enhancement Proposal
["Introduce new platform type `External` in the OpenShift specific Infrastructure resource"](https://github.com/openshift/enhancements/blob/master/enhancements/cloud-integration/infrastructure-external-platform-type.md).

To begin learning about the process, we are providing the following pages to achieve your goal:

- [Installing a Platform External OpenShift cluster](./installing.md): A generic
  guide exploring the components and steps to provision an OpenShift cluster
- [Use Case: Installing an OpenShift cluster in Oracle Cloud Infrastructure](./use-case-oci-upi.md)