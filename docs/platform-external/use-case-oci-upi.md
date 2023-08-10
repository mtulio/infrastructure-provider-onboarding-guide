# Use case of installing a cluster with external platform type in Oracle Cloud Infrastructure

> NOTE: almost ready for review in the content (description/topics).
> The technical (commands) is in progress.

!!! note "Goal"
    Describe the steps to install an OCP cluster using UPI in OCI, detailing
    the customization options to setup external platform type.

This guide provides details of how to setup an OpenShift cluster using external
platform type in Oracle Cloud Infrastructure (OCI).

The installation method used in this guide for external platform type is
User-Provided Infrastructure (UPI). The steps provide low-level details to
customize the provider's components like Cloud Controller Manager (CCM).

!!! tip "Automation options"
    The goal of this document is to provide details of platform external type,
    without focus in the automation of required infrastructure. The tool used to
    provisioning the resources described in this guide is the OCI CLI.

    Alternatively the automation can be done using official
    [Ansible](https://docs.oracle.com/en-us/iaas/tools/oci-ansible-collection/4.25.0/index.html)
    or [Terraform](https://registry.terraform.io/providers/oracle/oci/latest/docs)
    modules to achieve the same goal.

!!! danger "Unsupported Document"
    This guide is created only for Red Hat partners or providers willing to integrate
    its component in OpenShift, and should not be used as an official or supported
    OpenShift installation method.

    Please look at the product documentation to get the supported path.

<!-- As described in the [installing section](./installing.md), when setting the Platform type to `External`, the OpenShift components (Kube Controller Manager, Cluster Cloud Controller Manager, Kubelet) expects to an external Cloud Controller Manager be installed to initialize the nodes.

... -->

Table of Contents

- [Prerequisites]()
    - [Clients]()
    - [Setup the Provider Account]()
    - [Upload the RHCOS image]()
- [Section 1. Create Infrastructure resources]()
    - [Identity]()
    - [Network]()
    - [DNS]()
    - [Load Balancer]()
- [Section 2. Preparing the installation]()
    - [Create install-config.yaml]()
    - [Create manifests]()
        - [Create manifests for CCM]()
        - [Create custom manifests for Kubelet]()
    - [Create ignition files]()
- [Section 3. Create the cluster]()
    - [Cluster nodes]()
        - [Bootstrap]()
        - [Control Plane]()
        - [Compute]()
    - [Review the installation]()

## Prerequisites

### Clients

#### OpenShift clients

Download the OpenShift CLI and installer:

- [Navigate to the release controller](https://openshift-release.apps.ci.l2s4.p1.openshiftapps.com/#4-dev-preview)

- Choose the release image name and extract the tools (clients):

!!! tip "Credentials"
    The Red Hat developer credential is required to pull from OpenShift CI
    repository `registry.ci.openshift.org`.

    The [Red Hat Cloud credential (Pull secret)](https://console.redhat.com/openshift/install/metal/agent-based)
    is required to pull from the repository `quay.io/openshift-release-dev/ocp-release`.

    Alternatively you can provide the option `-a /path/to/pull-secret.json`.

!!! warning "Available Releases"
    The Platform External is available in releases 4.14+ created after dev
    preview release 4.14.0-ec.4.

```sh
export OCP_RELEASE=quay.io/openshift-release-dev/ocp-release:4.14.0-ec.4-x86_64
export OPENSHIFT_INSTALL_RELEASE_IMAGE_OVERRIDE=$OCP_RELEASE
oc adm release extract -a $PULL_SECRET_FILE --tools $OCP_RELEASE
```

Extract the tarbal files:

```sh
tar xvfz openshift-client-*.tar.gz
tar xvfz openshift-install-*.tar.gz
```

The clients `openshift-install` and `oc` must be in the current directory.

#### OCI Command Line Interface

The OCI CLI is used in this guide to create infrastructure resources in the OCI.

- [Install the CLI](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm#InstallingCLI__linux_and_unix):

```sh
python3.9 -m venv ./venv-oci && source ./venv-oci/bin/activate
pip install oci-cli
```

- [Setup the user](https://docs.oracle.com/en-us/iaas/tools/oci-ansible-collection/4.25.0/guides/authentication.html#api-key-authentication) (Using [Console](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/apisigningkey.htm#two))


#### Utilities

- [jq](https://jqlang.github.io/jq/download/): used to filter the results returned by CLI

- [yq](https://github.com/mikefarah/yq/releases/tag/v4.34.1): used to patch the `yaml` manifests.

~~~bash
wget -O yq "https://github.com/mikefarah/yq/releases/download/v4.34.1/yq_linux_amd64"
chmod u+x yq
~~~

- [butane](https://github.com/coreos/butane): used to create MachineConfig files.

~~~bash
wget -O butane "https://github.com/coreos/butane/releases/download/v0.18.0/butane-x86_64-unknown-linux-gnu"
chmod u+x butane
~~~

### Setup the Provider Account

Oracle provides Compartments to aggregate resources. The compartments also
can be used to apply policies of permissions for those resources.

Compartments structure:

```
[parent]
 '- openshift
   '- controlplane
   '- compute
```

!!! tip "Compartments organization"
    The Compartments of the Control Plane and Compute nodes are nested to set
    fine-granted permissions (Policies) when resources running in Control
    Plane to access the cloud APIs.
    This is not required, but it is used as reference in this guide.

    The `parent` compartment can be used in any level in your tenant,
    including `root` level.

Create the Compartments:

- Set the parent compartment id variable:

```bash
export PARENT_COMPARTMENT_ID="<ocid1.compartment.oc1...>"
```

- Create `openshift` compartment:

```bash
oci iam compartment create \
    --compartment-id "$PARENT_COMPARTMENT_ID" \
    --description "openshift compartment" \
    --name "openshift"

COMPARTMENT_ID_OPENSHIFT=$(oci iam compartment list --name openshift \
    --compartment-id "$PARENT_COMPARTMENT_ID" | jq -r '.data[0].id')
```

!!! tip "Helper"
    OCI CLI documentation for [`oci iam compartment create`](https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.29.1/oci_cli_docs/cmdref/iam/compartment/create.html)

    OCI Console path: `Menu > Identity & Security > Compartments`

- Create `openshift/controlplane` compartment and export the variable `COMPARTMENT_ID_OPENSHIFT_CPL`:

```bash
# TODO/tmp: Move back to nested compartment after troubleshooting install issues (CCM access to APIs)
# oci iam compartment create --name "controlplane" \
#     --compartment-id "$COMPARTMENT_ID_OPENSHIFT" \
#     --description "openshift compartment for control plane nodes"

# COMPARTMENT_ID_OPENSHIFT_CPL=$(oci iam compartment list --name controlplane \
#    --compartment-id "$COMPARTMENT_ID_OPENSHIFT" | jq -r '.data[0].id')

COMPARTMENT_ID_OPENSHIFT_CPL=$COMPARTMENT_ID_OPENSHIFT
```

- Create `openshift/compute` compartment and export the variable `COMPARTMENT_ID_OPENSHIFT_CMP`:

```bash
# oci iam compartment create --name "compute" \
#     --compartment-id "$COMPARTMENT_ID_OPENSHIFT" \
#     --description "openshift compartment for control plane nodes"

# COMPARTMENT_ID_OPENSHIFT_CMP=$(oci iam compartment list --name compute \
#     --compartment-id "$COMPARTMENT_ID_OPENSHIFT" | jq -r '.data[0].id')

COMPARTMENT_ID_OPENSHIFT_CMP=$COMPARTMENT_ID_OPENSHIFT
```

### Upload the RHCOS image

The image used in this guide is QCOW2. The `openshift-install` command
provides the option `coreos print-stream-json` to show all the available
artifacts. The steps below describes how to download the iamge, upload to
a OCI bucket, then create a custom image.

- Get the image name to be used in later steps:

```bash
export IMAGE_NAME=$(basename $(./openshift-install coreos print-stream-json | jq -r '.architectures["x86_64"].artifacts["openstack"].formats["qcow2.gz"].disk.location'))
```

- Download the `QCOW2` image:

~~~bash
wget $(./openshift-install coreos print-stream-json | jq -r '.architectures["x86_64"].artifacts["openstack"].formats["qcow2.gz"].disk.location')
~~~

- Create the bucket `openshift-infra`:

```bash
oci os bucket create --name openshift-infra --compartment-id $COMPARTMENT_ID_OPENSHIFT
```

!!! tip "Helper"
    OCI CLI documentation for [`oci os bucket create`](https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.29.1/oci_cli_docs/cmdref/os/bucket/create.html)

    OCI Console path: `Menu > Storage > Buckets > (Choose the Compartment `openshift`) > Create Bucket`

- Upload the image to OCI Bucket:

```bash
oci os object put -bn openshift-infra --name images/${IMAGE_NAME} --file ${IMAGE_NAME}
```

!!! tip "Helper"
    OCI CLI documentation for [`oci os object put`](https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.29.1/oci_cli_docs/cmdref/os/object/put.html)

    OCI Console path: `Menu > Storage > Buckets > (Choose the Compartment `openshift`) > (Choose the Bucket `openshift-infra`) > Objects > Upload`

- Import to the Instance Image service:

```bash
STORAGE_NAMESPACE=$(oci os ns get | jq -r .data)
oci compute image import from-object -bn openshift-infra --name images/${IMAGE_NAME} \
    --compartment-id $COMPARTMENT_ID_OPENSHIFT -ns $STORAGE_NAMESPACE \
    --display-name ${IMAGE_NAME} --launch-mode "PARAVIRTUALIZED" \
    --source-image-type "QCOW2"
```

!!! tip "Helper"
    OCI CLI documentation for [`oci compute image import`](https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.29.1/oci_cli_docs/cmdref/compute/image/import/from-object.html)

    OCI CLI documentation for [`oci os ns get`](https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.29.1/oci_cli_docs/cmdref/os/ns/get.html)

    OCI Console path: `Menu > Storage > Buckets > (Choose the Compartment `openshift`) > (Choose the Bucket `openshift-infra`) > Objects > Upload`

## Section 1. Create Infrastructure resources

### Identity

Set the policies to allow access from Cloud Controller Manager workloadss,
running on control planes nodes, to the OCI API of resources running
in the compartment `openshift`.

- Create Dynamic Group name `openshift-controlplanes` with the following rule:

```bash
$ echo "Any {instance.compartment.id = '$COMPARTMENT_ID_OPENSHIFT_CPL'}"
```

!!! tip "Step to create Dynamic Group using OCI Console"
    OCI Console path: `Menu > Identity & Security > Domains > (Choose the compartment [root]) > Select the Identity Domain, or select "Default" > Dynamic Groups > Create dynamic group`

    - Name: `openshift-controlplanes`
    - Description: `Group of OpenShift Control Plane instances`
    - Rule 1: ** < Value of `echo` with compartment id > **

- Create policies allowing the Dynamic Group `openshift-controlplanes`
  access resources in the Compartment `openshift`:

```bash
oci iam policy create --name openshift-oci-cloud-controller-manager \
    --compartment-id $COMPARTMENT_ID_OPENSHIFT \
    --description "Policies to run OCI CCM in OpenShift" \
    --statements '[
"Allow dynamic-group openshift-controlplanes to manage volume-family in compartment openshift",
"Allow dynamic-group openshift-controlplanes to manage instance-family in compartment openshift",
"Allow dynamic-group openshift-controlplanes to manage security-lists in compartment openshift",
"Allow dynamic-group openshift-controlplanes to use virtual-network-family in compartment openshift",
"Allow dynamic-group openshift-controlplanes to manage load-balancers in compartment openshift"]'
```

!!! tip "Helper"
    OCI CLI documentation for [`oci iam policy create`](https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.29.1/oci_cli_docs/cmdref/iam/policy/create.html)

    OCI Console path: `Menu > Identity & Security > Policies >
    (Select the Compartment 'openshift') > Create Policy > Name=openshift-oci-cloud-controller-manager`

### Network

>> WIP/TODO/Question: How deepen we'll in partner provided infrastructure?
>> This is very specific for the provider, but (IMO) to provide a full example,
>> we'll need to detail it with provider-specific commands (CLI, terraform, ansible, etc)

<!--
> Temporary steps to create OCI network infra (VCN*, DNS and Load Balancers) to test the platform External feature. Note: this automation is not a goal for this document, I'll keep the step commented until WIP is able to test the doc, without focusing on the infra sections. I will check how to improve those sections later, maybe only providing references to the product docs - but considering those steps require effort for the user, we could point to some automation (maybe our CI tests?).

```
# -1) removing existing project/cluster setup: rm -rf ~/.ansible/okd-installer/clusters/$CLUSTER_NAME/
# 0) Install ansible dependencies: https://ansible-collection-okd-installer-csfcfiew8-mtulio.vercel.app/guides/OCI/oci-prerequisites/
# 1) Create the $VARS_FILE :
CLUSTER_NAME=oci-ext00
VARS_FILE=./vars-oci-ha_${CLUSTER_NAME}.yaml
BASE_DOMAIN="splat-oci.devcluster.openshift.com"
SSH_PUB_KEY_FILE="$HOME/.ssh/openshift-dev.pub"
OCI_CONFIG_CLUSTER_REGION=us-sanjose-1
OCI_COMPARTMENT_ID_DNS="${COMPARTMENT_ID_SPLAT}"
COMPARTMENT_ID_OPENSHIFT="${COMPARTMENT_ID_OPENSHIFT}"

# https://ansible-collection-okd-installer-csfcfiew8-mtulio.vercel.app/guides/OCI/oci-install-ccm/
cat <<EOF > ${VARS_FILE}
provider: oci
cluster_name: ${CLUSTER_NAME}
config_cluster_region: ${OCI_CONFIG_CLUSTER_REGION}
config_featureset: TechPreviewNoUpgrade
oci_compartment_id: ${COMPARTMENT_ID_OPENSHIFT}
oci_compartment_id_dns: ${OCI_COMPARTMENT_ID_DNS}
oci_compartment_id_image: "${COMPARTMENT_ID_OPENSHIFT}"
cluster_profile: ha
config_base_domain: $BASE_DOMAIN
config_ssh_key: "$(cat ${SSH_PUB_KEY_FILE})"
config_pull_secret_file: "${PULL_SECRET_FILE}"
release_image: quay.io/openshift-release-dev/ocp-release
release_version: 4.14.0-ec.4
EOF

# 3) Create install-config.yaml manually (Section 2/#Create install-config.yaml)
# 4) Load project and create the infrastructure stacks (network, DNS and load balancers) running:
ansible-playbook mtulio.okd_installer.install_clients -e @$VARS_FILE
ansible-playbook mtulio.okd_installer.config -e mode=create-manifests -e @$VARS_FILE
ansible-playbook mtulio.okd_installer.stack_network -e @$VARS_FILE
ansible-playbook mtulio.okd_installer.stack_dns -e @$VARS_FILE
ansible-playbook mtulio.okd_installer.stack_loadbalancer -e @$VARS_FILE
```
 -->

The provider network must be created using the [Networking requirements for user-provisioned infrastructure](https://docs.openshift.com/container-platform/4.13/installing/installing_platform_agnostic/installing-platform-agnostic.html#installation-network-user-infra_installing-platform-agnostic).

!!! tip "Info"
    The resource name provided in this guide is not standard, but follows
    a similar naming convention created by installer in supported cloud
    providers. The names will also be used in future sections to discover resources.

Create the VCN and dependencies with the following configuration:

| Resource | Name | Attributes | Note |
| -- | -- | -- | -- |
| VCN | `${CLUSTER_NAME}-vcn` | CIDR 10.0.0.0/16 | |
| Subnet | `${CLUSTER_NAME}-net-public` | 10.0.0.0/20 | Regional,Resolve DNS |
| Subnet | `${CLUSTER_NAME}-net-private` | 10.0.128.0/20 | Regional,Resolve DNS  |
| NSG | `${CLUSTER_NAME}-nsg-nlb` | | Used by Load Balancer |
| NSG | `${CLUSTER_NAME}-nsg-controlplane` | | Used by Control Plane nodes |
| NSG | `${CLUSTER_NAME}-nsg-compute` | | Used by Compute nodes |
| ... | ... | ... | ... |
| Internet GW | | -- | |
| Nat GW | | -- | |
| ... | ... | ... | ... |

### DNS

>> WIP

!!! tip "Helper"
    It's not required to have a public accessible API and DNS domain, but it
    will allow to access the cluster without needing to keep a bastion host.

DNS records for an API accessed from the internet:

| Domain | Record | Value |
| -- | -- | -- |
| `${CLUSTER_NAME}`.`${BASE_DOMAIN}` | api | Public IP Address or DNS for the Load Balancer |
| `${CLUSTER_NAME}`.`${BASE_DOMAIN}` | api-int | Private IP Address or DNS for the Load Balancer |
| `${CLUSTER_NAME}`.`${BASE_DOMAIN}` | *.apps | Public IP Address or DNS for the Load Balancer |

### Load Balancer

>> WIP

- Create the Network Load Balancer with name `${CLUSTER_NAME}-nlb`:

Backend Sets (BSet):

| BSet Name | Port | Health Check (Proto/Path/Interval/Timeout) |
| -- | -- | -- |
| `${CLUSTER_NAME}-api` | TCP/6443 | HTTPS`/readyz`/10/3 |
| `${CLUSTER_NAME}-mcs` | TCP/22623 | HTTPS`/healthz`/10/3 |
| `${CLUSTER_NAME}-http` | TCP/80 | TCP/80/10/3 |
| `${CLUSTER_NAME}-https` | TCP/443 | TCP/443/10/3 |

Listeners:

| Name | Port | BSet Name |
| -- | -- | -- |
| `${CLUSTER_NAME}-api` | TCP/6443 | `${CLUSTER_NAME}-api` |
| `${CLUSTER_NAME}-mcs` | TCP/22623 | `${CLUSTER_NAME}-mcs` |
| `${CLUSTER_NAME}-http` | TCP/80 | `${CLUSTER_NAME}-http` |
| `${CLUSTER_NAME}-https` | TCP/443 | `${CLUSTER_NAME}-https` |

## Section 2. Preparing the installation

This section describes how to setup the OpenShift customizing the manifests
used in the installation.

### Create the installer configuration

Modify and export the variables used to build the `install-config.yaml` and
the later steps:

```bash
export CLUSTER_NAME=oci-ext03
export BASE_DOMAIN=splat-oci.devcluster.openshift.com

#export INSTALL_DIR=./install-dir
#> tmp path while using automation to create infra stacks (network, dns and LB)
export INSTALL_DIR=${HOME}/.ansible/okd-installer/clusters/${CLUSTER_NAME}/

export SSH_PUB_KEY_FILE="${HOME}/.ssh/bundle.pub"
export PULL_SECRET_FILE="${HOME}/.openshift/pull-secret-latest.json"

# Create the install directory
mkdir -p $INSTALL_DIR
```

#### Create install-config.yaml

Create the `install-config.yaml` setting the Platform type to `external`:

```bash
cat <<EOF > ${INSTALL_DIR}/install-config.yaml
apiVersion: v1
baseDomain: ${BASE_DOMAIN}
metadata:
  name: "${CLUSTER_NAME}"
platform:
  external:
    platformName: oci
publish: External
pullSecret: >
  $(cat ${PULL_SECRET_FILE})
sshKey: |
  $(cat ${SSH_PUB_KEY_FILE})
EOF
```

### Create manifests

```bash
./openshift-install create manifests --dir $INSTALL_DIR
```

#### Create manifests for OCI Cloud Controller Manager

The steps in this section describes how to customize the OpenShift installation
providing the Cloud Controller Manager manifests to be added in the bootstrap process.

!!! warning "Info"
    This guide is based in the OCI CCM v1.25.0. You must read the
    [project documentation](https://github.com/oracle/oci-cloud-controller-manager)
    for more information.

- Create the namespace manifest:

!!! danger "Important"
    Is not recommended to create resources in namespaces prefixed with `kube-*`
    and `openshift-*`. The custom namespace manifest must be created, then
    deployment manifests must be adapted to used the custom namespace.

    See [the documentation](https://docs.openshift.com/container-platform/4.13/applications/projects/working-with-projects.html) for more information.

```bash
export OCI_CCM_NAMESPACE=oci-cloud-controller-manager

cat <<EOF > ${INSTALL_DIR}/manifests/oci-00-ccm-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: $OCI_CCM_NAMESPACE
  annotations:
    workload.openshift.io/allowed: management
    include.release.openshift.io/self-managed-high-availability: "true"
  labels:
    "pod-security.kubernetes.io/enforce": "privileged"
    "pod-security.kubernetes.io/audit": "privileged"
    "pod-security.kubernetes.io/warn": "privileged"
    "security.openshift.io/scc.podSecurityLabelSync": "false"
    "openshift.io/run-level": "0"
    "pod-security.kubernetes.io/enforce-version": "v1.24"
EOF
```

!!! danger "TODO"
    - The Pod Admission Security must be reviewed aiming to use other than `privileged`
    - Set [critical pod annotation](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/#rescheduler-guaranteed-scheduling-of-critical-add-ons) in this namespace.

- Create the OCI cloud-config secret:

```bash
export OCI_CONFIG_CLUSTER_REGION=us-sanjose-1
export OCI_CCM_COMPARTMENT_ID=$COMPARTMENT_ID_OPENSHIFT

## Discover VCN ID
export OCI_VCN_ID=$(oci network vcn list --compartment-id $OCI_CCM_COMPARTMENT_ID \
  | jq -r .data[0].id)

## Discover public Subnet ID
export OCI_LB_SUBNET_ID=$(oci network subnet list --compartment-id $OCI_CCM_COMPARTMENT_ID \
  | jq -r '.data[] | select(.["display-name"] | endswith("public")).id')

# Creating the file and duplicating it in the secret to prevent CCM Error in the logs.
cat <<EOF > ./cloud-provider.yaml
auth:
  region: $OCI_CONFIG_CLUSTER_REGION
useInstancePrincipals: true

compartment: $OCI_CCM_COMPARTMENT_ID
vcn: $OCI_VCN_ID

loadBalancer:
  securityListManagementMode: None
  subnet1: $OCI_LB_SUBNET_ID

# Optional rate limit controls for accessing OCI API
rateLimiter:
  rateLimitQPSRead: 20.0
  rateLimitBucketRead: 5
  rateLimitQPSWrite: 20.0
  rateLimitBucketWrite: 5
EOF

# Create the secret manifest
cat <<EOF1 > ${INSTALL_DIR}/manifests/oci-01-ccm-00-secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: oci-cloud-controller-manager
  namespace: $OCI_CCM_NAMESPACE
data:
  cloud-provider.yaml: $(base64 -w0 < ./cloud-provider.yaml)
EOF1
```

<!-- !!! danger "Bug"
    The CCM is raising the error below in the startup/logs. The secret data `config.yaml` has been duplicated to prevent this error.

    ```
    ERROR   oci/ccm.go:122  Metrics collection could not be enabled {"component": "cloud-controller-manager", "error": "failed to load configuration file at path /etc/oci/config.yaml: open /etc/oci/config.yaml: no such file or directory"
    ```


!!! warning "Question"
    - Is it possible to use NSG instead of SecList in Load Balancer?
-->

- Download manifests from [OCI CCM's Github](https://github.com/oracle/oci-cloud-controller-manager)
  and save it in the directory `${INSTALL_DIR}/manifests`:

```bash
export CCM_RELEASE=v1.26.0

wget https://github.com/oracle/oci-cloud-controller-manager/releases/download/${CCM_RELEASE}/oci-cloud-controller-manager-rbac.yaml -O oci-cloud-controller-manager-rbac.yaml

wget  https://github.com/oracle/oci-cloud-controller-manager/releases/download/${CCM_RELEASE}/oci-cloud-controller-manager.yaml -O oci-cloud-controller-manager.yaml
```

- Patch the RBAC file setting the correct namespace in the `ServiceAccount`:

```bash
./yq ". | select(.kind==\"ServiceAccount\").metadata.namespace=\"$OCI_CCM_NAMESPACE\"" oci-cloud-controller-manager-rbac.yaml > ./oci-cloud-controller-manager-rbac_patched.yaml
```

- Patch the RBAC file setting the correct namespace in the `ServiceAccount`:

```bash
cat << EOF > ./oci-ccm-rbac_patch_crb-subject.yaml
- kind: ServiceAccount
  name: cloud-controller-manager
  namespace: $OCI_CCM_NAMESPACE
EOF

./yq eval-all -i ". | select(.kind==\"ClusterRoleBinding\").subjects *= load(\"oci-ccm-rbac_patch_crb-subject.yaml\")" ./oci-cloud-controller-manager-rbac_patched.yaml
```

- Split the RBAC manifest file:

```bash
./yq -s '"./oci-01-ccm-01-rbac_" + $index' ./oci-cloud-controller-manager-rbac_patched.yaml &&\
mv -v ./oci-01-ccm-01-rbac_*.yml ${INSTALL_DIR}/manifests/
```

- Patch the CCM DaemonSet manifest setting the namespace, append the tolerations,
  mount CA, and add env vars for the kube API URL used in OpenShift:

```bash
# Create the pod template patch
cat <<EOF > ./oci-cloud-controller-manager-ds_patch1.yaml
metadata:
  namespace: $OCI_CCM_NAMESPACE
spec:
  template:
    spec:
      tolerations:
        - key: node.kubernetes.io/not-ready
          operator: Exists
          effect: NoSchedule
EOF

# Create the containers' patch (TODO merge both patches)
cat <<EOF > ./oci-cloud-controller-manager-ds_patch2.yaml
spec:
  template:
    spec:
      containers:
        - env:
          - name: KUBERNETES_PORT
            value: "tcp://api-int.$CLUSTER_NAME.$BASE_DOMAIN:6443"
          - name: KUBERNETES_PORT_443_TCP
            value: "tcp://api-int.$CLUSTER_NAME.$BASE_DOMAIN:6443"
          - name: KUBERNETES_PORT_443_TCP_ADDR
            value: "api-int.$CLUSTER_NAME.$BASE_DOMAIN"
          - name: KUBERNETES_PORT_443_TCP_PORT
            value: "6443"
          - name: KUBERNETES_PORT_443_TCP_PROTO
            value: "tcp"
          - name: KUBERNETES_SERVICE_HOST
            value: "api-int.$CLUSTER_NAME.$BASE_DOMAIN"
          - name: KUBERNETES_SERVICE_PORT
            value: "6443"
          - name: KUBERNETES_SERVICE_PORT_HTTPS
            value: "6443"
EOF

# Merge required objects for pod template
./yq eval-all '. as $item ireduce ({}; . *+ $item)' oci-cloud-controller-manager.yaml oci-cloud-controller-manager-ds_patch1.yaml > oci-cloud-controller-manager-ds_patched1.yaml

# Merge required objects for containers
./yq eval-all '.spec.template.spec.containers[] as $item ireduce ({}; . *+ $item)' oci-cloud-controller-manager-ds_patched1.yaml ./oci-cloud-controller-manager-ds_patch2.yaml > ./oci-cloud-controller-manager-ds_patched2.yaml

# merge patched files
./yq eval-all '.spec.template.spec.containers[] *= load("./oci-cloud-controller-manager-ds_patched2.yaml")' oci-cloud-controller-manager-ds_patched1.yaml > ${INSTALL_DIR}/manifests/oci-01-ccm-02-daemonset.yaml
```

The following CCM manifests files must be created in the installation `manifests/` directory:

```bash
$ tree $INSTALL_DIR/manifests/
[...]
├── oci-00-namespace.yaml
├── oci-01-ccm-00-secret.yaml
├── oci-01-ccm-01-rbac_0.yml
├── oci-01-ccm-01-rbac_1.yml
├── oci-01-ccm-01-rbac_2.yml
├── oci-01-ccm-02-daemonset.yaml
[...]
```

#### Create custom manifests for Kubelet

The `providerID`, a Kubelet configuration, must be set, sets the unique ID of the instance that an external provider (i.e. cloudprovider) can use to identify a specific node.

This section describes the steps to create MachineConfig for kubeket to set
the cloud provider ID of the node that an external provider used to identify a specific node.

The Provider ID must be set dynamically for each node. The steps below describes
how to create a MachineConfig manifest to setup a systemd unit to create a kubelet
configuration discoverying the Provider ID in OCI by querying the
[Instance Metadata Service (IMDS)](https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/gettingmetadata.htm).

- Create the butane files for master and worker configurations:

```bash
function create_machineconfig_kubelet() {
    local node_role=$1
    cat << EOF > ./mc-kubelet-$node_role.bu
# Butane file to setup kubelet for node role $node_role
variant: openshift
version: 4.13.0
metadata:
  name: 00-$node_role-kubelet-providerid
  labels:
    machineconfiguration.openshift.io/role: $node_role
storage:
  files:
  - mode: 0755
    path: "/usr/local/bin/kubelet-providerid"
    contents:
      inline: |
        #!/bin/bash
        set -e -o pipefail
        NODECONF=/etc/systemd/system/kubelet.service.d/20-providerid.conf
        if [ -e "\${NODECONF}" ]; then
            echo "Not replacing existing \${NODECONF}"
            exit 0
        fi

        PROVIDERID=\$(curl -H "Authorization: Bearer Oracle" -sL http://169.254.169.254/opc/v2/instance/ | jq -r .id);

        cat > "\${NODECONF}" <<EOF
        [Service]
        Environment="KUBELET_PROVIDERID=\${PROVIDERID}"
        EOF
systemd:
  units:
  - name: kubelet-providerid.service
    enabled: true
    contents: |
      [Unit]
      Description=Fetch kubelet provider id from Metadata
      Wants=afterburn.service
      After=afterburn.service
      After=NetworkManager-wait-online.service
      Before=kubelet.service
      [Service]
      # Mark afterburn environment file optional, due to it is possible that afterburn service was not executed
      EnvironmentFile=-/run/metadata/afterburn
      ExecStart=/usr/local/bin/kubelet-providerid
      Type=oneshot
      [Install]
      WantedBy=network-online.target
EOF
}

create_machineconfig_kubelet "master"
create_machineconfig_kubelet "worker"
```

- Process the butane files to `MachineConfig` objects:

```bash
function process_butane() {
    local src_file=$1; shift
    local dest_file=$1

    ./butane $src_file -o $dest_file
}

process_butane "./mc-kubelet-master.bu" "${INSTALL_DIR}/openshift/99_openshift-machineconfig_00-master-kubelet-providerid.yaml"
process_butane "./mc-kubelet-worker.bu" "${INSTALL_DIR}/openshift/99_openshift-machineconfig_00-worker-kubelet-providerid.yaml"
```

The MachineConfig files must exists:

```bash
ls ${INSTALL_DIR}/openshift/99_openshift-machineconfig_00-*-kubelet-providerid.yaml
```

### Create ignition files

Once the manifests are placed, you can create the cluster ignition configurations:

~~~bash
./openshift-install create ignition-configs --dir $INSTALL_DIR
~~~

The ignition files must be generated in the install directory (files with externsion `*.ign`):

```text
$ tree $INSTALL_DIR
/path/to/install-dir
├── auth
│   ├── kubeadmin-password
│   └── kubeconfig
├── bootstrap.ign
├── master.ign
├── metadata.json
└── worker.ign
```

- Upload the `bootstrap.ign` to the infrastructure bucket

```bash
oci os object put -bn openshift-infra --name bootstrap-${CLUSTER_NAME}.ign \
    --file $INSTALL_DIR/bootstrap.ign
```

!!! tip "Helper"
    OCI CLI documentation for [`oci os object put`](https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.29.1/oci_cli_docs/cmdref/os/object/put.html)


- Generate the signed URL for the bootstrap object to be used in the user-data:

```bash
EXPIRES_TIME=$(date -d '+1 hour' --rfc-3339=seconds)
USER_DATA_URL=$(oci os preauth-request create --name bootstrap-${CLUSTER_NAME} \
    -bn openshift-infra -on bootstrap-${CLUSTER_NAME}.ign \
    --access-type ObjectRead  --time-expires "$EXPIRES_TIME" \
    | jq -r '.data["full-path"]')
```

!!! tip "Helper"
    OCI CLI documentation for [`oci os preauth-request create`](https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.29.0/oci_cli_docs/cmdref/os/preauth-request/create.html)

The ignition URL for the Bootstrap node must be available in the `$USER_DATA_URL`.

!!! warning "Attention"
    Bucket Object URL will expires in one hour, if you are planning to create
    the bootstrap later, please adjust it.

    The install certificates expires in 24 hours after the ignition files have been
    created, consider regenerating it if the ignitions are older than that.


## Section 3. Create the cluster

### Cluster nodes

| Compartment | Node Name  | User Data |
| -- | -- | -- |
| [parent]/openshift/controlplane | bootstrap | `${PWD}/user-data-bootstrap.json` |
| [parent]/openshift/controlplane | control planes nodes (pool) | `${INSTALL_DIR}/master.json` |
| [parent]/openshift/compute | compute nodes (pool) | `${INSTALL_DIR}/worker.json` |

Export the required variables to be used on instance creation:

- `IMAGE_ID`: Custom RHCOS image previously uploaded.
- `SUBNET_ID_PUBLIC`: Public regional subnet used in bootstrap.
- `SUBNET_ID_PRIVATE`: Private regional subnet used to create control plane and compute nodes.
- `NSG_ID_CPL`: Network Security Group ID used in Control Planes

```bash
# Gather the Custom Compute image for RHCOS
export IMAGE_ID=$(oci compute image list --compartment-id $COMPARTMENT_ID_OPENSHIFT \
  --display-name $IMAGE_NAME | jq -r '.data[0].id')

# Gather the subnet IDs
export SUBNET_ID_PUBLIC=$(oci network subnet list --compartment-id $OCI_CCM_COMPARTMENT_ID \
  | jq -r '.data[] | select(.["display-name"] | endswith("public")).id')
export SUBNET_ID_PRIVATE=$(oci network subnet list --compartment-id $OCI_CCM_COMPARTMENT_ID \
  | jq -r '.data[] | select(.["display-name"] | endswith("private")).id')

# Gather the Network Security group for control plane
export NSG_ID_CPL=$(oci network nsg list -c $COMPARTMENT_ID_OPENSHIFT \
  | jq -r '.data[] | select(.["display-name"] | endswith("controlplane")).id')
```

!!! tip "Helper"
    OCI CLI documentation for [`oci compute image list`](https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.29.1/oci_cli_docs/cmdref/compute/image/list.html)

    OCI CLI documentation for [`oci network subnet list`](https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.29.1/oci_cli_docs/cmdref/network/subnet/list.html)

    OCI CLI documentation for [`oci network nsg list`](#)

#### Bootstrap

The Bootstrap node is responsible to create the temporary control plane, and serve
the ignition files to other nodes through Machine Config Server.

Considering the size limitation of user-data, the cloud-init points to the
temporary Bucket Object URL where the full ignition file is stored.

Once the bootstrap instance is created, it must be attached to the Load Balancer in the
Backends for Kubernetes API Server and Machine Config Server.

- Generate the user-data:

```bash
cat <<EOF > user-data-bootstrap.json
{
  "ignition": {
    "config": {
      "replace": {
        "source": "${USER_DATA_URL}"
      }
    },
    "version": "3.1.0"
  }
}
EOF
```

Create the Instance for bootstrap with the following configuration:

```bash
AVAILABILITY_DOMAIN="gzqB:US-SANJOSE-1-AD-1"
INSTANCE_SHAPE="VM.Standard.E4.Flex"

# Launch
oci compute instance launch \
    --hostname-label "bootstrap" \
    --display-name "bootstrap" \
    --availability-domain "$AVAILABILITY_DOMAIN" \
    --fault-domain "FAULT-DOMAIN-1" \
    --compartment-id $COMPARTMENT_ID_OPENSHIFT_CPL \
    --subnet-id $SUBNET_ID_PUBLIC \
    --nsg-ids "[\"$NSG_ID_CPL\"]" \
    --shape "$INSTANCE_SHAPE" \
    --shape-config "{\"memoryInGBs\":16.0,\"ocpus\":8.0}" \
    --source-details "{\"bootVolumeSizeInGBs\":120,\"bootVolumeVpusPerGB\":60,\"imageId\":\"${IMAGE_ID}\",\"sourceType\":\"image\"}" \
    --agent-config '{"areAllPluginsDisabled": true}' \
    --assign-public-ip True \
    --user-data-file "./user-data-bootstrap.json"
```

!!! tip "Helper"
    OCI CLI documentation for [`oci compute instance launch`](https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.29.1/oci_cli_docs/cmdref/compute/instance/launch.html)

    OCI CLI documentation for [`oci compute shape list`](https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.29.1/oci_cli_docs/cmdref/compute/shape/list.html)

!!! tip "Warning"
    You can SSH to the node and follow the bootstrap process:
    `journalctl -b -f -u release-image.service -u bootkube.service`


- Discover the Load Balancer and Bootstrap IDs:

```bash
# Lookup the Network Load Balancer ending with name 'nlb'
NLB_ID=$(oci nlb network-load-balancer list --compartment-id $COMPARTMENT_ID_OPENSHIFT | jq -r ".data.items[] | select(.[\"display-name\"] | endswith(\"-nlb\")).id")
BES_API_NAME=$(oci nlb backend-set list --network-load-balancer-id $NLB_ID | jq -r '.data.items[] | select(.name | endswith("api")).name')
BES_MCS_NAME=$(oci nlb backend-set list --network-load-balancer-id $NLB_ID | jq -r '.data.items[] | select(.name | endswith("mcs")).name')

INSTANCE_ID_BOOTSTRAP=$(oci compute instance list  -c $COMPARTMENT_ID_OPENSHIFT_CPL | jq -r '.data[] | select((.["display-name"]=="bootstrap") and (.["lifecycle-state"]=="RUNNING")).id')

test -z $INSTANCE_ID_BOOTSTRAP && echo "ERR: Bootstrap Instance ID not found=[$INSTANCE_ID_BOOTSTRAP]. Try again."
```

!!! tip "Helper"
    [TODO] OCI CLI documentation for:<br>
    - [`oci nlb network-load-balancer list`]()<br>
    - [`oci nlb backend-set list`]()<br>
    - [`oci compute instance list`]()<br>


- Add the bootstrap to the Load Balancer's backend set API
```bash
# oci nlb backend-set update --generate-param-json-input backends
cat <<EOF > ./nlb-bset-backends-api.json
[
  {
    "isBackup": false,
    "isDrain": false,
    "isOffline": false,
    "name": "${INSTANCE_ID_BOOTSTRAP}:6443",
    "port": 6443,
    "targetId": "${INSTANCE_ID_BOOTSTRAP}"
  }
]
EOF

# Update API Backend Set
oci nlb backend-set update --wait-for-state SUCCEEDED \
  --backend-set-name $BES_API_NAME \
  --network-load-balancer-id $NLB_ID \
  --backends file://nlb-bset-backends-api.json
```

!!! tip "Helper"
    OCI CLI documentation for [`oci nlb backend-set update`](https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.29.1/oci_cli_docs/cmdref/nlb/backend-set/update.html)

- Add the bootstrap to the Load Balancer's backend set MCS[Machine Config Server]:

```bash
cat <<EOF > ./nlb-bset-backends-mcs.json
[
  {
    "isBackup": false,
    "isDrain": false,
    "isOffline": false,
    "name": "${INSTANCE_ID_BOOTSTRAP}:22623",
    "port": 22623,
    "targetId": "${INSTANCE_ID_BOOTSTRAP}"
  }
]
EOF

# Update MCS Backend Set
oci nlb backend-set update --wait-for-state SUCCEEDED \
  --backend-set-name $BES_MCS_NAME \
  --network-load-balancer-id $NLB_ID \
  --backends file://nlb-bset-backends-mcs.json
```

#### Control Plane

Three control plane instances will be created. The instances is created using
Compute Pool, which will automatically inherit the same configuration and add
to the desired listeners: API and MCS.

- Creating the Instance Configuration required by Instance Pool:

```bash
# To generate all the options:
# oci compute-management instance-configuration create --generate-param-json-input instance-details
cat <<EOF > ./instance-config-details-controlplanes.json
{
  "instanceType": "compute",
  "launchDetails": {
    "agentConfig": {"areAllPluginsDisabled": true},
    "compartmentId": "$COMPARTMENT_ID_OPENSHIFT_CPL",
    "createVnicDetails": {
      "assignPrivateDnsRecord": true,
      "assignPublicIp": false,
      "nsgIds": ["$NSG_ID_CPL"],
      "subnetId": "$SUBNET_ID_PRIVATE"
    },
    "displayName": "${CLUSTER_NAME}-controlplane",
    "launchMode": "PARAVIRTUALIZED",
    "metadata": {"user_data": "$(base64 -w0 < $INSTALL_DIR/master.ign)"},
    "shape": "$INSTANCE_SHAPE",
    "shapeConfig": {"memoryInGBs":16.0,"ocpus":8.0},
    "sourceDetails": {"bootVolumeSizeInGBs":120,"bootVolumeVpusPerGB":60,"imageId":"${IMAGE_ID}","sourceType":"image"}
  }
}
EOF

oci compute-management instance-configuration create \
  --display-name "openshift-controlplane-config" \
  --compartment-id $COMPARTMENT_ID_OPENSHIFT_CPL \
  --instance-details file://instance-config-details-controlplanes.json
```

- Creating the Instance Pool:

```bash
INSTANCE_CONFIG_ID_CPL=$(oci compute-management instance-configuration list --compartment-id $COMPARTMENT_ID_OPENSHIFT_CPL | jq -r '.data[] | select(.["display-name"] | startswith("openshift-controlplane")).id')

#
# oci compute-management instance-pool create --generate-param-json-input load-balancers
cat <<EOF > ./instance-pool-loadbalancers-cpl.json
[
  {
    "backendSetName": "$BES_API_NAME",
    "loadBalancerId": "$NLB_ID",
    "port": 6443,
    "vnicSelection": "PrimaryVnic"
  },
  {
    "backendSetName": "$BES_MCS_NAME",
    "loadBalancerId": "$NLB_ID",
    "port": 22623,
    "vnicSelection": "PrimaryVnic"
  }
]
EOF

# oci compute-management instance-pool create --generate-param-json-input placement-configurations
cat <<EOF > ./instance-pool-placement.json
[
  {
    "availabilityDomain": "$AVAILABILITY_DOMAIN",
    "faultDomains": ["FAULT-DOMAIN-1","FAULT-DOMAIN-2","FAULT-DOMAIN-3"],
    "primarySubnetId": "$SUBNET_ID_PRIVATE",
  }
]
EOF

#
oci compute-management instance-pool create \
  --compartment-id $COMPARTMENT_ID_OPENSHIFT_CPL \
  --instance-configuration-id "$INSTANCE_CONFIG_ID_CPL" \
  --size 0 \
  --display-name "ocp-controlplane" \
  --placement-configurations "file://instance-pool-placement.json" \
  --load-balancers file://instance-pool-loadbalancers-cpl.json
```

!!! tip "Helper TODO"
    OCI CLI documentation for [`oci compute-management instance-pool create`]()

- Scale up (alternatively the `--size` can be adjusted when creating the Instance Pool):

```bash
INSTANCE_POOL_ID_CPL=$(oci compute-management instance-pool list \
    --compartment-id  $COMPARTMENT_ID_OPENSHIFT_CPL \
    | jq -r '.data[] | select(
        (.["display-name"]=="ocp-controlplane") and
        (.["lifecycle-state"]=="RUNNING")
    ).id')

oci compute-management instance-pool update --instance-pool-id $INSTANCE_POOL_ID_CPL --size 1
```
!!! tip "Helper TODO"
    OCI CLI documentation for [`oci compute-management instance-pool update `]()

#### Compute/workers

- Creating the Instance Configuration

```bash
export NSG_ID_CMP=$(oci network nsg list -c $COMPARTMENT_ID_OPENSHIFT | jq -r '.data[] | select(.["display-name"] | endswith("compute")).id')

# oci compute-management instance-configuration create --generate-param-json-input instance-details
cat <<EOF > ./instance-config-details-compute.json
{
  "instanceType": "compute",
  "launchDetails": {
    "agentConfig": {"areAllPluginsDisabled": true},
    "compartmentId": "$COMPARTMENT_ID_OPENSHIFT_CMP",
    "createVnicDetails": {
      "assignPrivateDnsRecord": true,
      "assignPublicIp": false,
      "nsgIds": ["$NSG_ID_CMP"],
      "subnetId": "$SUBNET_ID_PRIVATE"
    },
    "displayName": "${CLUSTER_NAME}-compute",
    "launchMode": "PARAVIRTUALIZED",
    "metadata": {"user_data": "$(base64 -w0 < $INSTALL_DIR/worker.ign)"},
    "shape": "$INSTANCE_SHAPE",
    "shapeConfig": {"memoryInGBs":16.0,"ocpus":8.0},
    "sourceDetails": {"bootVolumeSizeInGBs":120,"bootVolumeVpusPerGB":20,"imageId":"${IMAGE_ID}","sourceType":"image"}
  }
}
EOF

oci compute-management instance-configuration create \
  --display-name "openshift-compute-config" \
  --compartment-id $COMPARTMENT_ID_OPENSHIFT_CMP \
  --instance-details file://instance-config-details-compute.json
```

- Creating the Instance Pool

```bash
BES_HTTP_NAME=$(oci nlb backend-set list --network-load-balancer-id $NLB_ID | jq -r '.data.items[] | select(.name | endswith("http")).name')
BES_HTTPS_NAME=$(oci nlb backend-set list --network-load-balancer-id $NLB_ID | jq -r '.data.items[] | select(.name | endswith("https")).name')

INSTANCE_CONFIG_ID_CMP=$(oci compute-management instance-configuration list --compartment-id $COMPARTMENT_ID_OPENSHIFT_CMP | jq -r '.data[] | select(.["display-name"] | startswith("openshift-compute")).id')

#
# oci compute-management instance-pool create --generate-param-json-input load-balancers
cat <<EOF > ./instance-pool-loadbalancers-cmp.json
[
  {
    "backendSetName": "$BES_HTTP_NAME",
    "loadBalancerId": "$NLB_ID",
    "port": 80,
    "vnicSelection": "PrimaryVnic"
  },
  {
    "backendSetName": "$BES_HTTPS_NAME",
    "loadBalancerId": "$NLB_ID",
    "port": 443,
    "vnicSelection": "PrimaryVnic"
  }
]
EOF

#
oci compute-management instance-pool create \
  --compartment-id $COMPARTMENT_ID_OPENSHIFT_CMP \
  --instance-configuration-id "$INSTANCE_CONFIG_ID_CMP" \
  --size 0 \
  --display-name "ocp-compute" \
  --placement-configurations "[{\"availabilityDomain\":\"$AVAILABILITY_DOMAIN\",\"faultDomains\":[\"FAULT-DOMAIN-1\",\"FAULT-DOMAIN-2\",\"FAULT-DOMAIN-3\"],\"primarySubnetId\":\"$SUBNET_ID_PRIVATE\"}]" \
  --load-balancers file://instance-pool-loadbalancers-cmp.json
```

- Scale up

```bash
INSTANCE_POOL_ID_CMP=$(oci compute-management instance-pool list \
    --compartment-id  $COMPARTMENT_ID_OPENSHIFT_CMP \
    | jq -r '.data[] | select(
        (.["display-name"]=="ocp-compute") and
        (.["lifecycle-state"]=="RUNNING")
    ).id')

oci compute-management instance-pool update --instance-pool-id $INSTANCE_POOL_ID_CMP --size 2
```

### Review the installation

Export the kubeconfig

```bash
export KUBECONFIG=$INSTALL_DIR/auth/kubeconfig
```

#### OCI CCM

- Check if the nodes have been initialized

```bash
oc get nodes
```

- Check if the controllers are running for each master:

```bash
oc get all -n oci-ccm
```

#### Wait for bootstrap complete

- Wait for bootstrap complete
```bash
./openshift-install --dir $INSTALL_DIR wait-for bootstrap-complete
```

You can delete the bootstrap instance (or use it as bastion node until
the instlalation has been completed).

#### Approve certificates

```bash
oc adm certificate approve $(oc get csr  -o json |jq -r '.items[] | select(.status.certificate == null).metadata.name')
```

#### Check installation complete

```bash
watch -n5 oc get clusteroperators

./openshift-install --dir $INSTALL_DIR wait-for install-complete
```

## Next

TBD