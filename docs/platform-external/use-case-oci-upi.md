# Installing a cluster with UPI on Oracle Cloud Infrastructure with Platform External

> TODO describe how to install using UPI (referencing the official documentation of agnostic installation).

This guide provides details how to install OpenShift cluster using User-Provided Infrastructure in a non-integrated provider using the OpenShift Platform type `External` in Oracle Cloud Infrastructure.

As described in the [installing section](./installing.md), when setting the Platform type to `External`, the OpenShift components (Kube Controller Manager, Cluster Cloud Controller Manager, Kubelet) expects to an external Cloud Controller Manager be installed to initialize the nodes.

...


## Prerequisites

### Clients

#### OpenShift clients

Download the OpenShift CLI and installer:

- Navigate to the release controller: https://openshift-release.apps.ci.l2s4.p1.openshiftapps.com/#4.14.0-0.nightly

- Choose the release image name and extract the tools (clients):

> The Platform External is available in releases 4.14+ created after 2023-06-23

```bash
export RELEASE=registry.ci.openshift.org/ocp/release:4.14.0-0.nightly-2023-06-26-004511
oc adm release extract -a ~/.openshift/pull-secret-latest.json --tools $RELEASE
```

#### OCI CLI

The OCI CLI is used in this guide to create resources in the Oracle Cloud Infrastructure.

- Steps to Download:
- Steps to setup user:

> TODO / TBD

#### Utilities

- [yq](https://github.com/mikefarah/yq/releases/tag/v4.34.1)

`yq` is used to patch the `yaml` manifests.

- [Butane](https://github.com/coreos/butane)

`butane` is used to create MachineConfig files.

wget -O butane "https://github.com/coreos/butane/releases/download/v0.17.0/butane-x86_64-unknown-linux-gnu"

### Setup the Provider Account

Oracle provides Compartments to aggregate resources. The compartments also can be used to apply policies of permissions for those resources.

The Control Plane and Compute nodes Compartments are nested to allow fine granted permissions when allowing resources running in Control Plane to access the OCI APIs/resources.

Create the Compartments:

```
[parent]
 '- openshift
   '- ocp-control-plane
   '- ocp-compute
```

Export variables for each compartment

```bash
export COMPARTMENT_ID_OPENSHIFT=
export COMPARTMENT_ID_OPENSHIFT_CPL=
export COMPARTMENT_ID_OPENSHIFT_CMP=
```

### Upload the RHCOS image

- Create the bucket `openshift-infra`

```bash
oci os bucket create --name openshift-infra --compartment-id $COMPARTMENT_OPENSHIFT
```

- Download the QCOW2 image

~~~bash
export IMAGE_NAME=$(basename ./openshift-install coreos print-stream-json | jq -r '.architectures["x86_64"].artifacts["openstack"].formats["qcow2.gz"].disk.location')
wget $(./openshift-install coreos print-stream-json | jq -r '.architectures["x86_64"].artifacts["openstack"].formats["qcow2.gz"].disk.location')
~~~

- Upload the image to OCI Bucket

```bash
oci os object put -bn openshift-infra --name images/${IMAGE_NAME} --file ${IMAGE_NAME}
```

- Import to the Instance Image service

> [OCI CLI Help](https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.29.1/oci_cli_docs/cmdref/compute/image/import/from-object.html)

```bash
oci compute image import from-object -bn openshift-infra --name images/${IMAGE_NAME} \
    --compartment-id $COMPARTMENT_OPENSHIFT -ns ${BUCKET_NAMESPACE} \
    --display-name ${IMAGE_NAME} --launch-mode "PARAVIRTUALIZED" \
    --source-image-type "QCOW2"
```

## Create Infrastructure resources


### Identity

Allow access from control planes, running CCM's replicas, to the OCI API compartment.

- Create Dynamic Group

```
Any {instance.compartment.id = '$COMPARTMENT_ID_OPENSHIFT_CPL'}
```

- Create policies:

```
Allow dynamic-group ocp-compute to manage volume-family in compartment openshift
Allow dynamic-group ocp-compute to manage instance-family in compartment openshift
Allow dynamic-group ocp-compute to manage security-lists in compartment openshift
Allow dynamic-group ocp-compute to use virtual-network-family in compartment openshift
Allow dynamic-group ocp-compute to manage load-balancers in compartment openshift
```

### Network

Create the VCN and dependencies with the following configuration:

> TODO/WIP

| Resource | Value | Note |
| -- | -- | -- |
| VCN | CIDR 10.0.0.0/16 | |
| Public Subnet | 10.0.0.0/20 | Regional,Resolve DNS |
| Private Subnet | 10.0.128.0/20 | Regional,Resolve DNS  |
| Nat GW | TODO | |
| Internet GW | TODO | |

Network Security Groups:

- Control Planes

> TODO/WIP

| Rule Type | Source | DST Port | Note |
| -- | -- | -- | -- |
| Inbound | | | |
| Outbound | | | |

> TODO

- Worker

> TODO

- Load Balancer

> TODO

### DNS

> TODO

### Load Balancers

- Create the Network Load Balancer

Backend Sets (BSet)

| BSet Name | Port | Health Check (Proto/Path/Interval/Timeout) |
| -- | -- | -- |
| TODO | | |

## Preparing the installation


### Create the install-config.yaml

Create the install-config.yaml setting the Platform Type as `None`:

> TODO: the install-config.yaml must be adapted to set the `platform.external.*` when [the PR in the `openshift-installer`](https://github.com/openshift/installer/pull/7217) is merged.

> The Infrastructure manifest will be temporarially patched to `External`, replacing the `None` type

```bash
# Change Me
export INSTALL_DIR=./install-dir
export CLUSTER_NAME=oci-ext00
export BASE_DOMAIN=splat-oci.devcluster.openshift.com

export SSH_PUB_KEY_FILE="${HOME}/.ssh/id_rsa.pub"
export PULL_SECRET_FILE="${HOME}/.openshift/pull-secret-latest.json"

# Create the install directory
mkdir -p $INSTALL_DIR
```

Create the `install-config.yaml`:

```bash
cat <<EOF > ${INSTALL_DIR}/install-config.yaml
apiVersion: v1
baseDomain: ${BASE_DOMAIN}
metadata:
  name: "${CLUSTER_NAME}"
platform:
  none: {}
publish: External
pullSecret: >
  $(cat ${PULL_SECRET_FILE})
sshKey: >
  $(cat ${SSH_PUB_KEY_FILE})
EOF
```

### Create manifests

```bash
./openshift-install create manifests --dir $INSTALL_DIR
```

#### Patch Infrastructure Object

- Create the patch object for Platform External:

```bash
cat <<EOF > patch_cluster-infrastructure-02-config.yml
spec:
  platformSpec:
    external:
      platformName: oci
    type: External
status:
  platform: External
  platformStatus:
    type: External
    external:
      cloudControllerManager:
        state: External
EOF
```

- Apply the patch to the Infrastructure manifest

```bash
./yq eval-all -i '. * load("patch_cluster-infrastructure-02-config.yml")' $INSTALL_DIR/manifests/cluster-infrastructure-02-config.yml
```

#### Create manifests for CCM

- Create the namespace manifest

> TODO: explain why we don't advice to create components in namespaces `kube-*` or `openshift-*`

```bash
export OCI_CCM_NAMESPACE=oci-cloud-controller-manager

cat <<EOF > ${INSTALL_DIR}/manifests/oci-00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: $OCI_CCM_NAMESPACE
  labels:
    "pod-security.kubernetes.io/enforce": "privileged"
    "pod-security.kubernetes.io/audit": "privileged"
    "pod-security.kubernetes.io/warn": "privileged"
    "security.openshift.io/scc.podSecurityLabelSync": "false"
    "openshift.io/run-level": "0"
    "pod-security.kubernetes.io/enforce-version": "v1.24"
EOF
```

- Create the OCI cloud-config secret:

```bash
# Change Me:
export OCI_CONFIG_CLUSTER_REGION=us-sanjose-1
export OCI_CCM_COMPARTMENT_ID=<ChangeMe:openshift compartment>
export OCI_VCN_ID=ChangeMe
export OCI_LB_SUBNET_ID=ChangeMe

# Create the secret manifest
cat <<EOF1 > ${INSTALL_DIR}/manifests/oci-01-ccm-00-secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: oci-cloud-controller-manager
  namespace: $OCI_CCM_NAMESPACE
data:
  cloud-provider.yaml: $(cat <<EOF2 | base64 -w0
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
EOF2
)
EOF1
```

- Download manifests from [OCI CCM's Github](https://github.com/oracle/oci-cloud-controller-manager) and save it in the directory `${INSTALL_DIR}/manifests`:

```bash
export RELEASE=v1.25.0

wget https://github.com/oracle/oci-cloud-controller-manager/releases/download/${RELEASE}/oci-cloud-controller-manager-rbac.yaml -O oci-cloud-controller-manager-rbac.yaml

wget  https://github.com/oracle/oci-cloud-controller-manager/releases/download/${RELEASE}/oci-cloud-controller-manager.yaml -O oci-cloud-controller-manager.yaml
```

- Patch the RBAC file setting the correct namespace

```bash
./yq ". | select(.kind==\"ServiceAccount\").metadata.namespace=\"$OCI_CCM_NAMESPACE\"" oci-cloud-controller-manager-rbac.yaml > ${INSTALL_DIR}/manifests/oci-01-ccm-01-rbac.yaml
```

- Patch the CCM DaemonSet manifest setting the namespace, append the tolerations, mount CA, and add env vars for the kube API URL used in OpenShift:

```bash
# Create the pod template patch
cat <<EOF > ./patch1_oci-cloud-controller-manager.yaml
metadata:
  namespace: $OCI_CCM_NAMESPACE
spec:
  template:
    spec:
      tolerations:
        - key: node.kubernetes.io/not-ready
          operator: Exists
          effect: NoSchedule
        - key: CriticalAddonsOnly
          operator: Exists
      volumes:
        - name: trusted-ca
          configMap:
            name: ccm-trusted-ca
            items:
              - key: ca-bundle.crt
                path: tls-ca-bundle.pem
EOF

# Create the containers' patch (TODO merge both patches)
cat <<EOF > ./patch2_oci-cloud-controller-manager.yaml
spec:
  template:
    spec:
      containers:
        - volumeMounts:
            - name: trusted-ca
              mountPath: /etc/pki/ca-trust/extracted/pem
              readOnly: true
          env:
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
./yq eval-all '. as $item ireduce ({}; . *+ $item)' oci-cloud-controller-manager.yaml patch1_oci-cloud-controller-manager.yaml > patched1_oci-cloud-controller-manager.yaml

# Merge required objects for containers
./yq eval-all '.spec.template.spec.containers[] as $item ireduce ({}; . *+ $item)' patched1_oci-cloud-controller-manager.yaml patch2_oci-cloud-controller-manager.yaml > patched2_oci-cloud-controller-manager.yaml

# join merges
./yq eval-all '.spec.template.spec.containers[] *= load("patched2_oci-cloud-controller-manager.yaml")' patched1_oci-cloud-controller-manager.yaml  > ${INSTALL_DIR}/manifests/oci-01-ccm-02-daemonset.yaml
```

The following CCM manifests files must be created in the installation `manifests/` directory:

```bash
$ tree $INSTALL_DIR/manifests/
install-dir/manifests/
├── oci-00-namespace.yaml
├── oci-01-ccm-00-secret.yaml
├── oci-01-ccm-01-rbac.yaml
└── oci-01-ccm-02-daemonset.yaml
```

#### Create custom manifests for Kubelet

The Machine Config Operator will set the Kubelet flag `--provider` to the value `external`, then it is expected to the provider ID must be set.

The Provider ID must be set dynamically for each node. The steps below describes how to create a MachineConfig manifest to setup a systemd unit to create a kubelet configuration discoverying the Provider ID in OCI by querying the [Instance Metadata Service (IMDS)](https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/gettingmetadata.htm).

- Create the butane file

```bash
function create_machineconfig_kubelet() {
    local node_role=$1
    cat << EOF > ./mc-kubelet-$node_role.bu
# Butane file to setup kubelet for node role $node_role
variant: openshift
version: 4.12.0
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
        Environment="KUBELET_PROVIDERID=providerName://\${PROVIDERID}"
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

- Process the butane files:

```bash
function process_butane() {
    local src_file=$1; shift
    local dest_file=$1

    ./butane $src_file -o $dest_file
}

process_butane "./mc-kubelet-master.bu" "${INSTALL_DIR}/openshift/99_openshift-machineconfig_00-master-kubelet-providerid.yaml"
process_butane "./mc-kubelet-worker.bu" "${INSTALL_DIR}/openshift/99_openshift-machineconfig_00-worker-kubelet-providerid.yaml"
```

### Create ignition files

Once the manifests are placed, you can create the cluster ignition configurations:

~~~bash
./openshift-install create ignition-configs --dir $INSTALL_DIR
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

- Upload the `bootstrap.ign` to the infrastructure bucket

> [OCI CLI help](https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.29.1/oci_cli_docs/cmdref/os/object/put.html)

```bash
oci os object put -bn openshift-infra --name bootstrap-${CLUSTER_NAME}.ign --file $INSTALL_DIR/bootstrap.ign
```

- Generate the signed URL for the bootstrap object to be used in the user-data:

```bash
# Presigned URL must expire in one hour
EXPIRES_TIME=$(date -d '+1 hour' --rfc-3339=seconds)
BOOTSTRAP_URI=$(oci os preauth-request create -bn openshift-infra -on bootstrap-${CLUSTER_NAME}.ign --access-type AnyObjectRead --name bootstrap-${CLUSTER_NAME} --time-expires $EXPIRES_TIME | jq -r '.data.["access-uri"]')

export USER_DATA_URL="https://objectstorage.${CLUSTER_REGION}.oraclecloud.com${BOOTSTRAP_URI}"
```

## Create compute nodes

### Bootstrap

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

| Attribute  | Value  | Note |
| -- | -- | -- |
| | | |
| user data | user-data-bootstrap.json | |

### Control Plane

| Attribute  | Value  | Note |
| -- | -- | -- |
| [...] | | |
| user data | $INSTALL_DIR/master.ign | |

### Compute/workers

| Attribute  | Value  | Note |
| -- | -- | -- |
| [...] | | |
| user data | $INSTALL_DIR/worker.ign | |


### Day 1 tasks

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
./openshift-install --dir <installation_directory> wait-for bootstrap-complete \
    --log-level=info
```

You can delete the bootstrap instance (or use it as bastion node until the instlalation has been completed).

#### Approve certificates

```bash
oc adm certificate approve $(oc get csr  -o json |jq -r '.items[] | select(.status.certificate == null).metadata.name')
```

#### Check installation complete

```bash
watch -n5 oc get clusteroperators

./openshift-install --dir <installation_directory> wait-for install-complete
```

## Next

TBD