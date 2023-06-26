# Installing a cluster with Platform External

The steps to install an OpenShift cluster with "External" platform type must follow the same infrastructure requirements of the official documentation to ["Installing a cluster on any platform"](https://docs.openshift.com/container-platform/4.13/installing/installing_platform_agnostic/installing-platform-agnostic.html).

The `install-config.yaml` configuration must be changed for the `External` platform type. and the providers' manifests must be placed into the install directory before creating the ignitions. We will explore the steps below.


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


## Prerequisites

### Clients

The clients and pull secrets must be downloaded before you install OpenShift, see the steps described in ["Obtaining the installation program"](https://docs.openshift.com/container-platform/4.13/installing/installing_platform_agnostic/installing-platform-agnostic.html).

### Upload the RHCOS image

> ToDo: show how to retrieve existing formats of RHCOS images from Stream (openshift-install)

The Red Hat Core OS image is based on different types. You must choose the supported into your cloud provider. You can obtain the URL for each platform, architecture, and type using the following command:

~~~bash
./openshift-install coreos print-stream-json
~~~

For example, to download an image format `QCOW2` built for `x86_64` architecture to the `OpenStack` platform, you can use the following command:

~~~bash
wget $(./openshift-install coreos print-stream-json | jq -r '.architectures["x86_64"].artifacts["openstack"].formats["qcow2.gz"].disk.location')
~~~

You must upload the downloaded image to your cloud provider image service and use it when creating virtual machines.

### Setup the Provider Account

<!-- > ToDo: Provide general instructions and best practices when preparing the account to install OCP

- Use the least privileges
- Segregate by containers (Resource Group, Compartments, or similar for cloud-specific) -->

The account setup may vary from the cloud provider and the architecture you are deploying OpenShift. Some providers segregate the resources by containers, resource groups, compartments, and accounts.

The agnostic installations do not require OpenShift integration with the cloud provider, it will not require any additional cloud provider account. You must check the documentation for the components you are extending in OpenShift, for example, Cloud Controller Manager, CSI Driver, etc.

## Create Infrastructure resources

As mentioned earlier, the `External` platform type uses the User-Provisioned Infrastructure, which means there is no platform-specific automation on the installer to create infrastructure resources. Although, we strongly recommend providing documentation with a programmatic method to provision the infrastructure resources using official CLI or any Infrastructure as a Code tool, like Ansible or Terraform.

The installer provides its implementation for each provider, using Terraform as a backend in the Installer-Provisioned Infrastructure (IPI) method. You can also check many UPI examples for integrated infrastructure provided in the installer repository:

- [AWS CloudFormation Templates](https://github.com/openshift/installer/tree/master/upi/aws/cloudformation) for [AWS UPI](https://docs.openshift.com/container-platform/4.13/installing/installing_aws/installing-aws-user-infra.html)
- [ARM Templates](https://github.com/openshift/installer/tree/master/upi/azure) for [Azure UPI](https://docs.openshift.com/container-platform/4.13/installing/installing_azure/installing-azure-user-infra.html)
- [Ansible Playbooks](https://github.com/openshift/installer/tree/master/upi/openstack) for [OpenStack UPI](https://docs.openshift.com/container-platform/4.13/installing/installing_openstack/installing-openstack-user-kuryr.html)

The steps below will point to the OpenShift documentation for each infrastructure component section.

### Identity

The agnostic installation used by `External` platform type does not require any identity, although the provider's components may require identity to communicate with the cloud APIs. The OpenShift prioritizes, and recommends, the least privileges and password-less authentication method, or short-lived tokens, when providing credentials to components.

### Network

There is no specific configuration for the "External" platform type when deploying network components.

See the [Networking requirements for user-provisioned infrastructure](https://docs.openshift.com/container-platform/4.13/installing/installing_platform_agnostic/installing-platform-agnostic.html#installation-network-user-infra_installing-platform-agnostic) for details of the deployment requirements.

In production environments for the cloud provider is recommended to deploy OpenShift in more than one location, some clouds named zones or fault domains. The most important is to increase the high availability of the control plane and worker nodes using the provider offerings.

### DNS

There is no specific configuration for DNS when using platform `External`, the setup must follow the same as agnostic installation and the provider configuration.

Please take a look at the following guides to the DNS setup:

- [User-provisioned DNS requirements](https://docs.openshift.com/container-platform/4.13/installing/installing_platform_agnostic/installing-platform-agnostic.html#installation-dns-user-infra_installing-platform-agnostic)
- [Validating DNS resolution for user-provisioned infrastructure](https://docs.openshift.com/container-platform/4.13/installing/installing_platform_agnostic/installing-platform-agnostic.html#installation-user-provisioned-validating-dns_installing-platform-agnostic)

### Load Balancers

The ["Load balancing requirements for user-provisioned infrastructure"](https://docs.openshift.com/container-platform/4.13/installing/installing_platform_agnostic/installing-platform-agnostic.html#installation-load-balancing-user-infra_installing-platform-agnostic).

You can use the cloud provider's Load Balancer when it meets the requirements.

Important nodes:

- the address `api-int.clusterDomain` must point to the internal Load Balancer address.
- the Load Balancers must support hairpin connections

## Preparing the installation

The `External` platform type configuration is created in this section. The steps below describe how to create the `install-config.yaml` with the required fields, and the steps used to customize the providers' manifests before generating the deployment/ignition configuration/ignition.

### Create the install-config.yaml

Follow the steps to [Manually create the installation configuration file](https://docs.openshift.com/container-platform/4.13/installing/installing_platform_agnostic/installing-platform-agnostic.html#installation-initializing-manual_installing-platform-agnostic), customizing the `platform` object, setting the type to `external`, and the `platformName` to the cloud provider's name:

> Q: Installer PR is in progress. Will the survey collect the `platformName` field?

> Q: Do we need to add ./openshift-install create install-config?

```yaml
platform:
  external:
    platformName: "providerName"
```

### Create manifests

Once the `install-config.yaml` was created, you can generate the manifests for the deployment by running the command:

```bash
./openshift-install create manifests
```

The `openshift/` and `manifests/` directory will be created.

The steps below describe how to customize the OpenShift installation.

The manifests directory will be copied to the bootstrap and applied when the core components are initialing the Control Plane.

#### Create custom manifests for CCM

In that stage, the Cloud Controller Manager (CCM) manifests must be created, as it is responsible to initialize the nodes. A set of tolerations must be set to the pod spec to ensure the controllers are initialized even if the node is not yet ready.

```yaml
      tolerations:
        - key: CriticalAddonsOnly
          operator: Exists
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: "NoSchedule"
        - key: node.cloudprovider.kubernetes.io/uninitialized
          operator: Exists
          effect: NoSchedule
        - key: node.kubernetes.io/not-ready
          operator: Exists
          effect: NoSchedule
```

The manifest below can be used as a Deployment configuration:

```yaml
# This a template that can be used as a base to run your own cluster controller manager in openshift, all you need is to:
# - replace values between {{ and }} with your own ones
# - specify a command to startup the CCM in your container
# - define and mount extra volumes if needed
# This example defines the CCM as a Deployment, but a DaemonSet is also possible as long as Pod's template is defined in the same way.
kind: Deployment
apiVersion: apps/v1
metadata:
  name: {{ cloud provider name }}-cloud-controller-manager # replace me
  namespace: {{ cloud provider name }}-cloud-controller-manager # replace me
  labels:
    k8s-app: {{ cloud provider name }}-cloud-controller-manager # replace me
    infrastructure.openshift.io/cloud-controller-manager: {{ cloud provider name }} # replace me
spec:
  replicas: 2
  selector:
    matchLabels:
      k8s-app: {{ cloud provider name }}-cloud-controller-manager # replace me
      infrastructure.openshift.io/cloud-controller-manager: {{ cloud provider name }} # replace me
  strategy:
    type: Recreate
  template:
    metadata:
      annotations:
        target.workload.openshift.io/management: '{"effect": "PreferredDuringScheduling"}'
      labels:
        k8s-app: {{ cloud provider name }}-cloud-controller-manager # replace me
        infrastructure.openshift.io/cloud-controller-manager: {{ cloud provider name }} # replace me
    spec:
      hostNetwork: true
      serviceAccount: cloud-controller-manager
      priorityClassName: system-cluster-critical
      nodeSelector:
        node-role.kubernetes.io/master: ""
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: "kubernetes.io/hostname"
            labelSelector:
              matchLabels:
                k8s-app: {{ cloud provider name }}-cloud-controller-manager # replace me
                infrastructure.openshift.io/cloud-controller-manager: {{ cloud provider name }} # replace me
      tolerations:
        - key: CriticalAddonsOnly
          operator: Exists
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: "NoSchedule"
        - key: node.cloudprovider.kubernetes.io/uninitialized
          operator: Exists
          effect: NoSchedule
        - key: node.kubernetes.io/not-ready
          operator: Exists
          effect: NoSchedule
      containers:
        - name: cloud-controller-manager
          image: {{ cloud controller image URI }} # replace me
          imagePullPolicy: "IfNotPresent"
          command:
          - /bin/bash
          - -c
          - |
            #!/bin/bash
            set -o allexport
            if [[ -f /etc/kubernetes/apiserver-url.env ]]; then
              source /etc/kubernetes/apiserver-url.env
            fi
            exec {{ cloud controller binary }} # replace me
          ports:
          - containerPort: 10258
            name: https
            protocol: TCP
          resources:
            requests:
              cpu: 200m
              memory: 50Mi
          volumeMounts:
            - mountPath: /etc/kubernetes
              name: host-etc-kube
              readOnly: true
            - name: trusted-ca
              mountPath: /etc/pki/ca-trust/extracted/pem
              readOnly: true
            # mount extra volumes if needed
      volumes:
        - name: host-etc-kube
          hostPath:
            path: /etc/kubernetes
            type: Directory
        - name: trusted-ca
          configMap:
            name: ccm-trusted-ca
            items:
              - key: ca-bundle.crt
                path: tls-ca-bundle.pem
        # add extra volumes if needed
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:cloud-controller-manager
  labels:
    kubernetes.io/cluster-service: "true"
rules:
  {{ define here the authorization rules required by your CCM }}
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: oci-cloud-controller-manager
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:cloud-controller-manager
subjects:
- kind: ServiceAccount
  name: cloud-controller-manager
  namespace: {{ cloud provider name }}-cloud-controller-manager
```


#### Create custom manifests for Kubelet

The [kubelet service](https://github.com/openshift/machine-config-operator/blob/master/templates/worker/01-worker-kubelet/_base/units/kubelet.service.yaml) is also started with the flags `--cloud-provider=external` and `--provider-id=${KUBELET_PROVIDERID}`, requiring a valid `${KUBELET_PROVIDERID}`

The value of `${KUBELET_PROVIDERID}` depends on CCM, we recommend reading the cloud provider's CCM to check the appropriate value.

In OpenShift you can set the `${KUBELET_PROVIDERID}` for each node using the [MachineConfiguration](https://docs.openshift.com/container-platform/4.13/rest_api/machine_apis/machineconfig-machineconfiguration-openshift-io-v1.html).

The example below shows how to create a `MachineConfig` to retrieve the provider's ID from the Instance/VM Metadata service, setting it to the syntax `providerName://ID`:

```yaml
# https://github.com/openshift/machine-config-operator/blob/master/templates/common/aws/files/usr-local-bin-aws-kubelet-providerid.yaml
variant: openshift
version: 4.12.0
metadata:
  name: 00-{{ machine_role }}-kubelet-providerid
  labels:
    machineconfiguration.openshift.io/role: {{ machine_role }}
storage:
  files:
  - mode: 0755
    path: "/usr/local/bin/kubelet-providerid"
    contents:
      inline: |
        #!/bin/bash
        set -e -o pipefail
        NODECONF=/etc/systemd/system/kubelet.service.d/20-providerid.conf
        if [ -e "${NODECONF}" ]; then
            echo "Not replacing existing ${NODECONF}"
            exit 0
        fi

        PROVIDERID=$(curl -sL http://169.254.169.254/metadata/id);

        cat > "${NODECONF}" <<EOF
        [Service]
        Environment="KUBELET_PROVIDERID=providerName://${PROVIDERID}"
        EOF
systemd:
  units:
  - name: kubelet-providerid.service
    enabled: true
    contents: |
      [Unit]
      Description=Fetch kubelet provider id from Metadata
      # Run afterburn service for collect info from metadata server
      # see: https://coreos.github.io/afterburn/usage/attributes/
      # Not required due to OCP 4.1 boot image does not contain afterburn service
      # see: https://issues.redhat.com/browse/OCPBUGS-7559
      Wants=afterburn.service
      After=afterburn.service
      # Wait for NetworkManager to report it's online
      After=NetworkManager-wait-online.service
      # Run before kubelet
      Before=kubelet.service
      [Service]
      # Mark afterburn environment file optional, due to it is possible that afterburn service was not executed
      EnvironmentFile=-/run/metadata/afterburn
      ExecStart=/usr/local/bin/kubelet-providerid
      Type=oneshot
      [Install]
      WantedBy=network-online.target
```

The `{{ machine_role }}` must be `master` and `worker`. Both manifests must be saved to `config-master.bu` and `config-worker.bu`, then run the following commands:

```bash
$ butane config-master.bu \
  -o openshift/99_openshift-machineconfig_00-master-kubelet-providerid.yaml

$ butane config-worker.bu \
  -o openshift/99_openshift-machineconfig_00-worker-kubelet-providerid.yaml
```

#### Create additional manifests

If the CCM or any other component you want to add in install time requires dependencies, you can also add it to the respective directories.

For example, if the CCM requires secrets or custom configuration, you can create it:

> replace the `{{ secret_value | b64encode }}` to a valid base64 string

```bash
cat <<EOF > manifests/cloud-controller-manager-01-secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: cloud-controller-manager-config
data:
  cloud-provider.yaml: {{ secret_value | b64encode }}
EOF
```

### Create ignition files

Once the manifests are placed, you can create the cluster ignition configurations:

~~~bash
./openshift-install create ignition-configs
~~~

The files with the extension `.ign` must be generated.

```text
.
├── auth
│   ├── kubeadmin-password
│   └── kubeconfig
├── bootstrap.ign
├── master.ign
├── metadata.json
└── worker.ign
```

## Create compute nodes

The step to provision compute nodes must be cloud-specific. The required steps to boot RHCOS is different for each compute role:

- bootstrap node: must use the ignition file `bootstrap.ign` to initialize
- control-plane nodes: must use the ignition file `master.ign`
- compute nodes: must use the ignition file `worker.ign`

The steps below describe each one.

Requirements:

- You created the ignition configs
- You uploaded the RHCOS image to the cloud image service
- You reviewed the ["Minimum resource requirements for cluster installation"](https://docs.openshift.com/container-platform/4.13/installing/installing_platform_agnostic/installing-platform-agnostic.html#installation-minimum-resource-requirements_installing-platform-agnostic)

### Bootstrap

The `bootstrap.ign` must be used to create the bootstrap node. Most of the cloud providers have limitations on the user-data size, so you must store the `bootstrap.ign` externally, and retrieve it in the boot process. If the cloud provides the blob service allowing to create of a signed HTTPS URL, it can be used in the ignition file as follow:

```json
{
  "ignition": {
    "config": {
      "replace": {
        "source": "https://blob.url/bootstrap.ign?signingKeys"
      }
    },
    "version": "3.1.0"
  }
}
```

Once the node is created, you can attach the bootstrap node to the public and private API Load Balancer.

### Control Plane

Three control plane nodes must be created using the ignition file `master.ign` when setting the cloud init/user-data configuration.

### Compute/workers

Three compute nodes must be created using the ignition file `master.ign` when setting the cloud init/user-data configuration.

#### Approve certificates

The Certificate Signing Requests (CSR) must be approved by the operator in the UPI deployments.

- [Certificate signing requests management](https://docs.openshift.com/container-platform/4.13/installing/installing_platform_agnostic/installing-platform-agnostic.html#csr-management_installing-platform-agnostic)

- [Approving the certificate signing requests for your machines](https://docs.openshift.com/container-platform/4.13/installing/installing_platform_agnostic/installing-platform-agnostic.html#installation-approve-csrs_installing-platform-agnostic)


## Next

- [Run conformance tests on your custom installation](./use-case-test-with-opct.md)
- [Check the use case of deploying OpenShift cluster on Oracle Cloud Infrastructure using UPI method with Platform External](./use-case-oci-upi.md)
