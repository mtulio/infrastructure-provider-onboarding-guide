# Installing a cluster with Platform External using CAPA

This is the **reference implementation** of
[Installing a cluster with Platform External and CAPI](./capi.md), using AWS as the
cloud provider and [Cluster API Provider AWS (CAPA)](https://github.com/kubernetes-sigs/cluster-api-provider-aws)
as the infrastructure provider.

> NOTE: All that code is maitained by Provider/Integrator. The reference implementation uses AWS, an built-in provider, to provide a reference implementation of out-of-three provider installation using CAPI.

Everything here is done **by hand**, one command at a time, so that each step of the
contract between `openshift-install` and a provider payload is visible. Automation that
performs the same steps lives in
[installer-labs](https://github.com/openshift-eng/installer-labs) under
`external-capi/examples/aws-capa/scripts/`; use it once you have read this page, not
instead of it.

AWS is used here because it is the provider most readers can reproduce. It is also the
weakest possible *evidence* that the mechanism works, and the guide says so where it
matters — see [What this reference does and does not prove](#what-this-reference-does-and-does-not-prove).

!!! warning "Status: Developer Preview, and both dependencies are unmerged"

    This page describes a Developer Preview path targeted at OpenShift 5.1. It depends on
    two changes that are **not yet in a released artifact**:

    - The example tree, the `Containerfile` and the payload contents come from
      [openshift-eng/installer-labs PR #12](https://github.com/openshift-eng/installer-labs/pull/12/changes).
    - `openshift-install` must be a build that supports
      `platform.external.clusterAPI.controller.sourceType: image`. That support is not in
      any released `openshift-install`; you need a binary built from the corresponding
      installer branch.

    Until both land, treat every command below as reproducible only against those two
    sources.

## What the payload image is

In the local-path mode described in [capi.md](./capi.md), you hand `openshift-install`
a set of filesystem paths: a controller binary, a components bundle, a directory of
Cluster API manifests, a directory of day-0 manifests, and hook scripts.

In **image mode** all of that is published as a single OCI image. The installer pulls it,
extracts it into the install directory, validates it against a descriptor the image
carries, renders the templates, and runs the controller as a local process.

The `image` mode is the only supported method for Providers delivers the CAPI automation
required for openshift-install consume it, as well the OCI images are the usual method
to distributed versioned content for Kubernetes components.
The local-path mode must be used only for local development.

| In the image | Extracted to | Reaches the cluster? |
| --- | --- | --- |
| `bin/controller` | `<install-dir>/capi-installer/bin/controller` | No — runs locally, as an OS process |
| `infrastructure-components.yaml` | `<install-dir>/capi-installer/` | No — only its CRDs and webhooks, applied to the local control plane |
| `cluster-api/` | rendered into `<install-dir>/external-install/` | No — applied to the local, ephemeral control plane |
| `manifests/` | rendered into `<install-dir>/external-install/extra-manifests/` | **Yes** — folded into `<install-dir>/manifests/` |
| `hooks/` | `<install-dir>/capi-installer/hooks/` | No — run on the installer host |
| `openshift.yaml` | `<install-dir>/capi-installer/` | No — the payload descriptor |

!!! info "The image carries no credential, and that is a requirement"

    A payload image must be a **generic, multi-tenant artifact**: one image that any
    customer of your cloud can pull. Nothing customer-specific goes in it. The AWS
    credentials the cloud controller manager needs are supplied separately, as a Secret
    you create in [Section 2.4](#24-create-the-cloud-credentials-secret). The components
    bundle's own `${AWS_B64ENCODED_CREDENTIALS}` slot is inert here, because the installer
    never applies that bundle's Deployment.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Section 1. Build and publish the provider payload image](#section-1-build-and-publish-the-provider-payload-image)
- [Section 2. Prepare the install configuration](#section-2-prepare-the-install-configuration)
- [Section 3. Create manifests](#section-3-create-manifests)
- [Section 4. Create the cluster](#section-4-create-the-cluster)
- [Section 5. Approve the compute node certificates](#section-5-approve-the-compute-node-certificates)
- [Section 6. Review the installation](#section-6-review-the-installation)
- [Section 7. Destroy the cluster](#section-7-destroy-the-cluster)
- [What this reference does and does not prove](#what-this-reference-does-and-does-not-prove)
- [Known limitations](#known-limitations)

## Prerequisites

Tooling on the machine running the install:

| Tool | Why |
| --- | --- |
| `openshift-install` | a build with payload-image support (see the warning above) |
| `oc` | **required in image mode.** The installer extracts the payload with `oc image extract` |
| `podman` | builds and pushes the payload image |
| `skopeo` | optional, used here to verify the image is readable over the registry API |
| `jq` | reads the RHCOS boot image from the CoreOS stream metadata |
| `aws` | not used by the installer, only for inspecting the account |

`oc` is a **runtime dependency of image mode that local-path mode does not have**. The
installer checks for it and refuses before creating anything in the cloud.

Accounts and credentials:

- An AWS account and an AWS shared credentials file. Export its path; never paste its
  contents anywhere:

```bash
export AWS_SHARED_CREDENTIALS_FILE="${HOME}/.aws/credentials"
export AWS_REGION="us-east-1"
```

- A Red Hat pull secret, saved to a file. Export the path, not the value:

```bash
export PULL_SECRET_FILE="${HOME}/.openshift/pull-secret.json"
```

- A public Route 53 hosted zone for your base domain, in the same account.

Shell variables used throughout this page:

```bash
export CLUSTER_NAME="example"
export BASE_DOMAIN="your-zone.example.com"
export INSTALL_DIR="install-dir-${CLUSTER_NAME}"
export PAYLOAD_IMAGE="localhost:5000/openshift-install-external-capi-aws:dev"
export RELEASE_IMAGE="quay.io/openshift-release-dev/ocp-release:5.1.0-ec.1-x86_64"
export OPENSHIFT_INSTALL_RELEASE_IMAGE_OVERRIDE="${RELEASE_IMAGE}"
```

!!! info "Developer builds of `openshift-install`"

    A `MODE=dev` build reads its data assets from disk rather than from the binary, and
    resolves them relative to the working directory. If you built the installer yourself,
    also export `OPENSHIFT_INSTALL_DATA=/path/to/installer/data/data` (note the doubled
    path component). A released binary does not need this.


## Section 2. Prepare the install configuration

```bash
mkdir "${INSTALL_DIR}"
```

### 2.1 Resolve the RHCOS boot image and the CCM image

`platform: external` returns no boot image from the installer,
so, for this use case we are using the supported AMI ID shipped by CoreOS and CCM image in OpenShift release Payload.

The `AWS_AMI` has to come from the CoreOS stream metadata:

!!! warning "Set `OPENSHIFT_INSTALL_DATA` for custom builts"

    Set the path of installer repository where the openshift-install has built:
    `OPENSHIFT_INSTALL_DATA=/path/to/installer`

```bash
RHCOS_AMI=$(./openshift-install coreos print-stream-json \
  | jq -r ".architectures.x86_64.images.aws.regions[\"${AWS_REGION}\"].image")

echo "${RHCOS_AMI}"
export RHCOS_AMI="${RHCOS_AMI}"
```

To ensure compatibility in the component skew, the cloud controller manager image
must match the release being installed. Resolve it from the release payload,
using a registry auth file built from your pull secret:

```bash
CCM_IMAGE=$(oc adm release info -a "${PULL_SECRET_FILE}" "${RELEASE_IMAGE}" \
  --image-for=aws-cloud-controller-manager)


echo "${CCM_IMAGE}"
export CCM_IMAGE="${CCM_IMAGE}"
```

### 2.2 Write `install-config.yaml`

Create the `install-config.yaml` from discovered variables to the installer directory:

```bash

cat <<EOF > "${INSTALL_DIR}/install-config.yaml"
apiVersion: v1
metadata:
  name: ${CLUSTER_NAME}
baseDomain: ${BASE_DOMAIN}
platform:
  external:
    platformName: aws
    cloudControllerManager: External
    clusterAPI:
      name: aws
      controller:
        sourceType: image
        image: ${PAYLOAD_IMAGE}
        args:
        - --feature-gates=BootstrapFormatIgnition=true,ExternalResourceGC=true,TagUnmanagedNetworkResources=false,EKS=false,MachinePool=false
      variables:
        AWS_AMI: ${RHCOS_AMI}
        AWS_REGION: ${AWS_REGION}
        AWS_AZ: ${AWS_REGION}a
        CCM_IMAGE: ${CCM_IMAGE}
controlPlane:
  name: master
  replicas: 3
compute:
- name: worker
  replicas: 0
pullSecret: '$(cat "${PULL_SECRET_FILE}")'
sshKey: |
    ${SSH_KEYS}
EOF
```

Four things in that file are not self-evident:

- **`sourceType: image`** makes the field set a union. When it is `image`, no local path is
consulted; `capiInstallerPath`, `customResource` and `manifestPath` all default beneath
`<install-dir>/capi-installer` from the payload's own layout, so a conforming payload needs
no path configuration at all.

- **`BootstrapFormatIgnition=true` is not optional.** Without it CAPA's validating webhook
rejects the `AWSCluster` outright — `spec.s3Bucket: Forbidden: can be set only if the
BootstrapFormatIgnition feature gate is enabled` — and the same gate is what makes
`AWSMachine.spec.ignition` legal.

- **`variables`** supplies the `${VAR}` tokens in the payload's template trees.
`INSTALLER_`-prefixed variables are supplied by the installer and must not appear here;
everything else is the provider's namespace, and the prefix is what keeps the two sets from
colliding as each grows. A token the payload uses that nothing resolves is a validation
error, before any cloud call.


!!! info "There is no `hooks` stanza, and that is the expected shape"

    The payload declares all three hooks in `openshift.yaml` and the installer defaults to
    what it declares. A payload hook must therefore be **self-sufficient**: the layout
    names a program and nothing else, so there is nowhere for a payload to carry
    arguments.

    Setting a hook in `install-config.yaml` overrides that phase **and only that phase**.
    Each hook's root follows where that hook came from — an overridden program resolves
    against `<install-dir>/external-install`, a payload-declared one against
    `<install-dir>/capi-installer` — so replacing `postProvision` alone is supported and
    does not oblige you to restate the other two.

    A phase that neither the payload nor the install-config names is skipped. That is fine
    for `postProvision` and `preDestroy`. Skipping `infraReady` means nothing publishes
    the `api-int` record, and the install will run to the bootstrap timeout and die in
    `resolve-api-int-url` unless something out of band creates it. The installer warns and
    continues rather than refusing, because out-of-band DNS is a real deployment.

### 2.4 The cloud credentials Secret — nothing to do

**There is no step here any more.** The payload's `postProvision` hook creates the
`aws-cloud-credentials` Secret itself, reading the same `AWS_SHARED_CREDENTIALS_FILE` the
installer was run with. Export that variable, which you already have to do for the
provider's controller, and the rest happens on its own.

This page used to ask you to write the Secret into
`<install-dir>/external-install/extra-manifests/` by base64-encoding the credentials file.
**Do not do that.** It puts a live cloud credential into the install directory, which is
the directory you keep, archive and attach to a support case. The hook never writes the
value anywhere: it pipes the file into `oc create secret --dry-run=client -o yaml | oc
apply -f -` and logs only its length.

!!! info "Why a Secret and not an instance profile"

    This reference creates **no IAM at all**, so the control-plane machines run with no
    instance profile and the CCM has no metadata-service credentials to pick up. That is
    the more representative case rather than a workaround: `platform: external` means the
    installer knows nothing about the provider's cloud, so it cannot create that cloud's
    identities.

    A partner running this for real may well prefer an instance profile, and can — the
    CCM Deployment falls back to the metadata service when this Secret is absent.

!!! warning "The fallback is silent, and the error it produces points at the wrong thing"

    If the Secret is missing, the AWS SDK does not fail at `/etc/aws/credentials`. It
    falls through its credential chain to the instance metadata service, finds no
    instance profile, and reports a **404 from IMDS**. That reads like "the machine needs
    an instance profile" and it is not — it is the last link of the chain, not the cause.
    The CCM then exits fatally rather than degrading, so the symptom is a
    `CrashLoopBackOff` with nodes stuck `NotReady` behind the
    `node.cloudprovider.kubernetes.io/uninitialized` taint.

## Section 3. Create manifests

This is where the payload is pulled, extracted, validated and rendered, so a failure in any
of it is reported **before the first cloud call**.

```bash
openshift-install create manifests --log-level=debug --dir="${INSTALL_DIR}"
```

Five log lines are the ones worth reading:

```text
Provider payload localhost:5000/openshift-install-external-capi-aws:dev resolved to digest sha256:<...>
Running oc image extract --path=/:<install-dir>/capi-installer --only-files --confirm --filter-by-os=linux/amd64
Extracted the provider payload into <install-dir>/capi-installer
Rendered the payload's Cluster API templates into <install-dir>/external-install
Rendered the payload's day-0 manifest templates into <install-dir>/external-install/extra-manifests
```

Check that:

- the **digest matches** the one `skopeo inspect` reported in [1.3](#13-push-it-and-verify-it-is-readable);
- `--filter-by-os` names the **installer host's** architecture, not the cluster's — the
  provider runs as a local OS process on this machine;
- nothing under `external-install/` still contains an unresolved token:

```bash
grep -rlE '\$\{[A-Za-z_][A-Za-z0-9_]*' \
  "${INSTALL_DIR}/external-install/cluster.yaml" \
  "${INSTALL_DIR}"/external-install/machines/*.yaml
# must print nothing
```

List what was extracted — **by filename and size only**:

```bash
find "${INSTALL_DIR}/capi-installer" -type f -printf '%10s %p\n' | sort -k2
```

!!! warning "Do not read the generated manifest tree"

    `<install-dir>/manifests/` and `<install-dir>/openshift/` are now a credential store:
    they hold the machine-config-server TLS private key, a CA private key, the pull secret,
    the cloud credentials and the kubeadmin password hash. They look exactly like a
    directory of hand-authored sample YAML, and they are not. Never publish a captured
    tree as an example; inspect it by filename and size.

Stopping here costs nothing and creates nothing in AWS. It is the right loop to iterate on
while what you are changing is the payload rather than the infrastructure.

## Section 4. Create the cluster

```bash
openshift-install create cluster --log-level=debug --dir="${INSTALL_DIR}"
```

The phases, in order, with what each one proves:

| Phase | What to look for |
| --- | --- |
| Payload reuse | `Reusing the provider payload already extracted into …(sha256:…)` — the second command reuses the first command's extraction and still reports the digest |
| Local control plane | envtest etcd and kube-apiserver start, then `Running process: aws infrastructure provider` |
| Network infrastructure | `Waiting up to 15m0s for network infrastructure to become ready...` |
| `infraReady` hook | creates the private hosted zone and the `api-int` and `api` records |
| Machines | bootstrap, 3 masters and the payload's workers provision |
| `postProvision` hook | waits for the ingress Service to be given a load balancer, then publishes `*.apps` |
| Bootstrap | `Bootstrap status: complete`, then `Destroying the bootstrap resources...` |
| Cluster operators | `Waiting … for the cluster to initialize`, then `Install complete!` |

On a reference run the `create cluster` phase took **38m28s**, of which roughly 14 minutes
was the `postProvision` hook waiting for the ingress load balancer — that hook cannot
finish until the cluster is far enough along for the ingress operator to run, and that is
expected rather than a stall.

!!! info "`metadata.json` records which payload built the cluster"

    After `create manifests`, `<install-dir>/metadata.json` carries the resolved
    `image` **and** `imageDigest`, plus the hook roots. That file travels with a support
    bundle and is the only record of which build of a provider a cluster was installed by.
    Check it is populated:

    ```bash
    jq '.external.clusterAPI | {image, imageDigest, hookRoot}' "${INSTALL_DIR}/metadata.json"
    ```

!!! warning "The install log contains the kubeadmin password"

    If you redirected the install to a file, do not `cat` or `tail` it wholesale, and do
    not paste it into a ticket. Grep it narrowly instead:

    ```bash
    grep -c 'Install complete!' install.log
    grep -c 'level=fatal' install.log
    ```

## Section 5. Compute node certificates — approved by the payload

**There is no manual step here any more.** The payload's `postProvision` hook approves the
node certificates and waits for every expected node to become `Ready` before it returns.
A run that previously needed `oc adm certificate approve` twice, a minute apart, now
completes unattended.

```bash
export KUBECONFIG="${PWD}/${INSTALL_DIR}/auth/kubeconfig"
oc get nodes
oc get csr --no-headers | awk '$NF=="Pending"' | wc -l    # expect 0
```

All control-plane and compute nodes should be `Ready`, with no pending requests.

!!! info "What the hook approves, and why it is not `approve everything pending`"

    A reference implementation that rubber-stamped every pending CSR would be teaching
    partners to hand out node identities to anyone who can file a request. The hook
    approves only the `kubernetes.io/kube-apiserver-client-kubelet` and
    `kubernetes.io/kubelet-serving` signers, only from the expected requesters, and only
    when the CSR's `system:node:<name>` subject matches an instance carrying this
    cluster's ownership tag. Anything it cannot verify, it refuses and logs.

!!! warning "This covers day 0 only, and the underlying gap is unchanged"

    `cluster-machine-approver` is deployed and running on an External cluster and
    approves nothing, because it resolves a CSR by looking the node up in the **Machine
    API** and this platform has no Machine API Machines:

    ```
    csr-xxxxx: failed to find machine with InternalDNS matching ip-10-0-7-140, cannot approve
    ```

    The hook is a demonstration that the comparison is cheap and reliable, not a proposal
    that hooks are where this belongs. It runs on the installer host and exits, so
    certificate rotation, node replacement and scale-up after the install have nothing
    approving them. The durable answer is an in-cluster controller; it affects
    `platform: external` and `platform: none` alike and is tracked separately.

## Section 6. Review the installation

Review the installed version, cluster operators state, and nodes:

```bash
oc get clusterversion
oc get co
oc get nodes -o wide
```

**The platform type is what you asked for:**

```bash
oc get infrastructure cluster -o jsonpath='{.status.platform}{"\n"}'
# External
```

The checks that are specific to this install method:

### Cloud Controller Manager

**The CCM from the payload is running and initialised the nodes.** If `providerID` is
empty or a node still carries the `node.cloudprovider.kubernetes.io/uninitialized` taint,
the CCM never ran — the Secret in [2.4](#24-create-the-cloud-credentials-secret) is the
first thing to check.

```bash
oc get deployment -n openshift-cloud-controller-manager
oc get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.providerID}{"\n"}{end}'
oc get nodes -o jsonpath='{.items[*].spec.taints[*].key}' | tr ' ' '\n' | grep -c uninitialized
# must print 0
```

### Default Router

**The `*.apps` record the `postProvision` hook published resolves.**

```bash
curl -sk -o /dev/null -w '%{http_code}\n' \
  "https://$(oc get route console -n openshift-console -o jsonpath='{.spec.host}')"
# 200
```

## Section 7. Destroy the cluster

Destroy goes through **the provider that created the infrastructure**, so it needs three
things the install left behind:

- `<install-dir>/metadata.json`, which records the provider artifact paths and digest;
- `<install-dir>/.clusterapi_output/*.yaml`, the objects with their resource IDs;
- the payload itself, still extracted at `<install-dir>/capi-installer/`.

Keep the install directory. Without it there is nothing to destroy from.

```bash
openshift-install destroy cluster --log-level=debug --dir="${INSTALL_DIR}"
```

The `preDestroy` hook runs first and removes what CAPA did not create and therefore cannot
delete: the ingress load balancer and its target groups, the public `api` and `*.apps`
records, and the private hosted zone. Then CAPA deletes the machines, load balancers,
security groups, route tables, NAT gateway and VPC.

Verify against your own account rather than trusting the exit code — and **scope every
query to the cluster**, because an account-wide count tells you nothing:

```bash
INFRA_ID=$(jq -r .infraID "${INSTALL_DIR}/metadata.json")
VPC_ID=<the VPC id from the install log>

aws ec2 describe-instances \
  --filters "Name=tag-key,Values=kubernetes.io/cluster/${INFRA_ID}" \
  --query "length(Reservations[].Instances[?State.Name!='terminated'][])"
aws ec2 describe-vpcs --filters "Name=tag-key,Values=kubernetes.io/cluster/${INFRA_ID}" --query 'length(Vpcs)'
aws elbv2 describe-load-balancers --query "length(LoadBalancers[?VpcId=='${VPC_ID}'])"
aws elbv2 describe-target-groups  --query "length(TargetGroups[?VpcId=='${VPC_ID}'])"
aws route53 list-hosted-zones --query "length(HostedZones[?contains(Name, '${CLUSTER_NAME}.')])"
```

Every count must be `0`.

!!! warning "Run teardown twice, against two different clusters"

    A destroy path that works on the cluster it was developed against routinely fails on
    the next one, because the second cluster has resources the first did not. Treat a
    single clean teardown as unverified.

## What this reference does and does not prove

The installer **embeds CAPA**. So on AWS, a successful install cannot distinguish "ran the
binary the payload handed it" from "ran the binary it already had". This page is a faithful
demonstration of the *workflow* and a weak demonstration of the *mechanism*.

The claim that a provider genuinely external to the installer can drive the install is
established by a provider that is not embedded — CAPOCI on Oracle Cloud Infrastructure is
the arm used for that. If you are evaluating this path for your own cloud, the OCI arm is
the one to read; this one is the one to reproduce.

Two things here, however, *are* real evidence even on AWS:

- the controller is built from provider source by the `Containerfile` at a pinned commit,
  not copied from the installer, so the binary the payload supplies is demonstrably its own;
- the payload carries no credential and no customer-specific value, which is the property
  that makes a provider image publishable at all.

## Known limitations

- **`oc` is a runtime dependency** of image mode. Local-path mode has no such dependency.
- **Nothing in the cluster approves node certificates after the install.** The payload's
  hook covers day 0 (see [Section 5](#section-5-compute-node-certificates-approved-by-the-payload));
  rotation, replacement and scale-up afterwards have no approver.
- **The compute count comes from the payload, not from `compute.replicas`.** The example
  tree declares two worker `Machine` objects statically, so an `install-config.yaml`
  asking for zero workers still gets two. Edit the payload's machine manifests to change
  it. This decoupling is a property of day-0 Cluster API machines, not a defect of this
  example, and it is why `oc get nodes` can disagree with your install-config.
- **A tagged payload reference is traceable but not reproducible.** The install log and
  `metadata.json` record the resolved digest, so you can tell after the fact which image
  ran — but the install-config alone does not pin it. A disconnected or mirrored install
  must pin a digest; `imageDigestSources` redirects by digest and validation enforces that
  when it is set.
- **The RHCOS boot image must be supplied by hand**, because `platform: external` returns
  none. A generic artifact-override mechanism is a separate, open piece of work.
- **This is Developer Preview** and depends on two unmerged changes; see the warning at the
  top of this page.

## References

- [Installing a cluster with Platform External and CAPI](./capi.md) — the generic guide this implements
- [Installing a cluster with Platform External](./agnostic.md) — the UPI-style path, without CAPI
- [OpenShift components supported on External Platform](../components.md)
- [installer-labs](https://github.com/openshift-eng/installer-labs) — the example tree and scripts,
  pending [PR #12](https://github.com/openshift-eng/installer-labs/pull/12/changes)
- [Cluster API Provider AWS](https://github.com/kubernetes-sigs/cluster-api-provider-aws)
- Enhancement proposal: *Introduce new platform type `External` in the OpenShift specific
  Infrastructure resource* —
  [openshift/enhancements](https://github.com/openshift/enhancements/blob/master/enhancements/cloud-integration/infrastructure-external-platform-type.md)
