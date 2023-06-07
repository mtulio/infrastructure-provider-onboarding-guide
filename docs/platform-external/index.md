# Overview

<!-- > TODO/placeholder: provider overview of what Platform External is and goal of this section

- What is Platform External?
- When use Platform External?

... -->

The Platform `External` integration path allows third parties to self-serve and integrate with OpenShift without the need to modify any core payload components and without the need for direct involvement of OpenShift engineering.

The `External` platform type is aligned with Kubernetes upstream, and was created to simplify the integration process for new cloud providers in OpenShift/OKD.

This work is splitted into phases, the initial version is available on OCP 4.14+ which allows provides to install OpenShift cluster supplying the cloud-provider's Cloud Controller Manager (CCM) when the cluster is initialized.

To read more about the feature, we encorage you to read the OpenShift Enhancement Proposal ["Introduce new platform type "External" in the OpenShift specific Infrastructure resource"](https://github.com/openshift/enhancements/blob/master/enhancements/cloud-integration/infrastructure-external-platform-type.md).

To begin learning about the process, we are providing the following pages to achieve your goal:

- [Installing a Platform External OpenShift cluster](./installing.md): A generic guide exploring the components and steps to provision an OpenShift cluster
- [Use Case: Installing an OpenShift cluster in Oracle Cloud Infrastructure using User-provisioned Infrastructure](./use-case-oci-upi.md)