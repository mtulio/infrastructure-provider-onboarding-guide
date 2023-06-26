# Validate Deployment of OpenShift Data Foundation (ODF) with LSO

The steps below describe how to run acceptance tests manually for ODF (OpenShift Data Foundation) using LSO (Local Storage Operator).

Prerequisites

- OpenShift Cluster Installed in version 4.13 (or newer). Export the variable `CLUSTER_VERSION`. Example: `export CLUSTER_VERSION=4.13`
- Cluster name exported in the variable `CLUSTER_NAME`
- Cluster administrator credentials is saved under ${INSTALL_DIR}/auth, example:

~~~bash
$ tree ${INSTALL_DIR}
└── auth
    ├── kubeadmin-password
    └── kubeconfig
~~~

Steps:

1) Pull the image:

```bash
 podman login quay.io
 podman pull quay.io/ocsci/ocs-ci-container:stable
```

2) Run a simple test:

> The ":Z" option for the volume mount can be omited when running in OS different than Linux.

~~~bash
podman run -v ${INSTALL_DIR}:/opt/cluster:Z quay.io/ocsci/ocs-ci-container:stable run-ci --cluster-path /opt/cluster --ocp-version ${CLUSTER_VERSION} --ocs-version ${CLUSTER_VERSION} --cluster-name ${CLUSTER_NAME} tests/manage/z_cluster/test_noobaa_xss_vulnerability.py --dev-mode
~~~

3) Run acceptance suite:

~~~bash
podman run -v ${INSTALL_DIR}:/opt/cluster:Z quay.io/ocsci/ocs-ci-container:stable run-ci -m acceptance --junit-xml /opt/cluster/junit.xml --cluster-path /opt/cluster --ocp-version ${CLUSTER_VERSION} --ocs-version ${CLUSTER_VERSION} --color=yes --cluster-name ${CLUSTER_NAME} tests/
~~~

> The results must be saved in `${INSTALL_DIR}/junit.xml`.

4) Collect the Must Gather for ODF

~~~bash
oc adm must-gather --image=registry.redhat.io/odf4/ocs-must-gather-rhel8:v4.9
~~~

Collect the data and attach the file `odf-tests.tar.gz` to the support case:

~~~bash
tar cfz odf-tests.tar.gz must-gather.local* ${INSTALL_DIR}/junit.xml
~~~