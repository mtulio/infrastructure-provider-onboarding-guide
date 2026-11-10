# Installing a cluster with Platform External using CAPOCI

This is the **reference implementation** of
[Installing a cluster with Platform External and CAPI](./capi.md), using OCI/Oracle as the
cloud provider and [Cluster API Provider Oracle Cloud Infrastructure (CAPOCI)](https://github.com/oracle/cluster-api-provider-oci), an OpenShift out-of-tree cluster api provider,
as the infrastructure provisioner to have an OpenShift cluster.

OCI platform has been choosed for reference implementation as it already support their own UPI installation method with platform type External using agnost-installation method, and exploring their CAPI implementation will provide a full example of an out-of-tree provider implementation can take advantage of this installationmethod, encoraging any provider to explore this method as well.

!!! warning "Status: Developer Preview, and both dependencies are unmerged"

    This page describes a Developer Preview path targeted at OpenShift 5.1. It depends on
    two changes that are **not yet in a released artifact**:

    - The example tree, the `Containerfile` and the payload contents come from an example repository friednly called as BAR provider which is reference implementation of packaging CAPOCI and required interfaces for installer to provision with CAPI; See the repository
      [cluster-api-provider-bar-openshift](https://github.com/mtulio/cluster-api-provider-bar-openshift/tree/devel-enhancement) for more information.

!!! info "Why this page matters more than the AWS one"

    The installer **embeds CAPA**, so a successful AWS install cannot prove that the
    binary the payload supplied is the binary that ran. It does not embed CAPOCI and
    cannot. This arm is the evidence for the central claim: that a provider the installer
    has never heard of can provision an OpenShift cluster.

## Status of this arm

As of 2026-11-10 this path provisions the cloud infrastructure, boots the control plane
and reaches three `Ready` control-plane nodes with the cloud controller manager running.
It does **not** yet reach a fully converged cluster. What is proven and what is not:

| | |
| --- | --- |
| Payload image resolved, extracted, rendered; CAPOCI started from it | works |
| VCN, subnets, gateways, network load balancer, six instances | works |
| `infraReady` hook — private zone, resolver view, 22623 listener | works |
| `ignition` hook — bootstrap ignition staged to object storage behind a PAR | works |
| `postProvision` hook — backends, DNS, node certificate approval | works |
| `preDestroy` hook and `destroy cluster` — no orphaned resources | works |
| Control plane: three nodes `Ready`, CCM running | works |
| Compute nodes joining | **blocked**, see Known limitations |
| Full cluster convergence (34/34 cluster operators) | **not reached** |

## What the payload image is

The same contract as the AWS arm: one OCI image carrying the provider controller, the
Cluster API templates, the day-0 manifests, the hooks, and an `openshift.yaml` descriptor
that declares the layout. See
[Installing a cluster with Platform External and CAPI](./capi.md).

Two things are specific to this provider and are the reason this page exists.

### The controller's command line is the payload's, and three flags are not optional

`spec.capi.controllerArgs` in `openshift.yaml` is the **whole** command line, not an
addition to one the installer builds. Three entries are easy to omit and each produces a
failure that names something else:

```yaml
controllerArgs:
- -v=2
- --health-probe-bind-address={{suggestHealthHostPort}}
- --webhook-port={{.WebhookPort}}
- --webhook-cert-dir={{.WebhookCertDir}}
- --metrics-bind-address=0
- --init-oci-clients-on-startup=false
```

- **`--metrics-bind-address=0`.** CAPOCI's metrics server defaults to `:8080` on every
  interface. The installer allocates a free port for the health probe and the webhook,
  because it knows those flags exist; it has no token for a metrics address. Leave the
  default and anything else already using 8080 — including a concurrent install or
  teardown on the same host — makes the controller exit at start-up. The installer can
  only report `timeout waiting for process controller to start`.
- **`--init-oci-clients-on-startup=false`.** With the default, CAPOCI reads
  `AUTH_CONFIG_DIR` and builds one tenancy's client *before* it reconciles anything,
  exiting if the variable is unset. That default is written for how CAPOCI normally runs —
  a Deployment whose environment comes from a Secret. The installer runs it as a local OS
  process with no such environment, and this payload authenticates per cluster through an
  `OCIClusterIdentity`. The symptom of getting this wrong is
  `auth config directory environment variable is not set`, again behind a start-up timeout.
- **`--health-probe-bind-address`**, not `--health-addr`. Providers disagree on the
  spelling and the payload names its own.

### The credential Secret is not in the image, and you must place it yourself

`cluster-api/cluster.yaml` declares an `OCIClusterIdentity` whose `principalSecret` is
`<infra-id>-oci-credentials` in `openshift-cluster-api-guests`. **The payload cannot carry
it** — it holds an OCI API signing key, and an image carrying one tenancy's key is not a
provider payload.

So the Secret has to be applied to the installer's **local, temporary** Cluster API
control plane, and there is currently no declared mechanism for that. See
[Known limitations](#known-limitations).

## Prerequisites

- An OCI tenancy and a compartment you can create network, compute and load balancer
  resources in.
- The `oci` CLI configured, and `oc`, `jq`, `openssl` and `python3` on `PATH`.
- A **custom RHCOS image imported into the tenancy**, and its OCID. `platform: external`
  returns no boot image, so this is supplied by hand. The image must be the OpenStack
  variant — it is the only artifact whose Ignition provider reads OCI's metadata channel.
- An **object storage bucket** for the bootstrap ignition. The `ignition` hook uploads
  into it and mints a pre-authenticated request; it does not create the bucket.
- A **public DNS zone** for the base domain, managed by OCI DNS.
- A Secret document holding the six fields CAPOCI reads — `tenancy`, `user`,
  `fingerprint`, `key`, and optionally `passphrase` and `region` — kept outside any tree
  that gets archived or published.

## Section 1. Prepare the environment

Set the release image explicitly. **This is not optional.** Unset, the installer falls
back to the release its own build points at, which for a development build is
`registry.ci.openshift.org/origin/release:<version>` and needs CI credentials no ordinary
pull secret carries. The bootstrap then loops on

```
unable to retrieve auth token: invalid username/password: authentication required
```

and never starts `crio`, `kubelet`, `bootkube` or the machine config server. Nothing in
the install says "wrong release"; it looks like a bootstrap that simply never finishes.

```bash
export OPENSHIFT_INSTALL_RELEASE_IMAGE_OVERRIDE=quay.io/openshift-release-dev/ocp-release:<version>-x86_64
export OCI_PROVIDER_CREDENTIALS_FILE=/path/outside/this/tree/capoci-identity-secret.yaml
export OCI_CLI_CONFIG_FILE=/path/outside/this/tree/oci-config
```

## Section 2. Prepare the install configuration

```yaml
apiVersion: v1
metadata:
  name: <cluster-name>
baseDomain: <your-base-domain>
platform:
  external:
    platformName: bar
    cloudControllerManager: External
    clusterAPI:
      payloadImage: <registry>/cluster-api-provider-bar-openshift:<tag>
      # Consent to the payload's ignition hook -- the only hook handed the
      # bootstrap ignition, and therefore the cluster's day-0 secret material.
      # Without it the install is refused at PreProvision, before any cloud
      # resource exists.
      bootstrapIgnitionStaging: Allowed
      variables:
        OCI_COMPARTMENT_ID: <compartment-ocid>
        OCI_REGION: <region>
        OCI_IMAGE_ID: <rhcos-custom-image-ocid>
        OCI_CCM_IMAGE: <oci-ccm-image>
controlPlane:
  name: master
  replicas: 3
compute:
- name: worker
  replicas: 0
pullSecret: ""
sshKey: ""
```

!!! warning "Set `sshKey`"

    The bootstrap machine is given a public address precisely so that
    `openshift-install gather bootstrap` can reach it when an install fails. Without an
    `sshKey` the port answers and refuses every key, which is an open port rather than a
    diagnostic path.

## Section 3. Place the provider credential in the local control plane

Start the install, then — while it runs — apply the credential Secret to the installer's
local Cluster API control plane, named for this run's infrastructure ID:

```bash
KC=<install-dir>/.clusterapi_output/envtest.kubeconfig
INFRA=$(oc --kubeconfig "$KC" get ocicluster -A -o jsonpath='{.items[0].metadata.name}')
# Rewrite metadata.name to "${INFRA}-oci-credentials" and apply. Do not echo the file.
```

The window is the `infrastructureReady` timeout. Until the Secret exists, CAPOCI logs
`Unable to fetch ClientSecret: Secret "<infra-id>-oci-credentials" not found` and builds
nothing.

!!! danger "This is a gap in the contract, not a step that should exist"

    No hook phase can do this for you. `PreProvision` runs before the local control plane
    exists; `infraReady` runs only after `infrastructureReady`, which is the thing the
    credential is blocking. Teardown is different — `preDestroy` runs after the control
    plane is up and the payload's hook restores the Secret itself, so `destroy cluster`
    needs no manual step.

## Section 4. Create the cluster

```bash
openshift-install create cluster --dir=<install-dir> --log-level=debug
```

Node certificates are approved by the payload's `postProvision` hook, checked against the
`openshift-cluster-id` freeform tag on the instances. There is no manual approval step.

## Section 5. Destroy the cluster

```bash
openshift-install destroy cluster --dir=<install-dir> --log-level=debug
```

`preDestroy` removes what Cluster API does not own: the DNS records and private zone, the
22623 listener and backend set, the pre-authenticated request and the staged ignition
object, and the ingress load balancer the cluster's own CCM created — which `destroy
cluster` would otherwise orphan, because it deletes the machines the Service controller
ran on rather than the Service.

## Diagnosing a failed install

The bootstrap has a public address and your `sshKey`, so the ordinary tools work:

```bash
ssh core@<bootstrap-public-ip> 'sudo systemctl is-system-running; sudo journalctl -u bootkube'
ssh -J core@<bootstrap-public-ip> core@<master-private-ip> ...   # the others, through it
```

If the hook fails, it captures the serial console of every instance into
`<install-dir>/external-install/hook-state/console/`, one file per machine, mode 0600,
with URLs redacted — this works without SSH and without a public address, which is what
makes it useful when the machines never came up. The installer's error names that
directory first.

## Known limitations

- **Compute nodes do not join during bootstrap.** The bootstrap machine config server
  **refuses to serve the `worker` pool by design** — `refusing to serve bootstrap
  configuration to pool "worker"` — and day-0 Cluster API worker Machines boot while the
  bootstrap is still the only MCS backend. The network load balancer's `FIVE_TUPLE`
  policy pins each worker's flow to one backend, so a worker that hashes to the bootstrap
  retries against it indefinitely. Day-0 compute on this platform needs the bootstrap to
  be gone before workers can configure, and nothing approves their certificates by then.
- **The provider credential must be placed by hand during the install** (Section 3).
- **The CCM credential is unresolved on this arm.** The payload renders the CCM/CSI
  configuration with `api-key` auth, which gives the CCM a *user's* permissions across the
  whole tenancy rather than a dynamic group's scoped to one compartment. Instance
  principals are the correct production answer and are not wired up here.
- **`AUTH_CONFIG_DIR` and the metrics port are provider knowledge the payload must
  declare.** The installer cannot know a provider needs them. A payload that omits either
  fails at start-up with a message that names neither.
- **A local caching resolver can hold a negative answer** for `api.<cluster>.<domain>`
  from before the record was published. The hooks wait for it to expire and attempt a
  flush; a host where neither works needs `resolvectl flush-caches` by hand.
- **The RHCOS boot image must be imported by hand**, because `platform: external` returns
  none. A generic artifact-override mechanism is separate, open work.
- **This is Developer Preview** and depends on two unmerged changes; see the warning at
  the top of this page.

## References

- [Installing a cluster with Platform External and CAPI](./capi.md) — the generic guide this implements
- [Installing a cluster with Platform External using CAPA](./capi-ref-image-aws.md) — the AWS arm
- [Installing a cluster with Platform External](./agnostic.md) — the UPI-style path, without CAPI
- [OpenShift components supported on External Platform](../components.md)
- [cluster-api-provider-oci](https://github.com/oracle/cluster-api-provider-oci) — CAPOCI upstream
