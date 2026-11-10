# CAPI Payload Integrator

The CAPI installation method with platform type `External` is performed
through an external payload OCI container image built by the Cloud Provider Integrator.

That OCI image provides declarative interfaces used by installer to
perform the CAPI provider installation, specially the installer hooks which provides an extended method to allow the provider engineering to implement nuances that the CAPI provider implementation does not cover, such as DNS management, identities, Object Storage signed URLs for bootstrap ignitions, CCM configuration at install time, etc. All that interfaces are maitained by the integrator/provider teams, it is only called by installer to provide a fully IPI experience.

The image is built and published by the integrator, and the user/cluster creator defines it install-config.yaml, then installer pulls it when the `create cluster` is triggered.

The steps bellow describes how the integrator can create, build and publish an "openshift-provider" payload image.

## Section 1. Build and publish the provider payload image

### 1.1 Start a registry the installer can reach

The installer pulls through `oc`, which speaks the **registry HTTP API**. It does not read
podman's or docker's local image store. An image that exists only as
`podman images` output is invisible to the installer; it has to be in a registry.

For a single-host run that registry can be on the same host:

```bash
podman run -d --name capi-payload-reg -p 5000:5000 docker.io/library/registry:2
```

Confirm it answers:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/v2/_catalog
# 200
```

!!! info "TLS on a loopback registry"

    The installer passes `--insecure` to `oc` when the payload reference is a loopback
    address, so no TLS material is needed for `localhost:5000` or `127.0.0.1:5000`. A
    registry reachable by any other name needs a certificate `oc` trusts.

A published registry (quay.io, or your own) is required only when something **other than
this host** has to pull the image: CI, or another person reproducing your run.

### 1.2 Build the image

Clone [installer-labs](https://github.com/openshift-eng/installer-labs) and build from the
AWS/CAPA example directory. The build context is that directory:

```bash
cd installer-labs/external-capi/examples/aws-capa
podman build -f Containerfile -t "${PAYLOAD_IMAGE}" .
```

!!! warning "One image name, used four times"

    `$PAYLOAD_IMAGE` is used by the build, the push, the `skopeo` check and
    `install-config.yaml`, and all four must agree. Build under one name and push under
    another and you get two local tags with only one of them in the registry; the symptom
    arrives two steps later as `manifest unknown`, which does not mention the name.

    If you opened a new terminal since [Prerequisites](#prerequisites), re-export it before
    building.

The `Containerfile` is a two-stage build:

1. **Builder stage** — clones CAPA at a pinned commit and builds a statically linked
   `bin/controller` with `CGO_ENABLED=0`, then generates `infrastructure-components.yaml`
   and `metadata.yaml` with `make release-manifests` from that same checkout, so the
   bundle and the binary cannot disagree.
2. **Payload stage** — `FROM scratch`, containing only the declared payload.

`scratch` is deliberate. The descriptor declares every path the payload contains and the
installer **rejects an executable the payload did not declare**, so a conventional base
image would fail layout validation on its own `/bin` and `/usr/bin`.

#### Customize CAPA image (optional)

The CAPA commit is pinned by a build argument, to build from a custom image:

!!! info "`CAPA_REF` requires version skew validation"

    Changing the version of CAPA requires compatibility with OpenShift,
    requiring to review the `compatibility.kubernetes` in `payload/openshift.yaml`.

```bash
podman build -f Containerfile \
  --build-arg CAPA_REF=<full-40-character-sha> \
  --build-arg CAPA_REPO=https://github.com/<org>/cluster-api-provider-aws \
  -t "${PAYLOAD_IMAGE}" .
```


!!! warning "`CAPA_REF` must be the full 40-character SHA"

    The build does a shallow `git fetch origin <ref>`, which resolves a commit id
    server-side rather than matching a ref prefix. An abbreviated SHA — such as the
    12-character form carried in a Go pseudo-version — fails with
    `couldn't find remote ref`, which reads like a missing commit and is really a missing
    28 characters. Expand it with `git rev-parse <short-sha>` against a full clone.

### 1.3 Push it and verify it is readable

Send the image to the registry:

```bash
podman push --tls-verify=false "${PAYLOAD_IMAGE}"
```

Verify over the registry API — this is the same request the installer's first round trip
makes, so a failure here is a failure you want *before* an install directory exists:

```bash
skopeo inspect --tls-verify=false "docker://${PAYLOAD_IMAGE}" \
  | jq -r '"digest: \(.Digest)\ncreated: \(.Created)"'
```

Record that digest. The installer logs the digest it resolved, and the two must match.


### 1.4 Inspect the payload descriptor

The image root carries `openshift.yaml`, the OpenShift-specific half of the payload
contract, maitained, tested and distribured by partner. It declares the Kubernetes compatibility range and the layout:

```yaml
apiVersion: external.openshift.io/v1alpha1
kind: ProviderPayload

compatibility:
  kubernetes:
    min: "1.33"
    max: "1.37"

layout:
  controller: bin/controller
  customResource: infrastructure-components.yaml
  clusterTemplates: cluster-api/
  manifestTemplates: manifests/
  hooks:
    infraReady: hooks/infra-hook.sh
    postProvision: hooks/infra-hook.sh
    preDestroy: hooks/infra-hook.sh
```

Validation is exhaustive and happens before any cloud call: every declared path must
exist, the controller must be executable, the compatibility range must admit the release
being installed, and any executable the payload did **not** declare is rejected.

The other half of the contract is clusterctl's own `metadata.yaml`, copied verbatim from
the provider repository. It answers "which Cluster API contract does this provider
implement"; it says nothing about Kubernetes versions or where the binary and hooks live,
which is exactly the residue `openshift.yaml` carries.

The `compatibility` is used by installer to validate if the openshift-install binary, and release image used to create the cluster is compatible with the provider's installer payload for platform-external, this must be update by provider every time Cluster API provider binary is updated.

## Continuos Test Payload Image

Partner is responsible to continuous test the Payload image to ensure compatibility with OpenShift and openshift-install program.

openshift-install program provides the validation methods you can integrate with your pipeline to ensure readiness:

> TBD if `openshift-install external *` commands will be develired as a helper


## Reference implementation

- Partner helper reference implementation:
  - "FOO" (AWS with CAPA)
  - "BAR" (OCI with CAPOCI)