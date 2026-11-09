# Installing a cluster with Platform External and CAPI

<!--
A quickly starting guide describing how to install an platform external cluster with CAPI, the Enhancement proposal, the installSource (local or image), etc
-->

Provisioning an OpenShift cluster with platform type External using CAPI provider implementation can be achieved through a shared responsability model, where:
- The installer will develop and maitain all configurations to deploy OpenShift cluster, and provide interfaces o openshift-install for CAPI provider implementation and required provider code implementation, not covered by CAPI, be executed in seemsly fully automated experience, a.k.a IPI
- The Provider/Integrator will develop, test, and maitain the CAPI provider implementation with all required interface implementation, a.k.a installer hooks, encapsulated through a CAPI provider payload image, to allow openshift-install program to deliver a fullty automated experience to the cluster creator.


The shared responsability mode is described in this slide deck: https://htmlpreview.github.io/?https://raw.githubusercontent.com/mtulio/enhancements/refs/heads/platform-external-capi/enhancements/cloud-integration/install-external-platform-type-with-capi.html

To get started, the following documentation are created to the integrator to help you building your own payload image:
- [CAPI Payload Image](./capi-payload-integrator.md)
- [Install OCP cluster on a reference provider (integrator helper)](./capi-ref-image-aws.md)