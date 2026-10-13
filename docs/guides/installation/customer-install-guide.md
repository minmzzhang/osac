# Installing OSAC on OpenShift Container Platform (WIP)

**Last Updated**: 2026-10-13
**Audience**: Cloud administrators
**Status**: WIP — see the TBD notes in [Section 6.2](#62-installing-osac-for-caas-with-the-netris-network-backend)
and [Section 8.3](#83-registering-the-hub) for what's still open

---

Deploy OSAC (Open Sovereign AI Cloud) onto an existing Red Hat OpenShift
Container Platform cluster by using the OpenShift CLI (`oc`) and Helm. This
guide assumes the prerequisite Operators and infrastructure layer already
exist on the cluster. If you need to set those up yourself, see the
[Helm Deployment Guide](https://github.com/osac-project/osac/blob/main/docs/guides/installation/helm-deployment-guide.md)
first. The guide covers the VMaaS, CaaS, and BMaaS services.

## Contents

1. [OSAC installation architecture](#1-osac-installation-architecture)
2. [Prerequisites](#2-prerequisites)
3. [OSAC component versions](#3-osac-component-versions)
4. [Installing OSAC](#4-installing-osac)
5. [Helm chart configuration parameters](#5-helm-chart-configuration-parameters)
6. [Installation workflows by service](#6-installation-workflows-by-service)
7. [Verifying the installation](#7-verifying-the-installation)
8. [Postinstallation tasks](#8-postinstallation-tasks)
9. [Supported configurations](#9-supported-configurations)
10. [Uninstalling OSAC](#10-uninstalling-osac)
11. [Troubleshooting](#11-troubleshooting)
12. [Glossary](#12-glossary)

---

## 1. OSAC installation architecture

OSAC installs as three ordered Helm releases. Each release is a plain
`helm upgrade --install` command.

**Phase 1a: `osac-deps`.** Installs Operator Lifecycle Manager (OLM)
`Subscription` resources for the platform Operators in their own namespaces:
cert-manager and Ansible Automation Platform (AAP) by default, and standalone
multicluster engine, LVM Storage, MetalLB, OpenShift Virtualization, and Streams
for Apache Kafka when enabled. Every Operator is gated by a toggle — see
[Table 2.1](#table-21-platform-operators-and-components). Post-installation hooks wait for the cert-manager and AAP
`ClusterServiceVersion` (CSV) resources to reach `Succeeded`.

**Phase 1b: `osac-infra`.** Installs the shared, cluster-scoped resources: the
`default-ca` `ClusterIssuer`, trust-manager, the CA bundle `ConfigMap`,
Keycloak and the `osac` realm in the `keycloak` namespace, and the operand
custom resources (CRs) `HyperConverged`, `LVMCluster`, the MetalLB
`IPAddressPool`, and the Kafka CR. The `configure-*` hooks wait for each
remaining Operator CSV to reach `Succeeded` before applying its operand. For
testing, this phase can also deploy bundled PostgreSQL database and OpenBao.

**Phase 2: `osac`.** Installs the platform into your namespace: the OSAC
Operator and its CRDs, the Fulfillment Service with an Envoy sidecar, an AAP
instance and its bootstrap job, the OSAC web console, metering, the CSI driver,
and the Bare Metal Fulfillment Operator (BMaaS only). It connects to the
bundled OpenBao or the external secret store configured for the deployment.

This guide installs only phase 2. Phase 1a and phase 1b are set up ahead of
time by whoever prepares the cluster — see the
[Helm Deployment Guide](https://github.com/osac-project/osac/blob/main/docs/guides/installation/helm-deployment-guide.md)
if that's you.

### 1.1 What is released

The OSAC platform chart (phase 2) is published as an OCI artifact at
`oci://ghcr.io/osac-project/charts/osac`. The latest tagged release is
`0.0.17`, and a rolling `0.0.9-nightly.*` channel is also available. The chart
includes a `values.schema.json` file and a `values-example.yaml` file.

To install for a specific service, follow the workflow in
[Section 6](#6-installation-workflows-by-service). Each workflow lists the
toggles and values to set.

---

## 2. Prerequisites

### 2.1 Cluster and access

- You have a Red Hat OpenShift Container Platform 4.22 cluster (4.22.4 to
  4.22.6, channel `stable-4.22`) and the `cluster-admin` role on it.
- A default storage class exists. The pre-installation validation hook issues
  a warning if none exists. To set one, run the following command:

  ```console
  $ oc patch storageclass <storage_class_name> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
  ```

  For more information, see
  [Changing the default storage class](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/storage/dynamic-provisioning#change-default-storage-class_dynamic-provisioning).
- The cluster pull secret (`openshift-config/pull-secret`) authenticates to
  `registry.redhat.io` and `quay.io`. To download a current pull secret, see
  the pull secret page in the
  [Red Hat Hybrid Cloud Console](https://console.redhat.com/openshift/install/pull-secret).
  To apply it, see
  [Updating the global cluster pull secret](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/images/managing-images).
- `github.com`, `ghcr.io`, `quay.io`, and `registry.redhat.io` are reachable
  from the cluster: `github.com` and `ghcr.io` for the OSAC images and the
  `osac-ui` OCI chart, `quay.io` for Keycloak, its PostgreSQL, trust-manager,
  and the `origin-cli` hook image, and `registry.redhat.io` for the Red Hat
  Operator catalogs and operands.
- The command
  `oc get ingresses.config/cluster -o jsonpath='{.spec.domain}'` returns your
  apps domain. The installation derives all route host names from it.

### 2.2 Client tools

- The OpenShift CLI (`oc`), matching the cluster version. The installation and
  its hooks call `oc` and `kubectl`.
- Helm 3.8 or later, for OCI registry support.
- The `osac` CLI, latest release, for postinstallation hub registration and
  day-2 operations (see [Section 8.2](#82-installing-the-osac-cli)). Not
  required for the Helm installation.

Installing the published phase-2 chart requires only `oc` and `helm`.

### 2.3 Platform Operators and components

OSAC relies on the Operators and components in
[Table 2.1](#table-21-platform-operators-and-components). These are typically
installed by whoever prepares the cluster, ahead of your install — see the
[Helm Deployment Guide](https://github.com/osac-project/osac/blob/main/docs/guides/installation/helm-deployment-guide.md)
if that's you. To install an Operator yourself, use OperatorHub in the web
console, or apply a `Subscription` and `OperatorGroup` from the
`redhat-operators` catalog. For more information, see
[Adding Operators to a cluster](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/operators/user-tasks).

The Channel column lists the update channel that OSAC subscribes to. The
resolved CSV versions float as the channels publish updates; the versions
observed on OpenShift Container Platform 4.22.6 in September 2026 are listed in
[Section 3](#3-osac-component-versions).

<a id="table-21-platform-operators-and-components"></a>
**Table 2.1. Platform Operators and components**

| Component | Channel | Namespace | Toggle | Required for |
|---|---|---|---|---|
| [cert-manager Operator for Red Hat OpenShift](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/security_and_compliance/cert-manager-operator-for-red-hat-openshift) | `stable-v1` | `cert-manager-operator` | `certManager.enabled` | All services |
| trust-manager | `v0.20.0` (fixed) | `cert-manager` | `trustManager.enabled` | All services |
| `default-ca` `ClusterIssuer` | Not applicable | Cluster-scoped | `caIssuer.enabled` | All services |
| Keycloak and the `osac` realm | Not applicable | `keycloak` | `keycloak.enabled` | All services |
| [Red Hat Ansible Automation Platform Operator](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/installing_on_openshift_container_platform/index) | `stable-2.6-cluster-scoped` | `ansible-aap` | `aapOperator.enabled` | All services |
| [Streams for Apache Kafka](https://access.redhat.com/articles/6644711) | `stable` | `osac-kafka` | `kafka.enabled` | Fulfillment events and optional metering; an external Kafka cluster can serve fulfillment. See [Kafka configuration](kafka-configuration.md). |
| [OpenShift Virtualization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/virtualization/installing) | `stable` | `openshift-cnv` | `cnv.enabled` | VMaaS |
| [LVM Storage](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html-single/storage/index#persistent-storage-using-lvms) | `stable-<cluster_minor>` | `openshift-storage` | `lvms.enabled` | VMaaS |
| [MetalLB Operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/networking_operators/metallb-operator) | `stable` | `metallb-system` | `metallb.enabled` | VMaaS and CaaS |
| [multicluster engine for Kubernetes Operator](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13/html/clusters/cluster_mce_overview) | `stable-2.17` | `multicluster-engine` | `mce.enabled` | CaaS |

Notes on individual components:

- **cert-manager Operator for Red Hat OpenShift** installs into the
  `cert-manager-operator` namespace and creates its operands in the
  `cert-manager` namespace. The `certificates.cert-manager.io` CRD is a
  mandatory pre-installation check.
- **trust-manager** distributes the CA bundle. To use an existing
  trust-manager installation, set `trustManager.upstream.enabled=true`.
- **`default-ca` `ClusterIssuer`** is referenced by `service.certs.issuerRef`.
  If you disable `caIssuer.enabled`, provide your own issuer.
- **Keycloak** provides OIDC for the API and web console. This is a custom
  deployment, not the Red Hat build of Keycloak Operator.
- **Red Hat Ansible Automation Platform Operator** installs the Operator only.
  The `osac` chart creates the AAP instance. You supply the subscription
  manifest. See [Section 2.4](#24-credentials-and-external-services).
- **Streams for Apache Kafka** (formerly AMQ Streams) uses a manual install
  plan. Approve the install plan in the `osac-kafka` namespace if it does not
  progress.
- **OpenShift Virtualization** was formerly named Container-native
  Virtualization (CNV).
- **LVM Storage** uses a channel that tracks the OpenShift Container Platform
  minor version; the installation sets it. Any dynamic storage class works if
  you disable `lvms.enabled`.
- **MetalLB Operator** provides a `LoadBalancer`-class implementation. Any
  solution that provides one works if you disable `metallb.enabled`.
- **multicluster engine for Kubernetes Operator** is required for
  agent-based cluster provisioning but is disabled by default. Set
  `mce.enabled=true` when this installation owns standalone MCE; the `caas-ci`
  infrastructure profile does so explicitly. Leave it `false` when Red Hat
  Advanced Cluster Management for Kubernetes (RHACM) or another MCE
  installation already owns it. The disabled state also prevents the phase-1
  chart from applying its temporary Assisted image override `ConfigMap` and
  compatibility RBAC.

### 2.4 Credentials and external services

Always required:

- **Kafka access for the Fulfillment Service**, even when metering is
  disabled. Supply broker addresses, a SCRAM-SHA-512 username and password,
  certificate trust, and topic/group permissions. For an external broker,
  complete the [Kafka configuration guide](kafka-configuration.md) before
  installing OSAC. It includes a Strimzi example and the additional
  requirements for metering.
- **AAP subscription manifest (`license.zip`).** A Subscription Allocation
  export from the [Red Hat Customer Portal](https://access.redhat.com/). The AAP
  bootstrap job cannot start without it. You load it into the
  `config-as-code-manifest-ig` Secret in
  [Section 4.1](#41-preparing-the-cluster). For more information, see
  "Obtaining an AAP License" in
  [`osac-installer/README.md`](https://github.com/osac-project/osac/blob/main/osac-installer/README.md).

Required for a production deployment:

- **An external PostgreSQL 18 or later database.** The bundled PostgreSQL is
  for testing only and is not intended for production. For production, run
  your own database and create the `osac-db-config` and `osac-db-client-cert` Secrets in
  the install namespace, and the `osac-db-metering-config` and
  `osac-db-metering-client-cert` Secrets when metering is enabled. For more
  information, see
  [`fulfillment-service/docs/INSTALL.md`](https://github.com/osac-project/osac/blob/main/fulfillment-service/docs/INSTALL.md).
- **An external secret store.** The bundled OpenBao instance is for testing and
  CI, not production. Before installing the `osac` chart, prepare Vault or
  OpenBao, its Keycloak client, and the Kubernetes Secret and CA bundle used by
  OSAC. See the
  [secrets management configuration guide](secrets-management-configuration.md).

Required for CaaS:

- **DNS credentials.** Route 53 is the only DNS backend the chart schema
  accepts, so a Helm install requires an `AWS_ACCESS_KEY_ID` value and an
  `AWS_SECRET_ACCESS_KEY` value that can manage the target hosted zone, even
  in an on-premises environment. To run without OSAC managing DNS, see
  [Section 8.4](#84-installing-without-dns-management). For more information,
  see
  [`dns-backend.md`](https://github.com/osac-project/osac/blob/main/osac-installer/docs/dns-backend.md).
- **A `NetworkClass` with a registered manager.** OSAC provisions cluster
  networking through a `NetworkClass` custom resource, which requires a
  fabric manager, a Kubernetes manager, or both, to be registered and
  referenced (`networkClass.fabricManager` / `networkClass.k8sManager`). At
  this stage, CaaS assumes the cluster runs on servers that are part of a
  physical fabric managed by a fabric manager, such as Netris. Netris
  requires the controller URL, credentials, site and tenant IDs, and SSH keys
  to the servers and the bastion host. Other fabric and Kubernetes managers
  may be registered on your cluster; check with whoever manages your OSAC
  networking configuration for what's available. For more information, see
  [`network-backend.md`](https://github.com/osac-project/osac/blob/main/osac-installer/docs/network-backend.md).

Required for BMaaS with the Metal3 backend:

- **BareMetalOperator and a `Provisioning` custom resource.** When
  `bmf.metal3.enabled=true`, the pre-installation validation hook requires the
  `baremetalhosts.metal3.io` CRD and a `Provisioning` resource with
  `spec.watchAllNamespaces: true`. OSAC does not install BareMetalOperator.
- **BMC reachability.** Redfish or IPMI access from the cluster to each host
  BMC, plus DHCP and PXE for Ironic inspection.

### 2.5 Requirements by service

- **VMaaS** (`global.services.vmaas.enabled`) requires OpenShift Virtualization,
  LVM Storage or another dynamic storage class, and MetalLB or another
  `LoadBalancer`-class implementation.
- **CaaS** (`global.services.caas.enabled`) requires multicluster engine
  (standalone, or provided by RHACM if it's installed), a DNS backend, a
  network backend, and `aap.instanceGroups.clusterFulfillment` configuration.
  For more information,
  see
  [`aap-configuration.md`](https://github.com/osac-project/osac/blob/main/osac-installer/docs/aap-configuration.md).
- **BMaaS** (`global.services.bmaas.enabled`) requires BareMetalOperator and a
  `Provisioning` custom resource with `spec.watchAllNamespaces: true`.

---

## 3. OSAC component versions

OSAC subscribes to a specific update channel for each platform Operator. The
channel-tracked versions float as the channels publish updates.
[Table 3.1](#table-31-component-versions) lists each channel and the CSV version
that channel resolved to on OpenShift Container Platform 4.22.6 in September
2026. Treat the observed versions as indicative.

<a id="table-31-component-versions"></a>
**Table 3.1. Component versions**

| Component | Channel or fixed version | Observed on OCP 4.22.6 | Defined in |
|---|---|---|---|
| OpenShift Container Platform | `stable-4.22` | 4.22.4 to 4.22.6 | Validated range |
| `osac` umbrella chart | `0.0.17` released; `0.0.9-nightly.*` rolling | `0.0.17` | `oci://ghcr.io/osac-project/charts/osac` |
| cert-manager Operator for Red Hat OpenShift | `stable-v1` | `v1.20.0` | `charts/osac-deps/values.yaml` |
| Red Hat Ansible Automation Platform | `stable-2.6-cluster-scoped` | `v2.6.0` | `charts/osac-deps/values.yaml` |
| LVM Storage | `stable-<cluster_minor>` | `v4.22.0` | `charts/osac-deps/values.yaml` |
| MetalLB | `stable` | `v4.22.0` | `charts/osac-deps/values.yaml` |
| OpenShift Virtualization | `stable` | `v4.22.6` | `charts/osac-deps/values.yaml` |
| multicluster engine | `stable-2.17` | `v2.17.2` | `charts/osac-deps/values.yaml` |
| Streams for Apache Kafka | `stable` | `v3.2.1-10` | `charts/osac-deps/values.yaml` |
| trust-manager | `v0.20.0` (fixed) | `v0.20.0` | `charts/osac-infra/templates/trust-manager.yaml` |
| Envoy (Fulfillment Service sidecar) | `v1.33.0` (fixed) | `v1.33.0` | `values/*/instance.yaml` |
| OpenBao (bundled secret store) | `2.6.2` (fixed) | `2.6.2` | `charts/osac/values.yaml` |
| Keycloak (bundled) | Not applicable | `26.6.4` | `charts/osac-infra/` |
| OSAC web console chart | `0.0.6`, pinned at `HEAD` and in release `0.0.17` | Not applicable | `charts/osac/Chart.yaml` |
| PostgreSQL | 18 or later | `18` (Keycloak database) | `fulfillment-service/docs/INSTALL.md` |

The `osac` chart release `0.0.17` pins its subcharts to `osac-operator-crds`
`0.0.14`, `osac-operator` `0.0.14`, `fulfillment-service` `0.0.107`, `osac-aap`
`0.0.15`, `bare-metal-fulfillment-operator` `0.0.14`, and `osac-ui` `0.0.6`
through its `Chart.lock` file.

A tagged release pins every subchart and image to a specific version. Always
install from a tagged release for a real deployment.

---

## 4. Installing OSAC

Install OSAC with `oc` and Helm.

### 4.1 Preparing the cluster

**Prerequisites**

- You are logged in to the cluster as a user with `cluster-admin` privileges.
- The cluster pull secret is in place. See
  [Section 2.1](#21-cluster-and-access).
- You have the AAP subscription manifest file (`license.zip`).
- For a production deployment, your external PostgreSQL database is running and
  reachable from the cluster.
- For a production deployment, your external Vault-compatible store is running
  and reachable from the cluster. See the
  [secrets management configuration guide](secrets-management-configuration.md).

**Procedure**

Run all installation commands in the same shell session.

1. Set the shell variables that the rest of the installation uses:

   ```console
   $ export NS=<namespace>
   $ export DOMAIN=$(oc get ingresses.config/cluster -o jsonpath='{.spec.domain}')
   $ export OCP_VERSION=$(oc get clusterversion version -o jsonpath='{.status.desired.version}' | cut -d. -f1,2)
   $ export AAP_LICENSE_FILE=/path/to/license.zip
   ```

2. Create the target namespace:

   ```console
   $ oc create namespace "$NS" --dry-run=client -o yaml | oc apply -f -
   ```

3. Create the Secret that holds the AAP subscription manifest:

   ```console
   $ oc create secret generic config-as-code-manifest-ig --from-file=license.zip="$AAP_LICENSE_FILE" -n "$NS" --dry-run=client -o yaml | oc apply --server-side -f -
   ```

4. Label the Secret so that the AAP bootstrap job reads it:

   ```console
   $ oc label secret config-as-code-manifest-ig osac.openshift.io/project=osac-aap -n "$NS" --overwrite
   ```

5. For a production deployment, create the database Secrets in `$NS`:
   `osac-db-config` and `osac-db-client-cert`, and, when metering is enabled,
   `osac-db-metering-config` and `osac-db-metering-client-cert`. For more
   information, see
   [`fulfillment-service/docs/INSTALL.md`](https://github.com/osac-project/osac/blob/main/fulfillment-service/docs/INSTALL.md).

   > **Note**
   >
   > The chart pre-installation hook fails if the connection URL that these
   > Secrets carry does not resolve to a ready PostgreSQL Service.

6. For a production deployment, follow the
   [secrets management configuration guide](secrets-management-configuration.md)
   to prepare the Vault configuration.

### 4.2 Configuring the Helm values

You create one values file for the `osac` chart, `my-values.yaml`.

**Procedure**

1. Retrieve the full set of value keys from the chart:

   ```console
   $ helm show values oci://ghcr.io/osac-project/charts/osac --version 0.0.17 > values-upstream.yaml
   ```

2. Create `my-values.yaml` for the `osac` chart. Base it on the Production
   block of the chart `values-example.yaml` file, which uses an external
   PostgreSQL database, an external Keycloak, and pinned image tags.

3. Add the following hardening to `my-values.yaml`:
   ```yaml
   keycloak:
     devFixtures:
       enabled: false
     adminUsername: <admin_user>
     adminPassword: <strong_password>
   bundledPostgres:
     enabled: false
   bundledVault:
     enabled: false
   ```

4. For a production deployment, add `service.vault` to `my-values.yaml` using
   the [secrets management configuration guide](secrets-management-configuration.md#configure-the-osac-instance).

5. Add the service-specific value blocks (`global.services.*`,
   `operator.networkManagers`, `networkClass`, `aap`, `metering`, and `bmf`)
   from [Section 6](#6-installation-workflows-by-service) for the service you
   are installing.

For a full parameter reference, see
[Section 5](#5-helm-chart-configuration-parameters).

### 4.3 Installing OSAC

Use this procedure when the cluster already meets the prerequisites.

**Prerequisites**

- The prerequisite Operators from
  [Table 2.1](#table-21-platform-operators-and-components) are installed.
- The `default-ca` `ClusterIssuer`, trust-manager, the `ca-bundle` `ConfigMap`,
  Keycloak with the `osac` realm, and the credential Secrets exist.
- For a production deployment, the external PostgreSQL database and its
  `osac-db-*` Secrets exist.
- For a production deployment, the external Vault-compatible is configured
  according to the [secrets management configuration guide](secrets-management-configuration.md).
- You completed [Section 4.1](#41-preparing-the-cluster) and
  [Section 4.2](#42-configuring-the-helm-values).

**Procedure**

- Install the `osac` chart by running the following command:

  ```console
  $ helm install osac oci://ghcr.io/osac-project/charts/osac --version 0.0.17 \
      -n "$NS" --create-namespace \
      -f my-values.yaml \
      --set global.clusterDomain="$DOMAIN" \
      --set service.externalHostname="fulfillment-api-$NS.$DOMAIN" \
      --set service.internalHostname="fulfillment-internal-api-$NS.$DOMAIN" \
      --wait --timeout 40m
  ```

  Use `--version 0.0.9-nightly.<build>` only to test unreleased fixes. The AAP
  bootstrap job takes 10 to 40 minutes. Helm does not return until it and, for
  CaaS, the `osac-publish-templates` hook have finished.

  On Helm 4, later `helm upgrade` (or re-running `helm upgrade --install`)
  must include `--force-conflicts`. The AAP operator takes field ownership of
  `app.kubernetes.io/managed-by` on the `osac-aap` custom resource, and Helm
  4 server-side apply fails without that flag. Helm 3 does not accept it.

**Verification**

- Complete [Section 7](#7-verifying-the-installation).

---

## 5. Helm chart configuration parameters

This section lists the parameters you are most likely to set. To retrieve the
complete set, run the following command, and review the `osac` chart
`values.schema.json` file:

```console
$ helm show values oci://ghcr.io/osac-project/charts/osac --version 0.0.17
```

Keys defined in `charts/osac/values.schema.json` are marked `schema`. Keys
marked `subchart` aren't in the umbrella schema; confirm those in the
subchart's own `values.yaml` file.

<a id="table-51-service-enablement-my-valuesyaml"></a>
**Table 5.1. Service enablement (`my-values.yaml`)**

| Parameter | Source | Description | Default |
|---|---|---|---|
| `global.clusterDomain` | schema | Apps domain. All route host names and the default issuer, IdP, and Vault URLs derive from it. Set at installation. | `""` |
| `global.osacDeploymentId` | schema | Required when `metering.enabled` is `true`. A stable identity for this deployment, stored in a retained `ConfigMap`. The chart fails to render without it when metering is on, and fails on upgrade if the value changes — never change it after the first successful install. | Not set |
| `global.services.vmaas.enabled` | schema | Enables the VMaaS tier. | `true` |
| `global.services.caas.enabled` | schema | Enables the CaaS tier. | `true` |
| `global.services.bmaas.enabled` | schema | Enables the BMaaS tier and gates the `bmf` subchart. | `true` |
| `global.services.maas.enabled` | schema | Enables the MaaS tier. | `true` |

**Table 5.2. OSAC Operator (`operator.*`)**

| Parameter | Source | Description |
|---|---|---|
| `operator.image.repository`, `operator.image.tag`, `operator.image.pullPolicy` | schema | Operator image. Use a release tag for production. |
| `operator.replicaCount`, `operator.resources.*` | schema | Operator sizing. |
| `operator.aap.url`, `operator.aap.token`, `operator.aap.insecureSkipVerify`, `operator.aap.statusPollInterval`, `operator.aap.templatePrefix` | schema | How the Operator reaches AAP. Set `insecureSkipVerify: "true"` for self-signed AAP routes. |
| `operator.fulfillment.serverAddress` | schema | Fulfillment gRPC endpoint used by the Operator. |
| `operator.fulfillment.credentials.secretName`, `clientIdKey`, `clientSecretKey` | schema | Name and keys of the Kubernetes Secret holding the `osac-controller` Keycloak service-account credentials. `osac-infra` provisions it for bundled Keycloak; external Keycloak deployments must provide an equivalent Secret. The OSAC chart only references it. |
| `operator.controllers.tenant`, `operator.controllers.networking`, `operator.controllers.storage` | schema | Enable or disable individual reconcilers. Which services run is driven by `global.services.*`. |
| `operator.controllers.networkingProvisioning` | schema | When `false`, networking custom resources reconcile to `READY` without a real fabric. |
| `operator.controllers.volume` | subchart | Enables the Volume reconciler. |
| `operator.configSecret.name`, `operator.configSecret.optional` | schema | Additional Operator configuration Secret. |
| `operator.hubAccess.enabled` | schema | Creates the hub-access `ClusterRole` that the Fulfillment Service binds to. |
| `operator.stall.preparingInfrastructureThreshold`, `operator.stall.controlPlaneStartingThreshold`, `operator.stall.workersJoiningThreshold` | schema | How long a cluster provisioning phase can wait before the Operator marks it stalled. |
| `operator.tenants[]` | subchart | Tenants pre-created at installation. `shared` is the built-in tenant. |
| `operator.networkManagers.fabricManagers.<name>.enabled`, `operator.networkManagers.k8sManagers.<name>.enabled` | subchart | Enables a fabric manager (`netris`, `cudn_net`) or a Kubernetes manager (`k8s_only`). |

**Table 5.3. Networking (umbrella level)**

| Parameter | Source | Description | Default |
|---|---|---|---|
| `networkManagers[]` | schema | Registers network-manager `ConfigMap` resources that the Operator dispatcher discovers. Each entry has `name`, `role` (`fabric` or `k8s`), `description`, and `capabilities`. Equivalent to `operator.networkManagers.*`. | `[]` |
| `networkClass.enabled` | schema | Creates the default `NetworkClass` in the Fulfillment Service after installation. Required for VirtualNetwork and tenant onboarding. | `true` |
| `networkClass.title`, `networkClass.description` | schema | Required title and description for the `NetworkClass`. | CUDN text |
| `networkClass.fabricManager`, `networkClass.k8sManager` | schema | Which registered manager backs the class. For fabric-less VMaaS, set `fabricManager: ""` and `k8sManager: "k8s_only"`. For Netris, set `fabricManager: "netris"`. | `cudn_net` and `""` |
| `networkClass.isDefault` | schema | Marks the class the deployment default. Only one `NetworkClass` can exist. | `true` |
| `networkClass.defaults.virtualNetworkIPv4CIDR`, `networkClass.defaults.subnetIPv4CIDR`, `networkClass.defaults.enableNatGateway`, `networkClass.defaults.egressRules` | schema | Tenant-onboarding defaults that auto-create the VirtualNetwork, Subnet, and SecurityGroup. | `10.200.0.0/16` and others |

**Table 5.4. Fulfillment Service (`service.*`, all `schema`)**

| Parameter | Description |
|---|---|
| `service.externalHostname`, `service.internalHostname` | Public and internal API route host names. Set at installation. |
| `service.variant` | `openshift` or `kind`. Use `openshift`. |
| `service.images.service`, `service.images.envoy` | Fulfillment Service and Envoy sidecar images. |
| `service.certs.issuerRef.kind`, `service.certs.issuerRef.name` | cert-manager issuer for the service certificates. |
| `service.certs.caBundle.configMap` | `ConfigMap` that holds the CA bundle. |
| `service.auth.issuerUrl` | OIDC issuer. Templated from `global.clusterDomain` by default. |
| `service.auth.controllerCredentials[]` | Volume sources for the controller OIDC client credentials. |
| `service.idp.provider`, `service.idp.url`, `service.idp.credentials[]` | Identity provider for user and role management. Only `keycloak` is supported. |
| `service.database.connection[]` | Volume sources for the database URL and client certificate, for example `osac-db-config` and `osac-db-client-cert`. |
| `service.log.level`, `service.log.headers`, `service.log.bodies` | Log verbosity. Keep `headers` and `bodies` set to `false` in production. |
| `service.vault.endpoint` | Vault-compatible API URL. Defaults to the bundled OpenBao service in `osac-infra`. Set it to your external Vault or OpenBao URL when `bundledVault.enabled` is `false`; the Fulfillment chart requires a nonempty endpoint. See the [secrets management configuration guide](secrets-management-configuration.md). |
| `service.vault.namespace`, `service.vault.kvMountPath`, `service.vault.lifecycleRole`, `service.vault.lifecycleMountPath`, `service.vault.keycloakClientId`, `service.vault.keycloakIssuerUrl`, `service.vault.keycloakAudience`, `service.vault.caBundle`, `service.vault.credentials` | Vault namespace, mount paths, JWT role, Keycloak client and audience, CA bundle, and client credentials. Configure these for an external store as described in the [secrets management configuration guide](secrets-management-configuration.md). |

**Table 5.5. AAP (`aap.*`, all `schema`)**

| Parameter | Description |
|---|---|
| `aap.aap.instance.enabled`, `aap.aap.instance.name` | Create the AAP instance custom resource. Set `enabled: false` when AAP is managed externally. |
| `aap.aap.instance.controller.disabled`, `aap.aap.instance.hub.disabled`, `aap.aap.instance.lightspeed.disabled` | Turn off individual AAP components. |
| `aap.aap.instance.redisMode` | `standalone` or `cluster`. |
| `aap.aap.instance.routeTlsTerminationMechanism` | `Edge`, `Passthrough`, or `Reencrypt` for the AAP routes. |
| `aap.bootstrap.enabled`, `aap.bootstrap.image`, `aap.bootstrap.eeImage`, `aap.bootstrap.backoffLimit` | The postinstallation bootstrap job that loads config-as-code. |
| `aap.configAsCode.manifestSecret` | Secret that holds `license.zip`. |
| `aap.configAsCode.secret` | Secret with config-as-code runtime flags. |
| `aap.configAsCode.projectGitUri`, `aap.configAsCode.projectGitBranch` | Git source for the Ansible content that the bootstrap job imports. |
| `aap.configAsCode.importAgentsEnabled`, `aap.configAsCode.importBcmAgentsEnabled` | Enable bare-metal agent import and the BCM inventory backend. |
| `aap.instanceGroups.clusterFulfillment.enabled`, `aap.instanceGroups.clusterFulfillment.config`, `aap.instanceGroups.clusterFulfillment.secret` | The `cluster-fulfillment` instance group for CaaS provisioning. See [Section 6.2](#62-installing-osac-for-caas-with-the-netris-network-backend). |
| `aap.instanceGroups.networkFulfillment.enabled`, `aap.instanceGroups.networkFulfillment.config`, `aap.instanceGroups.networkFulfillment.secret` | The `network-fulfillment` instance group for Netris. See [Section 6.2](#62-installing-osac-for-caas-with-the-netris-network-backend). |
| `aap.instanceGroups.storageFulfillment.config.STORAGE_SNAPSHOTS_ENABLED` | Snapshot toggle for the `storage-operations` instance group. |
| `aap.instanceGroups.publishTemplates.enabled` | Runs the postinstallation `osac-publish-templates` hook. Default `true`. Set it to `false` for VMaaS-only or BMaaS-only installations. |
| `aap.instanceGroups.publishTemplates.config.OSAC_TEMPLATE_COLLECTIONS`, `aap.instanceGroups.publishTemplates.config.OSAC_FULFILLMENT_SERVICE_URI` | The Ansible collections to publish and the internal Fulfillment Service URI. |

**Table 5.6. Web console, BMaaS, and other parameters**

| Parameter | Source | Description | Default |
|---|---|---|---|
| `ui.enabled` | schema | Deploys the OSAC web console. | `true` |
| `ui.externalHostname` | schema | Route host name. Auto-assigned from `global.clusterDomain` if empty. | `""` |
| `ui.api.fulfillment.url` | schema | Fulfillment API that the web console calls. | Internal URL |
| `ui.auth.oidcClientId` | schema | OIDC client ID for the web console. | `osac-ui` |
| `ui.images.ui` | schema | Web console image. | Release tag |
| `bmf.metal3.enabled`, `bmf.metal3.namespace`, `bmf.metal3.hostClass` | schema | Use the Metal3 backend. Set `namespace` to where your `BareMetalHost` resources live. Requires BareMetalOperator and a `Provisioning` custom resource with `spec.watchAllNamespaces: true`. | `false` |
| `bmf.env.aapInsecureSkipVerify` | schema | Skip TLS verification for AAP API calls. | `"true"` |
| `bmf.env.aapUrl`, `bmf.env.enableNetworkingProvisioning` | subchart | AAP controller URL that the Bare Metal Fulfillment Operator drives, and its networking-provisioning toggle. | Not applicable |
| `bmf.bcm.enabled`, `bmf.bcm.url`, `bmf.bcm.cert`, `bmf.bcm.key`, `bmf.bcm.caCert`, `bmf.bcm.insecureSkipVerify`, `bmf.bcm.hostClass`, `bmf.bcm.bmhNamespace` | subchart | Base Command Manager (BCM) backend. | `false` |
| `bmf.secrets.inventoryConfig`, `bmf.secrets.managementConfig`, `bmf.secrets.osClouds`, `bmf.configMaps.profiles` | schema | Names of the inventory, management, and `clouds.yaml` Secrets and the profiles `ConfigMap`. | Default names |
| `operatorCrds.install` | schema | Install the OSAC CRDs. Set it to `false` if a cluster administrator manages them. | `true` |
| `kafka.enabled` | schema | Creates the Fulfillment Service's Strimzi user and copies its credentials. Set to `false` for an external Kafka connection; this does not disable Kafka use or metering's Strimzi resources. See [Kafka configuration](kafka-configuration.md). | `true` |
| `service.kafka.connection` | schema | ConfigMap/Secret mappings for the required `brokers`, `user`, and `password` parameters. Supply resources in the OSAC namespace. | `fulfillment-service-kafka` Secret |
| `metering.enabled` | schema | Deploys the metering service. Requires Strimzi-managed Kafka, a database connection, and `global.osacDeploymentId` (see [Table 5.1](#table-51-service-enablement-my-valuesyaml) and [Kafka configuration](kafka-configuration.md#optional-metering)). | `false` |
| `metering.reconciliation.interval`, `metering.m360Adapter.enabled`, `metering.m360Adapter.m360.apiUrl`, `metering.m360Adapter.apiKeySecret` | subchart | Reconcile period and the Monetize360 billing adapter. | Not applicable |
| `validation.enabled` | schema | Runs the pre-installation validation hook. | `true` |
| `metallb.enabled`, `metallb.addressCIDR` | schema | In the `osac` chart, creates an `IPAddressPool` and an `L2Advertisement`. Edit the pool after installation to match your network. | `false` and `192.168.40.0/24` |
| `bundledVault.enabled`, `bundledVault.image` | schema | In-cluster OpenBao for testing and CI. Not intended for production. | `true` and `openbao:2.6.2` |
| `hubAccess.enabled` | schema | Creates hub-access RBAC and registers the local cluster as its own hub. Single-cluster development only. | `false` |
| `bundledPostgres.enabled` | schema | Bundled PostgreSQL for testing only; not intended for production. | `false` |
| `dbInit.host` | schema | Host that the `db-init` pre-installation hook connects to, to create the databases. Set it to your external PostgreSQL host for a production deployment. | `postgres.osac-infra.svc.cluster.local` |
| `clusterVersions.enabled`, `clusterVersions.versions[]` | schema | OpenShift Container Platform release images offered to hosted clusters. Each entry has `version`, `image`, and an optional `default`. | `false` and `[]` |

### 5.1 Values set on the command line

The installation command computes and passes the values in
[Table 5.7](#table-57-command-line-values). Add your own with `-f` files or
extra `--set` flags.

<a id="table-57-command-line-values"></a>
**Table 5.7. Command-line values**

| Flag | Value | Purpose |
|---|---|---|
| `--set global.clusterDomain=$DOMAIN` | `oc get ingresses.config/cluster -o jsonpath='{.spec.domain}'` | Route host names and the issuer and IdP URLs. |
| `--set service.externalHostname=fulfillment-api-$NS.$DOMAIN` | Derived | Public API route. |
| `--set service.internalHostname=fulfillment-internal-api-$NS.$DOMAIN` | Derived | Internal API route. |

---

## 6. Installation workflows by service

Each workflow gives the phase-2 (`my-values.yaml`) values to add for that
service. It assumes the phase-1 prerequisites for that service (see
[Table 2.1](#table-21-platform-operators-and-components)) already exist on
the cluster. Then follow [Section 4](#4-installing-osac).

### 6.1 Installing OSAC for VMaaS

**Prerequisites**

- OpenShift Virtualization is installed.
- LVM Storage is installed, or another dynamic storage class exists.
- MetalLB is installed, or another `LoadBalancer`-class implementation exists.

**Procedure**

1. In `my-values.yaml`, enable the VMaaS tier and configure fabric-less
   networking. Fabric-less networking creates the VirtualNetwork and Subnet by
   using ClusterUserDefinedNetwork (CUDN), the SecurityGroup by using a
   `NetworkPolicy`, and the ExternalIP by using MetalLB L2:

   The OSAC chart does not deploy the OSAC CSI routing meta-driver. Provider CSI
   setup and tenant storage provisioning are handled separately by the storage
   workflow.

   ```yaml
   global:
     services: { vmaas: { enabled: true }, caas: { enabled: false }, bmaas: { enabled: false }, maas: { enabled: false } }
   lvms:
     enabled: true
   operator:
     networkManagers:
       k8sManagers:
         k8s_only:
           enabled: true
   networkClass:
     fabricManager: ""
     k8sManager: "k8s_only"
   aap:
     instanceGroups:
       publishTemplates:
         enabled: false
   ```

   `lvms: { enabled: true }` runs the `osac` chart's `register-local-storage`
   hook, which creates the `local` `StorageBackend` and `StorageTier` that the
   first `ComputeInstance` needs. Without it, provisioning fails with an empty
   `storage_tier_definitions` even though every pod looks healthy.

2. Add the `service.*`, `aap.configAsCode.*`, Keycloak hardening, and database
   Secret settings from [Section 4.2](#42-configuring-the-helm-values).

3. Install OSAC. See [Section 4.3](#43-installing-osac).

**Verification**

- Complete [Section 7](#7-verifying-the-installation).
- Create a `ComputeInstance` custom resource and confirm that it reaches
  `RUNNING`:

  ```console
  $ oc get computeinstance -A
  ```

### 6.2 Installing OSAC for CaaS with the Netris network backend

> **TBD**
>
> `network-backend.md` now documents a `global.networking` facade
> (`fabricManager`/`k8sManager`) that Helm derives the AAP instance-group
> config from, and a supported `agentless_net` backend this section doesn't
> cover. This section still uses the older, direct
> `aap.instanceGroups.clusterFulfillment.config` style. Pending confirmation
> from the networking team (see the open PR discussion) before rewriting it
> around the facade.

The AWS Route 53 DNS backend is the default; it's independent of the network
backend. For the full variable reference, see
[`network-backend.md`](https://github.com/osac-project/osac/blob/main/osac-installer/docs/network-backend.md).

**Prerequisites**

- multicluster engine is installed (standalone, or provided by RHACM).
- MetalLB (or another `LoadBalancer`-class implementation) and LVM Storage
  (or another dynamic storage class) are installed.
- You have AWS Route 53 credentials for the target hosted zone.
- Access to the Netris controller: URL, user name, password, site and tenant
  IDs, and management VPC details.
- SSH private keys for the servers and the bastion host.

**Procedure**

1. In `my-values.yaml`, enable the CaaS tier, the OpenShift Container Platform
   release images offered to hosted clusters, and the `cluster-fulfillment`
   and `network-fulfillment` instance groups with the Netris coordinates:

   ```yaml
   global:
     services: { caas: { enabled: true }, vmaas: { enabled: false }, bmaas: { enabled: false }, maas: { enabled: false } }
   clusterVersions:
     enabled: true
     versions:
       - version: "4.22.0"
         image: "quay.io/openshift-release-dev/ocp-release:4.22.0-multi"
         default: true
   aap:
     instanceGroups:
       publishTemplates:
         enabled: true
       clusterFulfillment:
         enabled: true
         config:
           NETWORK_CLASS: "netris"
           NETWORK_STEPS_COLLECTION: "netris.steps"
           DNS_CLASS: "dns.route53.dns"
           NETRIS_CONTROLLER_URL: "https://netris.example.com"
           NETRIS_USERNAME: "netris"
           NETRIS_SITE_ID: "5"
           NETRIS_TENANT_ID: "1"
           NETRIS_TENANT_NAME: "Admin"
           NETRIS_MGMT_VPC_ID: "4"
           NETRIS_MGMT_VPC_NAME: "RH-Infra"
           NETRIS_RESOURCE_CLASS_MAP: '{"fc430":{"server_cluster_template_id":89,"mgmt_interface":"ens4","vpc_interfaces":["ens13"]}}'
           SERVER_SSH_BASTION_HOST: "bastion.example.com"
           SERVER_SSH_BASTION_USER: "ubuntu"
           SERVER_SSH_USER: "core"
           SERVER_MGMT_ROUTE_DESTINATION: "198.51.100.0/30"
           SERVER_MGMT_ROUTE_GATEWAY: "192.0.2.1"
           EXTERNAL_ACCESS_BASE_DOMAIN: "clusters.example.com"
           EXTERNAL_ACCESS_SUPPORTED_BASE_DOMAINS: "clusters.example.com"
           EXTERNAL_ACCESS_API_INTERNAL_NETWORK: "hypershift"
           HOSTED_CLUSTER_BASE_DOMAIN: "clusters.example.com"
           HOSTED_CLUSTER_CONTROLLER_AVAILABILITY_POLICY: "HighlyAvailable"
           HOSTED_CLUSTER_INFRASTRUCTURE_AVAILABILITY_POLICY: "HighlyAvailable"
       networkFulfillment:
         enabled: true
         config:
           NETRIS_CONTROLLER_URL: "https://netris.example.com"
           NETRIS_USERNAME: "netris"
           NETRIS_SITE_ID: "5"
           NETRIS_TENANT_ID: "1"
           NETRIS_TENANT_NAME: "Admin"
   ```

   In `NETRIS_RESOURCE_CLASS_MAP`, each key is a resource-class name;
   `server_cluster_template_id` is the Netris server-cluster template,
   `mgmt_interface` is the management NIC, and `vpc_interfaces` are the
   data-plane NICs.

2. Put the AWS and Netris credentials in a separate values file that is
   excluded from version control:

   ```yaml
   aap:
     instanceGroups:
       clusterFulfillment:
         secret:
           NETRIS_PASSWORD: "<netris_password>"
           AWS_ACCESS_KEY_ID: "<route53_access_key_id>"
           AWS_SECRET_ACCESS_KEY: "<route53_secret_access_key>"
           SERVER_SSH_KEY: |
             <contents_of_your_openssh_private_key_file>
           SERVER_SSH_BASTION_KEY: |
             <contents_of_your_openssh_bastion_private_key_file>
       networkFulfillment:
         secret:
           NETRIS_PASSWORD: "<netris_password>"
   ```

3. Register the `netris` fabric manager and point the default `NetworkClass` at
   it:

   ```yaml
   networkManagers:
     - name: netris
       role: fabric
       description: "Netris SDN fabric manager"
       capabilities: "ipv4"
   networkClass:
     fabricManager: "netris"
     k8sManager: ""
   ```

4. Install OSAC (see [Section 4.3](#43-installing-osac)), passing every
   values file, for example `helm ... -f my-values.yaml -f my-secrets.local.yaml
   ...`. For more information, see
   [`aap-configuration.md`](https://github.com/osac-project/osac/blob/main/osac-installer/docs/aap-configuration.md)
   and
   [`dns-backend.md`](https://github.com/osac-project/osac/blob/main/osac-installer/docs/dns-backend.md).

**Verification**

- Complete [Section 7](#7-verifying-the-installation), including step 10.
- Create a `ClusterOrder` custom resource and watch the AAP
  `cluster-fulfillment` and `network-fulfillment` jobs.

### 6.3 Installing OSAC for BMaaS

**Prerequisites**

- BareMetalOperator is installed, with a `Provisioning` custom resource that has
  `spec.watchAllNamespaces: true`. OSAC does not install these; the
  pre-installation validation hook fails if they are missing.

**Procedure**

1. In `my-values.yaml`, enable the BMaaS tier and the Metal3 backend:

   ```yaml
   global:
     services: { bmaas: { enabled: true }, vmaas: { enabled: false }, caas: { enabled: false }, maas: { enabled: false } }
   operator:
     controllers:
       networkingProvisioning: false
   bmf:
     metal3:
       enabled: true
       namespace: host-inventory
       hostClass: metal3
     env:
       aapUrl: "http://osac-aap/api/controller"
       aapInsecureSkipVerify: "true"
       enableNetworkingProvisioning: "false"
   aap:
     instanceGroups:
       publishTemplates:
         enabled: false
   ```

   Set `bmf.metal3.namespace` to the namespace where your `BareMetalHost`
   resources live.

2. Install OSAC. See [Section 4.3](#43-installing-osac).

**Verification**

- Complete [Section 7](#7-verifying-the-installation).
- Confirm that Metal3 is ready:

  ```console
  $ oc get provisioning
  $ oc get baremetalhosts -A
  ```

- Create a `BareMetalPool` or `BareMetalInstance` custom resource and confirm
  that the Bare Metal Fulfillment Operator reconciles it.

### 6.4 Manual NetApp ONTAP and Trident integration for VMaaS

> **Manual procedure.** The `osac` release does not enable this path.
>
> This procedure adds a NetApp ONTAP NFS tier to an existing VMaaS
> installation. The `csi-backends` chart supplies the Trident controller when
> that chart is rendered on its own. The chart does not install the Trident
> persistence CRDs or grant the SCCs, and the OSAC node DaemonSet cannot
> mount NFS. Setting `provider: ontap` does not enable the controller. The
> AAP collection has no ONTAP storage role, so the tenant StorageClass is
> created by hand and the first NetApp VM job is relaunched.

This procedure was tested with OpenShift 4.22, ONTAP NFS, and Trident
25.10.0. NetApp's Trident 25.10 requirements list OpenShift 4.12 and
4.14–4.20, so that combination is outside the versions NetApp documents.
Before you use this integration, check the current
[Trident requirements](https://docs.netapp.com/us-en/trident-2510/trident-get-started/requirements.html)
and confirm support for your OpenShift and Trident versions. See also NetApp's
[OpenShift installation guide](https://docs.netapp.com/us-en/trident-2510/trident-install/openshift-operator-deploy.html),
[ONTAP NAS backend preparation](https://docs.netapp.com/us-en/trident-2510/trident-use/ontap-nas-prep.html),
and [backend examples](https://docs.netapp.com/us-en/trident-2510/trident-use/ontap-nas-examples.html).
Do not install a second standalone Trident controller from OperatorHub, and
do not `helm upgrade` the `osac` release to enable Trident. Render only the
`csi-backends` chart, as in step 4.

The OSAC `StorageBackend` and the Trident backend are different objects.
Create them in the order below.

| Object | Role | When |
| --- | --- | --- |
| `StorageBackend` `provider: ontap` | Registers the array in OSAC. Does not install Trident or create a volume. | First, on the private API |
| `StorageTier` `ontap-nas` | NFS tier that a tenant selects on a VM | With the StorageBackend |
| Secret `trident-csi` and the Trident CRDs | TLS for the controller, and the persistence CRDs the chart does not install | Before the controller |
| Deployment `trident-csi-controller` | OSAC vendor controller from the `csi-backends` chart | After the Secret and CRDs |
| DaemonSet `trident-node-linux` | Official Trident node manifest, patched so it can mount NFS | After the controller is Ready |
| Trident backend `ontap-nas-economy` | Tells Trident which SVM and data LIF to use | After every worker mounts the export |
| Tenant and `tenant-admin` | Break-glass user created with the tenant. VMs require `tenant-admin`. | Before the StorageClass |
| StorageClass `ontap-nas-${TENANT}` | Class for that tenant's PVCs | After the Trident backend is online |

**Prerequisites**

- Complete [Section 6.1](#61-installing-osac-for-vmaas) and verify that a VM
  can use the existing `local` storage tier. Keep that tier. Add the NetApp
  tier alongside it.
- OpenShift Virtualization, OSAC VMaaS, the OSAC CSI driver, and AAP are
  installed. A cluster administrator can create namespaces, Secrets, CRDs,
  and SCCs.
- The storage administrator provides an ONTAP SVM management endpoint, the
  SVM name, an NFS data LIF, a dedicated export and capacity, and an account
  with the Trident permissions NetApp requires. Do not reuse an export or LUN
  that belongs to another workload.
- Keep ONTAP credentials in the approved secret manager or a protected shell
  environment. Do not put credentials, internal addresses, or access details
  in values files, source control, or this guide.
- The examples use `OSAC_DIR` (the `osac` checkout), `OSAC_NAMESPACE`, `TENANT`, `ONTAP_MANAGEMENT_ENDPOINT`
  (bare host, no `https://` prefix), `ONTAP_SVM`, `NFS_DATA_LIF`,
  `NFS_EXPORT_PATH`, `ONTAP_USERNAME`, `ONTAP_PASSWORD`, and
  `GUEST_PASSWORD`.

**Procedure**

1. Register the array in OSAC. Sign in on the private administrator API.
   Trust the route with the CA file from step 11 of
   [Section 7](#7-verifying-the-installation). A tenant session cannot create
   a `StorageBackend`. This step does not install Trident.

   ```bash
   INT_ROUTE=$(oc get route fulfillment-internal-api -n "$OSAC_NAMESPACE" \
     -o jsonpath='{.spec.host}')
   osac login --address "$INT_ROUTE" --private \
     --ca-file default-ca.crt \
     --token-script "oc create token -n $OSAC_NAMESPACE admin"
   osac create -f - <<EOF
   "@type": type.googleapis.com/osac.private.v1.StorageBackend
   metadata:
     name: ontap-lab
   spec:
     provider: ontap
     endpoint: "${ONTAP_MANAGEMENT_ENDPOINT}"
     credentials:
       username: "${ONTAP_USERNAME}"
       password: "${ONTAP_PASSWORD}"
   EOF
   BACKEND_ID=$(osac get storagebackend ontap-lab -o json \
     | jq -r '.id // .metadata.id // .spec.id')
   osac create storagetier \
     --name ontap-nas \
     --backend-id "$BACKEND_ID" \
     --protocol NFS
   osac get storagetier
   ```

   Pass a bare host in `endpoint`. `provider: ontap` is the CSI routing key.
   Do not print the credential variables, and do not replace them with
   literal values in the command. Expect tier `ontap-nas` with protocol
   `NFS`, and leave the `local` tier in place. This procedure covers NFS. It
   does not cover block, iSCSI, or FC storage. Setting `provider: ontap` does
   not enable the Trident controller. `provider` cannot be changed after
   create. If it is wrong, delete that `StorageBackend` and create it again.
   `StorageBackendReady` on the tenant can stay False. That does not remove
   the `local` tier.
2. Create the `osac-csi-backends` namespace, copy the cluster pull secret
   into it, and create the `trident-csi` TLS Secret. The controller image
   `registry.connect.redhat.com/netapp/trident:25.10.0` uses that pull secret.
   The Secret keys are the ones listed in
   [`csi-backends/README.md`](https://github.com/osac-project/osac/blob/main/osac-csi-driver/charts/csi-backends/README.md).
   Sign each certificate with its own serial. A second `-CAcreateserial` in
   the same directory skips the client certificate. Give the server
   certificate the SAN `DNS:trident-csi`. The client certificate common name
   must be `trident-node`. The controller accepts that name on its HTTPS REST
   port.

   ```bash
   oc create namespace osac-csi-backends --dry-run=client -o yaml | oc apply -f -
   oc get secret pull-secret -n openshift-config -o json \
     | jq 'del(.metadata.uid,.metadata.resourceVersion,.metadata.creationTimestamp,.metadata.namespace,.metadata.ownerReferences)
           | .metadata.name="pull-secret"
           | .metadata.namespace="osac-csi-backends"' \
     | oc apply -f -
   oc -n osac-csi-backends delete secret trident-csi --ignore-not-found
   TRIDENT_CERT_DIR=$(mktemp -d)
   openssl genrsa -out "$TRIDENT_CERT_DIR/caKey" 2048
   openssl req -new -x509 -key "$TRIDENT_CERT_DIR/caKey" -sha256 -days 3650 \
     -subj '/CN=trident-ca' -out "$TRIDENT_CERT_DIR/caCert"
   openssl genrsa -out "$TRIDENT_CERT_DIR/serverKey" 2048
   openssl req -new -key "$TRIDENT_CERT_DIR/serverKey" -subj '/CN=trident-csi-controller' \
     -out "$TRIDENT_CERT_DIR/server.csr"
   printf 'subjectAltName=DNS:trident-csi\n' > "$TRIDENT_CERT_DIR/server.ext"
   openssl x509 -req -in "$TRIDENT_CERT_DIR/server.csr" -CA "$TRIDENT_CERT_DIR/caCert" \
     -CAkey "$TRIDENT_CERT_DIR/caKey" -set_serial 1 -out "$TRIDENT_CERT_DIR/serverCert" \
     -days 3650 -sha256 -extfile "$TRIDENT_CERT_DIR/server.ext"
   openssl genrsa -out "$TRIDENT_CERT_DIR/clientKey" 2048
   openssl req -new -key "$TRIDENT_CERT_DIR/clientKey" -subj '/CN=trident-node' \
     -out "$TRIDENT_CERT_DIR/client.csr"
   openssl x509 -req -in "$TRIDENT_CERT_DIR/client.csr" -CA "$TRIDENT_CERT_DIR/caCert" \
     -CAkey "$TRIDENT_CERT_DIR/caKey" -set_serial 2 -out "$TRIDENT_CERT_DIR/clientCert" \
     -days 3650 -sha256
   openssl rand -hex 16 > "$TRIDENT_CERT_DIR/aesKey"
   oc -n osac-csi-backends create secret generic trident-csi \
     --from-file=caKey="$TRIDENT_CERT_DIR/caKey" \
     --from-file=caCert="$TRIDENT_CERT_DIR/caCert" \
     --from-file=serverKey="$TRIDENT_CERT_DIR/serverKey" \
     --from-file=serverCert="$TRIDENT_CERT_DIR/serverCert" \
     --from-file=clientKey="$TRIDENT_CERT_DIR/clientKey" \
     --from-file=clientCert="$TRIDENT_CERT_DIR/clientCert" \
     --from-file=aesKey="$TRIDENT_CERT_DIR/aesKey"
   rm -rf "$TRIDENT_CERT_DIR"
   ```
3. Install the Trident persistence CRDs from the official installer package.
   Do not run its controller installation.

   ```bash
   TRIDENT_VERSION=25.10.0
   curl -fsSL -o "/tmp/trident-installer-${TRIDENT_VERSION}.tar.gz" \
     "https://github.com/NetApp/trident/releases/download/v${TRIDENT_VERSION}/trident-installer-${TRIDENT_VERSION}.tar.gz"
   tar -xzf "/tmp/trident-installer-${TRIDENT_VERSION}.tar.gz" -C /tmp
   cd /tmp/trident-installer
   ./tridentctl install --generate-custom-yaml -n osac-csi-backends
   oc apply -f setup/trident-crds.yaml
   oc get crd | grep trident
   ```

   Do not apply `setup/trident-deployment.yaml` or
   `tridentorchestrator_cr.yaml`. Those files start a separate Trident
   controller or operator. Keep `setup/trident-daemonset.yaml`. Step 5
   patches that file into the node DaemonSet.
4. Deploy the OSAC Trident controller from the `csi-backends` chart. The
   chart does not install the CRDs from step 3, and it does not grant the
   `anyuid` SCC.

   ```bash
   export OSAC_DIR=/path/to/osac
   helm template trident-be "${OSAC_DIR}/osac-csi-driver/charts/csi-backends" \
     --namespace osac-csi-backends \
     --set namespace=osac-csi-backends \
     --set trident.enabled=true \
     --set trident.config.certsSecret=trident-csi \
     --set 'imagePullSecrets[0].name=pull-secret' \
     | oc apply -f -
   oc adm policy add-scc-to-user anyuid -z trident-vendor-sa -n osac-csi-backends
   oc rollout status deploy/trident-csi-controller \
     -n osac-csi-backends --timeout=180s
   ```

   Review the `anyuid` grant with the cluster administrator before you apply
   it. Do not enable the OSAC chart node DaemonSet
   (`vendors.trident.enabled`). That DaemonSet can provision a volume and
   still fail to mount NFS.
5. After the controller is Ready, install the official Trident node
   DaemonSet. The node service account and SCC below are not part of the
   OSAC distribution. A cluster administrator must review them against the
   selected Trident release and the cluster SCC policy before you apply them.
   The ClusterRole `trident-be-csi-backends-trident` comes from the chart in
   step 4, so create the binding only after that apply succeeds.

   ```bash
   cat > /tmp/trident-installer/setup/trident-node-linux-serviceaccount.yaml <<'EOF'
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: trident-node-linux
     namespace: osac-csi-backends
     labels:
       app: node.csi.trident.netapp.io
   EOF
   cat > /tmp/trident-installer/setup/trident-node-linux-scc.yaml <<'EOF'
   apiVersion: security.openshift.io/v1
   kind: SecurityContextConstraints
   metadata:
     annotations:
       kubernetes.io/description: 'trident-node-linux is a clone of the privileged built-in, and is meant just for use with trident.'
     name: trident-node-linux
     labels:
       app: node.csi.trident.netapp.io
   allowHostDirVolumePlugin: true
   allowHostIPC: true
   allowHostNetwork: true
   allowHostPID: true
   allowHostPorts: false
   allowPrivilegeEscalation: true
   allowPrivilegedContainer: true
   allowedCapabilities:
   - SYS_ADMIN
   allowedUnsafeSysctls: null
   defaultAddCapabilities: null
   fsGroup:
     type: RunAsAny
   groups: []
   priority: null
   readOnlyRootFilesystem: false
   requiredDropCapabilities: null
   runAsUser:
     type: RunAsAny
   seLinuxContext:
     type: RunAsAny
   supplementalGroups:
     type: RunAsAny
   users:
   - system:serviceaccount:osac-csi-backends:trident-node-linux
   volumes:
   - hostPath
   - downwardAPI
   - projected
   - emptyDir
   EOF
   oc apply -f /tmp/trident-installer/setup/trident-node-linux-serviceaccount.yaml
   oc apply -f /tmp/trident-installer/setup/trident-node-linux-scc.yaml
   oc adm policy add-scc-to-user trident-node-linux \
     -z trident-node-linux -n osac-csi-backends
   oc adm policy add-scc-to-user privileged \
     -z trident-node-linux -n osac-csi-backends
   oc create clusterrolebinding trident-node-linux-osac \
     --clusterrole=trident-be-csi-backends-trident \
     --serviceaccount=osac-csi-backends:trident-node-linux \
     --dry-run=client -o yaml | oc apply -f -
   ```

   Write `setup/trident-node-linux-osac.yaml` from
   `setup/trident-daemonset.yaml` in step 3. The script sets namespace
   `osac-csi-backends`, service account `trident-node-linux`, image
   `registry.connect.redhat.com/netapp/trident:25.10.0`, and the
   `pull-secret` from step 2. It removes the `trident-encryption-keys`
   secret from the `certs` volume. That secret is not created in step 2.
   Stop if this script exits with an error. Do not apply
   `setup/trident-deployment.yaml`.

   ```bash
   python3 <<'PY'
   from pathlib import Path
   src = Path("/tmp/trident-installer/setup/trident-daemonset.yaml")
   text = src.read_text()
   meta = "metadata:\n  name: trident-node-linux\n"
   if meta not in text:
       raise SystemExit("expected DaemonSet metadata name trident-node-linux")
   if "\n  namespace: osac-csi-backends\n" not in text:
       text = text.replace(meta, meta + "  namespace: osac-csi-backends\n", 1)
   sa = "      serviceAccount: trident-node-linux\n"
   if sa not in text:
       raise SystemExit("expected serviceAccount trident-node-linux")
   if "serviceAccountName: trident-node-linux\n" not in text:
       text = text.replace(
           sa, sa + "      serviceAccountName: trident-node-linux\n", 1)
   pull = "      serviceAccountName: trident-node-linux\n"
   if "name: pull-secret" not in text:
       text = text.replace(
           pull,
           pull + "      imagePullSecrets:\n      - name: pull-secret\n",
           1,
       )
   lines = text.splitlines(True)
   out = []
   in_main = False
   image_done = False
   i = 0
   while i < len(lines):
       line = lines[i]
       if line.strip() == "- name: trident-main":
           in_main = True
       elif in_main and line.startswith("      - name:"):
           in_main = False
       if in_main and not image_done and line.startswith("        image:"):
           out.append(
               "        image: registry.connect.redhat.com/netapp/trident:25.10.0\n")
           image_done = True
           i += 1
           continue
       if (line.strip() == "- secret:" and i + 1 < len(lines)
               and lines[i + 1].strip() == "name: trident-encryption-keys"):
           i += 2
           continue
       out.append(line)
       i += 1
   if not image_done:
       raise SystemExit("expected a trident-main image")
   patched = "".join(out)
   if "trident-encryption-keys" in patched:
       raise SystemExit("trident-encryption-keys is still present")
   Path("/tmp/trident-installer/setup/trident-node-linux-osac.yaml").write_text(
       patched)
   PY
   oc delete ds trident-csi-node -n osac-csi-backends --ignore-not-found
   oc apply -f /tmp/trident-installer/setup/trident-node-linux-osac.yaml
   oc rollout status ds/trident-node-linux \
     -n osac-csi-backends --timeout=180s
   fail=0
   if ! nodes=$(oc get nodes -l node-role.kubernetes.io/worker \
     -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'); then
     echo "Failed to list worker nodes" >&2
     exit 1
   fi
   if [ -z "$nodes" ]; then
     if ! nodes=$(oc get nodes \
       -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'); then
       echo "Failed to list nodes" >&2
       exit 1
     fi
   fi
   if [ -z "$nodes" ]; then
     echo "No nodes found" >&2
     exit 1
   fi
   for node in $nodes; do
     if ! oc get csinode "$node" \
         -o jsonpath='{range .spec.drivers[*]}{.name}{"\n"}{end}' \
         | grep -qx 'csi.trident.netapp.io'; then
       echo "csi.trident.netapp.io is not registered on ${node}"
       fail=1
     fi
   done
   test "$fail" -eq 0
   ```

   The last command fails if any worker is missing `csi.trident.netapp.io`.
   On a cluster with no `worker` role label, it checks every node.
6. From each node that will run VM pods, mount the dedicated NFS export with
   NFSv3, write a file, and unmount it. Stop if any node fails. Do not create
   the Trident backend until every node succeeds.

   ```bash
   if ! nodes=$(oc get nodes -l node-role.kubernetes.io/worker \
     -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'); then
     echo "Failed to list worker nodes" >&2
     exit 1
   fi
   if [ -z "$nodes" ]; then
     if ! nodes=$(oc get nodes \
       -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'); then
       echo "Failed to list nodes" >&2
       exit 1
     fi
   fi
   if [ -z "$nodes" ]; then
     echo "No nodes found" >&2
     exit 1
   fi
   for node in $nodes; do
     oc debug "node/${node}" -- chroot /host sh -c \
       "mkdir -p /mnt/osac-nfs-check && mount -t nfs -o nfsvers=3,sec=sys ${NFS_DATA_LIF}:${NFS_EXPORT_PATH} /mnt/osac-nfs-check && df -h /mnt/osac-nfs-check && echo ok > /mnt/osac-nfs-check/.osac-write-test && rm /mnt/osac-nfs-check/.osac-write-test && umount /mnt/osac-nfs-check" \
       || exit 1
   done
   ```

   The export policy on this share must allow each node's address. The
   Trident backend in step 7 uses export policy `default`. A successful mount
   of this dedicated export does not prove that `default` allows these nodes.
   Confirm that policy before you continue. This check does not mount the
   FlexVol pool Trident creates later. Confirm that pool when the first
   NetApp PVC is Bound, at the end of step 10. With `ontap-nas-economy`,
   Trident stores PVC qtrees in a FlexVol pool that it creates. Those qtrees
   are not placed in this export. Do not use `lun show` to verify this NFS
   workflow.
7. Create the Trident `ontap-nas-economy` backend and confirm that Trident
   reports it online. Write the JSON to a local file mode `600`. Do not
   commit that file. Applying the JSON as a Kubernetes resource does not
   create a Trident backend. Do not run `tridentctl create backend -n
   osac-csi-backends` on the workstation. That client looks for a stock
   Trident pod. The controller container has no `tar`, so `oc cp` cannot
   copy the file in. Send the JSON on stdin to `tridentctl` in the
   controller pod.

   ```bash
   umask 077
   ONTAP_BACKEND_FILE=$(mktemp)
   cat > "$ONTAP_BACKEND_FILE" <<EOF
   {
     "version": 1,
     "storageDriverName": "ontap-nas-economy",
     "backendName": "ontap-economy",
     "managementLIF": "${ONTAP_MANAGEMENT_ENDPOINT}",
     "dataLIF": "${NFS_DATA_LIF}",
     "svm": "${ONTAP_SVM}",
     "username": "${ONTAP_USERNAME}",
     "password": "${ONTAP_PASSWORD}",
     "storagePrefix": "osac_",
     "nfsMountOptions": "nfsvers=3",
     "defaults": {
       "spaceReserve": "none",
       "snapshotPolicy": "none",
       "exportPolicy": "default"
     }
   }
   EOF
   POD=$(oc get pod -n osac-csi-backends \
     -l app.kubernetes.io/name=trident-csi-controller \
     -o jsonpath='{.items[0].metadata.name}')
   oc exec -i -n osac-csi-backends "$POD" -c trident-main -- \
     tridentctl -s 127.0.0.1:8000 create backend -f - \
     < "$ONTAP_BACKEND_FILE" \
     && oc exec -n osac-csi-backends "$POD" -c trident-main -- \
       tridentctl -s 127.0.0.1:8000 get backend \
     && rm -f "$ONTAP_BACKEND_FILE"
   ```

   The file is removed only after create and get both succeed. If create
   fails, keep it and use a fallback below.

   `managementLIF` is the SVM management host from step 1. `dataLIF` is the
   NFS data LIF from step 6. See NetApp's
   [ONTAP NAS backend examples](https://docs.netapp.com/us-en/trident-2510/trident-use/ontap-nas-examples.html).

   If `tridentctl` in the container cannot reach `127.0.0.1:8000`,
   port-forward HTTP port 8000 to a free local port and use the installer
   `tridentctl`. Do not pass port `8443` to `tridentctl -s`. That port is
   TLS, and `tridentctl -s` speaks HTTP.

   ```bash
   oc port-forward -n osac-csi-backends deploy/trident-csi-controller 18000:8000
   ```

   ```bash
   /tmp/trident-installer/tridentctl -s 127.0.0.1:18000 create backend \
     -f "$ONTAP_BACKEND_FILE" \
     && /tmp/trident-installer/tridentctl -s 127.0.0.1:18000 get backend \
     && rm -f "$ONTAP_BACKEND_FILE"
   ```

   If nothing in the pod listens on port 8000, REST is HTTPS on port 8443.
   The controller requires the client certificate from step 2. Do not use
   `-k`. The server certificate name is `trident-csi`, so call that host and
   send the connection to the local port-forward.

   ```bash
   oc port-forward -n osac-csi-backends deploy/trident-csi-controller 18443:8443
   ```

   ```bash
   CERTDIR=$(mktemp -d)
   umask 077
   for f in caCert clientCert clientKey; do
     oc -n osac-csi-backends get secret trident-csi \
       -o jsonpath="{.data.${f}}" | base64 -d > "${CERTDIR}/${f}"
   done
   curl --fail --cacert "${CERTDIR}/caCert" \
     --cert "${CERTDIR}/clientCert" --key "${CERTDIR}/clientKey" \
     --resolve "trident-csi:18443:127.0.0.1" \
     -X POST "https://trident-csi:18443/trident/v1/backend" \
     -H 'Content-Type: application/json' \
     --data-binary @"$ONTAP_BACKEND_FILE" \
     && rm -f "$ONTAP_BACKEND_FILE"
   rm -rf "$CERTDIR"
   ```
8. Create the tenant on the private administrator session from step 1, if
   it does not already exist. A public session cannot create a tenant. The
   create output prints the break-glass username and password once. Copy
   them before you continue. The username is `${TENANT}-osac-break-glass`.
   Wait until the tenant is `SYNCED`. The default VirtualNetwork, Subnet,
   and SecurityGroup are created with the tenant. Do not create a second
   VirtualNetwork.

   ```bash
   osac create -f - <<EOF
   "@type": type.googleapis.com/osac.private.v1.Tenant
   metadata:
     name: ${TENANT}
   EOF
   oc get tenant "$TENANT" -n "$OSAC_NAMESPACE"
   oc get sc "osac-${TENANT}-local" --show-labels
   ```

   Creating the tenant grants IdP manager only. VM create requires
   `tenant-admin`. In the OSAC console, log in as the break-glass user,
   open **Role Bindings**, create a binding for role `tenant-admin` for
   that user, and wait until it is `READY`. Log out and back in. The header
   must show `tenant-admin`. If Keycloak asks for a new password, do not use
   `%` or `#`.

   Then sign in on the internal API as that user. `osac whoami` must list
   `tenant-admin`. `osac get tenants` returns PermissionDenied for this
   user.

   ```bash
   INT_ROUTE=$(oc get route fulfillment-internal-api -n "$OSAC_NAMESPACE" \
     -o jsonpath='{.spec.host}')
   KC_HOST=$(oc get route keycloak -n keycloak -o jsonpath='{.spec.host}')
   osac logout 2>/dev/null || true
   osac login --address "$INT_ROUTE" \
     --ca-file default-ca.crt \
     --issuer="https://${KC_HOST}/realms/osac" \
     --flow=password \
     --user="${TENANT}-osac-break-glass" \
     --password="$BREAK_GLASS_PASSWORD"
   osac whoami
   ```
9. Create one tenant StorageClass for the tier. Use the Trident
   `ontap-nas-economy` driver and thin provisioning. Set the tenant metadata
   and the labels OSAC uses to discover the class.

   ```bash
   cat <<EOF | oc apply -f -
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: ontap-nas-${TENANT}
     annotations:
       osac.openshift.io/tenant: "${TENANT}"
       osac.openshift.io/owner-reference: "Tenant/${TENANT}"
     labels:
       osac.openshift.io/tenant: "${TENANT}"
       osac.openshift.io/storage-tier: ontap-nas
   provisioner: csi.trident.netapp.io
   allowVolumeExpansion: true
   volumeBindingMode: Immediate
   parameters:
     backendType: ontap-nas-economy
     provisioningType: thin
   EOF
   ```

   After you apply the class, confirm that the tenant's
   `status.storageClasses` lists both the existing local class
   (`osac-${TENANT}-local`, tier `local`) and `ontap-nas-${TENANT}` (tier
   `ontap-nas`). Labels alone do not publish the class to Fulfillment.
   Request reconciliation first. Do not replace or remove the `local` entry.

   ```bash
   oc annotate tenant "$TENANT" -n "$OSAC_NAMESPACE" \
     osac.openshift.io/reconcile="$(date +%s)" --overwrite
   oc get tenant "$TENANT" -n "$OSAC_NAMESPACE" \
     -o jsonpath='{.status.storageClasses}{"\n"}'
   ```

   If reconciliation does not add the class, an authorized cluster
   administrator can patch the status subresource. Include both entries in
   the patch.

   ```bash
   cat > /tmp/tenant-storage-classes.json <<EOF
   {"status":{"storageClasses":[
     {"name":"osac-${TENANT}-local","tier":"local"},
     {"name":"ontap-nas-${TENANT}","tier":"ontap-nas"}
   ]}}
   EOF
   oc patch tenant "$TENANT" -n "$OSAC_NAMESPACE" \
     --type=merge --subresource=status \
     --patch-file=/tmp/tenant-storage-classes.json
   ```
10. Stay signed in as the tenant user from step 8. Do not use `--private`.
    A `--private` session creates the VM on the `shared` tenant. Read the
    default subnet and security group, then create a VM on the `local` tier
    and wait until it is `Running`. Its PVC must use `osac-${TENANT}-local`.
    Then create a second VM that uses the NetApp tier for its root disk.
    You can set an optional additional disk when you create that VM. This
    workflow cannot add a disk to a VM that already exists. If create fails
    because the catalog name is unknown, sign in with `--private`, run
    `osac get computeinstancecatalogitem`, and retry with that item's UUID
    on the tenant session.

    The guest password is written to a file and passed with `osac create -f`.
    It is not an argument of the `osac` process.

    ```bash
    SUBNET_ID=$(osac get subnet default-ipv4 -o json | jq -r '.id')
    SECURITY_GROUP_ID=$(osac get securitygroup default -o json | jq -r '.id')
    umask 077
    LOCAL_VM_FILE=$(mktemp)
    cat > "$LOCAL_VM_FILE" <<EOF
    "@type": type.googleapis.com/osac.public.v1.ComputeInstance
    metadata:
      name: local-vm
    spec:
      catalog_item: ${CATALOG_ITEM}
      instance_type: ${INSTANCE_TYPE}
      boot_disk:
        size_gib: 10
        storage_tier:
          name: local
      network_attachments:
        - subnet: ${SUBNET_ID}
          security_groups:
            - ${SECURITY_GROUP_ID}
      run_strategy: Always
      user_data: |
        #cloud-config
        user: osac-admin
        password: ${GUEST_PASSWORD}
        chpasswd:
          expire: false
        ssh_pwauth: true
    EOF
    osac create -f "$LOCAL_VM_FILE"
    rm -f "$LOCAL_VM_FILE"
    osac get computeinstance
    ```

    ```bash
    umask 077
    NETAPP_VM_FILE=$(mktemp)
    cat > "$NETAPP_VM_FILE" <<EOF
    "@type": type.googleapis.com/osac.public.v1.ComputeInstance
    metadata:
      name: netapp-vm
    spec:
      catalog_item: ${CATALOG_ITEM}
      instance_type: ${INSTANCE_TYPE}
      boot_disk:
        size_gib: 10
        storage_tier:
          name: ontap-nas
      additional_disks:
        - size_gib: 20
          storage_tier:
            name: ontap-nas
      network_attachments:
        - subnet: ${SUBNET_ID}
          security_groups:
            - ${SECURITY_GROUP_ID}
      run_strategy: Always
      user_data: |
        #cloud-config
        user: osac-admin
        password: ${GUEST_PASSWORD}
        chpasswd:
          expire: false
        ssh_pwauth: true
    EOF
    osac create -f "$NETAPP_VM_FILE"
    rm -f "$NETAPP_VM_FILE"
    ```

    Omit `additional_disks` for a root-only VM. The VM order must use the
    tenant's `ontap-nas` tier. The generated PVC must use the tenant-specific
    class `ontap-nas-${TENANT}`. The local VM does not need the AAP workaround
    below. The NetApp VM does. When that job fails, leave the
    `ComputeInstance` in `Starting`. Do not delete it.

**AAP workaround**

The first provisioning job for the NetApp VM fails with `the role
'osac.templates.ontap_storage' was not found`, and the `ComputeInstance`
stays in `Starting`. The AAP collection does not include that role. Do not
delete the `ComputeInstance`.

The workaround relaunches the failed job template after removing only the
`ontap-nas` entry from `storage_tier_definitions`. It fills in for the
missing role. Repeat it for each affected VM. It does not install Trident
or create the StorageClass.

In one terminal, port-forward the AAP controller service in the OSAC
namespace. In a second terminal, read the failed AAP job ID from that
`ComputeInstance` and run the script on a trusted admin workstation. The
script prompts for the AAP password so the password is not stored in shell
history. Confirm the failed job ID and the template ID before you launch the
job, and use the failed job for this VM only.

```bash
oc port-forward -n "$OSAC_NAMESPACE" svc/osac-aap-controller-service 18053:80
```

In the second terminal, set `CI_NAME` to the `ComputeInstance` that is still
in `Starting`:

```bash
CI_NAME="replace-with-computeinstance-name"
FAILED_JOB=$(oc get computeinstance "$CI_NAME" -n "$OSAC_NAMESPACE" \
  -o jsonpath='{.status.provisioningJobs[-1].jobID}')
printf 'Failed AAP job ID: %s\n' "$FAILED_JOB"
```

```python
import base64
import getpass
import json
import urllib.request

api = "http://127.0.0.1:18053"
username = input("AAP username: ")
password = getpass.getpass("AAP password: ")
job_id = input("Failed AAP job ID for this VM: ")
auth = base64.b64encode(f"{username}:{password}".encode()).decode()

def request(path, payload=None):
    data = None if payload is None else json.dumps(payload).encode()
    headers = {"Authorization": f"Basic {auth}", "Content-Type": "application/json"}
    req = urllib.request.Request(f"{api}{path}", data=data, headers=headers)
    with urllib.request.urlopen(req) as response:
        return json.load(response)

failed_job = request(f"/api/v2/jobs/{job_id}/")
template_id = failed_job["summary_fields"]["job_template"]["id"]
print(f"Job {job_id} status {failed_job.get('status')}, template {template_id}")
if input("Launch this template? Type yes: ") != "yes":
    raise SystemExit("not launched")
extra_vars = failed_job["extra_vars"]
if isinstance(extra_vars, str):
    extra_vars = json.loads(extra_vars)
job_vars = extra_vars["osac_job_vars"]
job_vars["storage_tier_definitions"] = [
    tier for tier in job_vars["storage_tier_definitions"]
    if tier.get("name") != "ontap-nas"
]
launched = request(
    f"/api/v2/job_templates/{template_id}/launch/",
    {"extra_vars": {"osac_job_vars": job_vars}},
)
print(f"Launched AAP job {launched.get('id')}")
```

Wait until the new AAP job finishes. Confirm that the `ComputeInstance`
reaches `Running`, and that both the root PVC and the optional `*-disk-1`
PVC are Bound to `ontap-nas-${TENANT}`. Confirm that the PV CSI driver is
`csi.trident.netapp.io`. In the guest, the additional disk appears as
`/dev/vdb` and is not formatted or mounted automatically. With
`ontap-nas-economy`, Trident stores PVCs as qtrees in Trident-managed FlexVol
pools. In the tested workflow, those qtrees were not inside the dedicated
export used to check NFS reachability. Do not use `lun show` to verify this
NFS workflow.

Use the subnet namespace from the tenant's default subnet to inspect the VM
and PVCs. `internalID` is the SVM, the FlexVol pool, and the qtree for that
disk. On ONTAP, run `qtree show -volume <flexvol>` and match `internalName`.
Those qtrees are on the pool volume, not on the export mounted in step 6.
If `driver` is `topolvm.io`, that disk is the `local` tier.

```bash
osac get computeinstance
oc get vm,dv,pvc -n "subnet-${SUBNET_ID}"
oc get pvc -n "subnet-${SUBNET_ID}" -o json \
  | jq -r --arg sc "ontap-nas-${TENANT}" '.items[] | select(.spec.storageClassName==$sc) | .metadata.name' \
  | while read -r pvc; do
      echo "===== ${pvc} ====="
      vol=$(oc get pvc -n "subnet-${SUBNET_ID}" "$pvc" -o jsonpath='{.spec.volumeName}')
      oc get pv "$vol" -o jsonpath='driver={.spec.csi.driver}{"\n"}internalID={.spec.csi.volumeAttributes.internalID}{"\n"}internalName={.spec.csi.volumeAttributes.internalName}{"\n"}'
    done
```

This workaround depends on AAP REST job variables and on the job template
that failed. A later AAP or OSAC release can change it.

---

## 7. Verifying the installation

Perform the following steps in order. Each step assumes that the previous steps
passed. If a step fails, see [Section 11](#11-troubleshooting).

**Procedure**

1. Check the Helm releases. The `osac` release must show `STATUS: deployed`. A
   status of `pending-install` or `pending-upgrade` means that a hook is still
   running or has failed.

   ```console
   $ helm list -A | grep osac
   ```

2. Check the pre-installation validation hook. The log ends with
   `=== Validation passed ===`.

   ```console
   $ oc logs job/osac-pre-install-validate -n <namespace>
   ```

   A missing `certificates.cert-manager.io` CRD aborts the installation. When
   `bmf.metal3.enabled` is set, a missing `BareMetalHost` CRD or a
   `Provisioning` resource without `watchAllNamespaces: true` also aborts the
   installation. A message about a missing default storage class is a warning
   only.

3. Check the phase-1 Operators. Each expected CSV must be `Succeeded`:

   ```console
   $ oc get csv -A | grep -E 'cert-manager|ansible-automation|lvms|metallb|kubevirt|multicluster'
   ```

4. Check the phase-1 operands. Each operand must be ready:

   ```console
   $ oc get hyperconverged -n openshift-cnv
   $ oc get lvmcluster -n openshift-storage
   $ oc get ipaddresspool -n metallb-system
   $ oc get clusterissuer default-ca
   $ oc get bundle -n cert-manager
   ```

5. Check the infrastructure layer:

   ```console
   $ oc get pods -n osac-infra
   ```

   If `keycloak.enabled` is `true`, also check that the `keycloak-service` and
   `keycloak-database` pods are `Running`. Skip this check for an external
   Keycloak.

   ```console
   $ oc get pods -n keycloak
   ```

6. Confirm that the Keycloak realm responds:

   ```console
   $ curl -sk "https://keycloak-keycloak.$DOMAIN/realms/osac/.well-known/openid-configuration"
   ```

7. Check the phase-2 pods. All pods must be `Running` or `Completed`, with no
   pods in `CrashLoopBackOff` or `ImagePullBackOff`:

   ```console
   $ oc get pods -n <namespace>
   ```

   Depending on the enabled services, expect the following pods:

   - Always: `fulfillment-grpc-server`, `fulfillment-rest-gateway`,
     `fulfillment-controller`, `fulfillment-ingress-proxy`, `osac-operator`,
     `osac-operator-console-proxy`, and the `osac-aap-*` pods.
   - With `ui.enabled`: `osac-ui`.
   - With `metering.enabled`: the `osac-metering` pods.
   - With `bundledVault.enabled`: `openbao-0`.
   - VMaaS: `fulfillment-console-proxy`.
   - BMaaS: the Bare Metal Fulfillment Operator pod.

8. Check the certificates and the database initialization. Every `Certificate`
   must be `Ready`, and the `osac-db-init` job must complete:

   ```console
   $ oc get certificate -n <namespace>
   $ oc logs job/osac-db-init -n <namespace>
   ```

9. Check the AAP bootstrap job. The `osac-aap-bootstrap` job runs as a
   postinstallation hook and takes 10 to 40 minutes:

   ```console
   $ oc logs -f job/osac-aap-bootstrap -n <namespace>
   ```

10. CaaS only: check that the cluster templates were published:

    ```console
    $ oc logs job/osac-publish-templates -n <namespace>
    $ osac get clustertemplates
    ```

    The `osac get clustertemplates` output must be non-empty.

11. Check API reachability and log in with the `osac` CLI:

    ```console
    $ ROUTE=$(oc get route fulfillment-api -n <namespace> -o jsonpath='{.spec.host}')
    $ curl -sk "https://$ROUTE/healthz"
    $ oc extract secret/default-ca -n cert-manager --keys=tls.crt --to=- > default-ca.crt
    $ osac login --address "$ROUTE" --token-script "oc create token fulfillment-controller -n <namespace> --duration 1h" --ca-file default-ca.crt
    $ osac get tenants
    ```

    Use `--ca-file` to trust the route's certificate, including the
    self-signed `default-ca`. Use `--insecure` (skips certificate
    verification) only for evaluation, never in production.

12. Check the web console. Open the console URL in a browser. The URL must
    redirect to Keycloak and, after you log in, show the OSAC console. Skip
    this step if `ui.enabled` is `false`.

    ```console
    $ UI=$(oc get route osac-ui -n <namespace> -o jsonpath='{.spec.host}')
    $ curl -skI "https://$UI"
    ```

    The response must be `200` or a `302` redirect to Keycloak.

13. Run a service smoke test, as described in the Verification section of the
    workflow in [Section 6](#6-installation-workflows-by-service) for your
    service.

---

## 8. Postinstallation tasks

### 8.1 Accessing the OSAC consoles

OSAC exposes four consoles as OpenShift Container Platform `Route` resources.
The OSAC web console, Fulfillment API, and AAP routes are in the install
namespace; the Keycloak route is in the `keycloak` namespace. Host names are
auto-assigned as `<route>-<namespace>.<cluster_domain>` unless you set
`ui.externalHostname`, `service.externalHostname`, or `keycloak.route.hostname`.

```console
$ oc get route -n <namespace>
$ oc get route -n keycloak
```

- **OSAC web console.** Deployed whenever `ui.enabled` is `true`, which is the
  default. The route uses edge TLS termination and redirects HTTP to HTTPS. Log
  in through Keycloak SSO against the `osac` realm. With the bundled Keycloak,
  use the `keycloak.adminUsername` and `keycloak.adminPassword` values that you
  set. Production deployments use realm users or a federated identity provider.
- **Fulfillment API.** Authenticate with an OIDC token or with `osac login`, as
  described in step 11 of [Section 7](#7-verifying-the-installation).
- **AAP.** Log in as `admin`. To retrieve the password, run the following
  command:

  ```console
  $ oc extract secret/osac-aap-admin-password -n <namespace> --to -
  ```

- **Keycloak admin console.** Log in with the `keycloak.adminUsername` and
  `keycloak.adminPassword` values.

### 8.2 Installing the `osac` CLI

Download the CLI version matching the `fulfillment-service` subchart pinned
by your `osac` chart release (`0.0.107` for release `0.0.17` — see
[Section 3](#3-osac-component-versions)). Releases are tagged and published
on the monorepo, not on the `fulfillment-service` repository itself:

```console
$ curl -L -o osac https://github.com/osac-project/osac/releases/download/fulfillment-service/v0.0.107/osac_Linux_x86_64
$ chmod +x osac
$ sudo mv osac /usr/local/bin/
```

### 8.3 Registering the hub

The hub is the OpenShift Container Platform cluster that the OSAC Operator
and AAP run on and that provisions resources. This procedure applies when the
Fulfillment Service and the hub run on the same cluster.

**Procedure**

1. Log in to the Fulfillment Service:

   ```console
   $ osac login \
       --address "$(oc get route fulfillment-api -n <namespace> -o jsonpath='{.spec.host}')" \
       --token-script "oc create token fulfillment-controller -n <namespace> --duration 1h"
   ```

2. Generate the hub-access kubeconfig file:

   ```console
   $ curl -sO https://raw.githubusercontent.com/osac-project/osac/refs/tags/osac/v0.0.17/osac-installer/scripts/create-hub-access-kubeconfig.sh
   $ chmod +x create-hub-access-kubeconfig.sh
   $ ./create-hub-access-kubeconfig.sh
   ```

3. Register the hub:

   ```console
   $ osac create hub --kubeconfig=kubeconfig.hub-access --id <hub_name> --namespace <namespace>
   ```

If the API route presents a certificate your client doesn't trust, such as
the self-signed `default-ca`, add `--ca-file default-ca.crt` to the
`osac login` command (see step 11 of
[Section 7](#7-verifying-the-installation) for how to extract it). Use
`--insecure` instead only for evaluation, never in production. Add
`--as system:admin` only when your `oc` context cannot mint the token.

Multi-hub environments are currently unsupported, and support may be added in future releases.

### 8.4 Installing without DNS management

`dns.route53.dns` is the only `DNS_CLASS` the chart schema accepts; a Helm
install always requires Route 53 credentials, even in an on-premises
environment with no Route 53 zone. To run CaaS without OSAC managing DNS,
install normally and then patch the `cluster-fulfillment-ig` `ConfigMap`
afterward — this bypasses schema validation, so it can't be set through Helm
values directly.

**Procedure**

1. After installation, and before creating the first `ClusterOrder`, patch
   the `ConfigMap`:

   ```console
   $ export NS=<namespace>
   $ oc -n "$NS" patch cm cluster-fulfillment-ig --type merge -p '{
       "data": {
         "DNS_CLASS": "dns.noop.dns",
         "EXTERNAL_ACCESS_BASE_DOMAIN": "<guest_domain>",
         "HOSTED_CLUSTER_BASE_DOMAIN": "<guest_domain>",
         "EXTERNAL_ACCESS_SUPPORTED_BASE_DOMAINS": "<supported_domain>"
       }
     }'
   ```

2. Confirm the values took:

   ```console
   $ oc get cm cluster-fulfillment-ig -n "$NS" -o yaml \
       | grep -E 'DNS_CLASS|EXTERNAL_ACCESS|HOSTED_CLUSTER'
   ```

3. Create these DNS records yourself; `dns.noop.dns` stops OSAC from managing
   DNS, it does not create records for you:

   ```text
   api.<cluster>.<guest_domain>       -> API endpoint
   *.apps.<cluster>.<guest_domain>    -> ingress endpoint
   ```

   The records must resolve from the hub, the AAP execution environment, the
   managed host, and the hosted cluster.

Reapply the patch after any infrastructure reinstall that recreates this
`ConfigMap`.

### 8.5 Additional resources

- CaaS network backend configuration:
  [`network-backend.md`](https://github.com/osac-project/osac/blob/main/osac-installer/docs/network-backend.md)
- CaaS DNS backend configuration:
  [`dns-backend.md`](https://github.com/osac-project/osac/blob/main/osac-installer/docs/dns-backend.md)
- AAP instance group configuration:
  [`aap-configuration.md`](https://github.com/osac-project/osac/blob/main/osac-installer/docs/aap-configuration.md)

---

## 9. Supported configurations

### 9.1 Supported for production

- Deployment onto an existing OpenShift Container Platform cluster with
  `cluster-admin` privileges.
- The published `osac` chart at a tagged release, such as `0.0.17`. The chart
  `values-example.yaml` file documents the Production block: an external
  PostgreSQL database, an external Keycloak, and pinned image tags.
- Prerequisite Operators and infrastructure already present on the cluster —
  installed by whoever prepares the cluster using the
  [Helm Deployment Guide](https://github.com/osac-project/osac/blob/main/docs/guides/installation/helm-deployment-guide.md).
- An external PostgreSQL 18 or later database with the `osac-db-*` Secrets
  created in advance. See
  [Section 2.4](#24-credentials-and-external-services).
- An external Vault-compatible store configured with a parent namespace,
  Keycloak JWT role, and `service.vault` values. See the
  [secrets management configuration guide](secrets-management-configuration.md).
- An external Keycloak configured with the `osac` realm, clients, and roles
  through `service.auth` and `service.idp`.
- A single hub cluster.

### 9.2 Manual procedures

- The NetApp ONTAP and Trident workflow in
  [Section 6.4](#64-manual-netapp-ontap-and-trident-integration-for-vmaas)
  adds an NFS tier to an existing VMaaS installation by hand. The
  `csi-backends` chart supplies the Trident controller. It does not install
  the Trident persistence CRDs, grant the SCCs, or provide a node DaemonSet
  that can mount NFS. A cluster administrator creates the Trident backend and
  the tenant StorageClass. The AAP collection has no ONTAP storage role, so
  the first NetApp VM job is relaunched as described in that section.

### 9.3 Evaluation only

- The bundled PostgreSQL database (`bundledPostgres.enabled: true`) is for
  testing only and is not intended for production.
- The bundled OpenBao secret store (`bundledVault.enabled: true`). It is for
  testing and CI, not production.
- `keycloak.devFixtures.enabled: true` and the default `admin` Keycloak
  credentials.
- Development or CI reference values files and `main`/`latest` image tags.

### 9.4 Not covered by this guide

- Setting up the phase-1 prerequisites yourself. See the
  [Helm Deployment Guide](https://github.com/osac-project/osac/blob/main/docs/guides/installation/helm-deployment-guide.md).
- Detailed configuration of an external Keycloak. The chart accepts an external
  Keycloak through `service.auth` and `service.idp`, but realm and client
  provisioning is out of scope. See
  [`fulfillment-service/docs/INSTALL.md`](https://github.com/osac-project/osac/blob/main/fulfillment-service/docs/INSTALL.md).
- Multi-hub topologies (TBD — see the note in
  [Section 8.3](#83-registering-the-hub)).

---

## 10. Uninstalling OSAC

**Procedure**

1. Uninstall the `osac` release:

   ```console
   $ helm uninstall osac -n "$NS"
   ```

2. The CRDs are retained because they carry the
   `helm.sh/resource-policy: keep` annotation. To remove them, run the
   following command:

   ```console
   $ oc delete crd -l app.kubernetes.io/part-of=osac
   ```

If your platform team also installed the phase-1 prerequisites for this
deployment and they need to come down too, see "Uninstall" in the
[Helm Deployment Guide](https://github.com/osac-project/osac/blob/main/docs/guides/installation/helm-deployment-guide.md#uninstall).
Do not remove shared prerequisites on a cluster used by other deployments.

---

## 11. Troubleshooting

Failed hook jobs are retained for inspection. To find them, run the following
command and then view the logs for each job:

```console
$ oc get pods -n <namespace> | grep -E 'validate|db-init|publish-templates|bootstrap'
```

### 11.1 The pre-installation validation hook fails

View the hook log:

```console
$ oc logs job/osac-pre-install-validate -n <namespace>
```

- `cert-manager CRDs not found`: cert-manager is not installed, or its CRDs are
  in a different API group. Install cert-manager. Set `certManager.enabled` to
  `false` only when a cert-manager distribution is present.
- `BareMetalHost CRD ... not found`, `No Provisioning CR found`, or
  `Provisioning CR has watchAllNamespaces: false`: these are BMaaS Metal3
  prerequisites. Install BareMetalOperator, then patch the `Provisioning`
  resource:

  ```console
  $ oc patch provisioning provisioning-configuration --type merge -p '{"spec":{"watchAllNamespaces": true}}'
  ```

  Or set `bmf.metal3.enabled` to `false`.
- `No default StorageClass found`: this is a warning, not a failure. Keycloak
  and PostgreSQL persistent volume claims remain `Pending`. Set a default
  storage class. See [Section 2.1](#21-cluster-and-access).

### 11.2 A Helm release is stuck in `pending-install` or `pending-upgrade`

A hook is still running or has failed. To find it, run the following command:

```console
$ oc get jobs,pods -n <namespace> | grep -Ev 'Complete|Running'
```

If a previous attempt was interrupted, clear the stuck release before you
retry:

```console
$ helm uninstall osac -n <namespace> --no-hooks
```

### 11.3 An Operator CSV never reaches `Succeeded`

```console
$ oc get subscription,installplan,csv -n <operator_namespace>
$ oc get pods -n openshift-marketplace
$ oc get packagemanifest | wc -l
```

`ImagePullBackOff` on the marketplace catalog pods, or an empty
`packagemanifest` list, indicates that the cluster pull secret cannot
authenticate to `registry.redhat.io`. Refresh `openshift-config/pull-secret`
(see [Section 2.1](#21-cluster-and-access)) and delete the failed marketplace
pods. The Streams for Apache Kafka `Subscription` uses a manual install plan;
approve its `InstallPlan` in the `osac-kafka` namespace.

### 11.4 An OCI pull fails

A `not found` error on `oci://ghcr.io/osac-project/charts/osac` usually
indicates an incorrect `--version` value. To list the tags, run the following
command:

```console
$ helm show chart oci://ghcr.io/osac-project/charts/osac --version 0.0.17
```

### 11.5 The `osac-db-init` hook fails

- `Secret osac-db-config not found`, `has an empty url key`, or
  `invalid PostgreSQL url`: create the `osac-db-config` and
  `osac-db-client-cert` Secrets before you install the `osac` chart (see
  [Section 4.1](#41-preparing-the-cluster)), or enable `bundledPostgres` for
  testing.
- `PostgreSQL Service ... has no ready endpoints`: the host in the database URL
  does not resolve to a running PostgreSQL database. Point `dbInit.host` and the
  URL at a reachable server.

### 11.6 Certificates never become `Ready`

```console
$ oc get certificate,certificaterequest -n <namespace>
$ oc get clusterissuer default-ca
$ oc logs -n cert-manager deploy/cert-manager
```

A missing or not-ready `default-ca` `ClusterIssuer` blocks every downstream
certificate. If you disabled `caIssuer.enabled`, `service.certs.issuerRef`
must name an issuer that exists.

### 11.7 The AAP instance does not start

The `osac-aap-*` pods are stuck, or the `AnsibleAutomationPlatform` custom
resource is not progressing:

```console
$ oc get aap,pods -n <namespace>
$ oc get apiservice v1beta1.metrics.k8s.io
$ oc get csr | grep -c Pending
```

- `Unable to determine if virtual resource` in the AAP custom resource: a
  broken `APIService`, usually `v1beta1.metrics.k8s.io` when metrics-server is
  not ready, makes API discovery fail for the Ansible-based Operator. Fix
  metrics-server, or delete the unavailable `APIService`.
- `tls: internal error` on `oc logs`, `oc debug`, or in the AAP status: pending
  `kubernetes.io/kubelet-serving` CSRs. To approve them, run the following
  command:

  ```console
  $ oc get csr -o name | xargs oc adm certificate approve
  ```

- `label validation error: key "app.kubernetes.io/managed-by" must equal "Helm"`:
  the AAP Operator rewrote the `managed-by` label of the custom resource after
  an interrupted installation. Restore the label and the
  `meta.helm.sh/release-name` and `meta.helm.sh/release-namespace` annotations,
  then reinstall:

  ```console
  $ oc label aap osac-aap -n <namespace> app.kubernetes.io/managed-by=Helm --overwrite
  ```

- a field-ownership conflict on `app.kubernetes.io/managed-by` owned by the
  AAP operator (Helm 4): add `--force-conflicts` to the `helm upgrade`
  command. The operator takes ownership of that field on the `osac-aap`
  custom resource after install.

### 11.8 The `osac-aap-bootstrap` job fails

```console
$ oc logs -f job/osac-aap-bootstrap -n <namespace>
$ oc get secret config-as-code-manifest-ig -n <namespace>
```

Common causes:

- The AAP subscription manifest Secret is missing or invalid. Recreate it. See
  [Section 4.1](#41-preparing-the-cluster).
- AAP is not yet reachable. The job retries up to `aap.bootstrap.backoffLimit`
  times.
- The config-as-code Git source (`aap.configAsCode.projectGitUri` and
  `aap.configAsCode.projectGitBranch`) is unreachable.

### 11.9 The `osac-publish-templates` hook fails

```console
$ oc logs job/osac-publish-templates -n <namespace> -c wait-for-fulfillment
$ oc logs job/osac-publish-templates -n <namespace> -c publish-templates
```

The hook runs by default, mainly for CaaS. The init container polls the
Fulfillment Service REST gateway for up to 600 seconds. The job then
launches the `osac-publish-templates` AAP job template and requires a valid
`osac-aap-api-token` Secret. To disable the hook for VMaaS-only or
BMaaS-only installations, set `aap.instanceGroups.publishTemplates.enabled`
to `false`.

### 11.10 The `fulfillment-*` pods are in `CrashLoopBackOff`

```console
$ oc logs deploy/fulfillment-grpc-server -n <namespace>
$ oc logs deploy/fulfillment-rest-gateway -n <namespace>
$ oc logs deploy/fulfillment-controller -n <namespace>
```

- `issuer URL '...' is not trusted`: the `--auth-issuer-url` and `--idp-url`
  values of the service do not match the external host name of Keycloak. These
  values derive from `global.clusterDomain`; confirm that it was set at
  installation and matches `keycloak.route.hostname`. To check the value the
  running container actually received:

  ```console
  $ oc get deploy fulfillment-grpc-server -n <namespace> -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n' | grep -E 'issuer|idp'
  ```
- Missing `osac-db-config` or controller-credential Secrets: the prerequisite
  infrastructure did not create them. Check with whoever prepared the cluster.
- `lookup openbao.<namespace>.svc ... no such host` or
  `Failed to provision vault namespace`: `bundledVault.enabled` is `false` but
  `service.vault.endpoint` still points at the in-cluster OpenBao, or the
  external store's namespace, policy, JWT role, or credentials are incorrect.
  Check the endpoint and follow the
  [secrets management configuration guide](secrets-management-configuration.md#troubleshooting).

### 11.11 Web console login loops with an `issuer not trusted` error

This has the same root cause as the Fulfillment Service issuer error: a
`global.clusterDomain` mismatch between the web console OIDC configuration, the
Fulfillment Service, and the Keycloak `KC_HOSTNAME`. Confirm that all three
resolve to `keycloak-keycloak.<cluster_domain>` and run `helm upgrade` again
with the correct `global.clusterDomain`.

### 11.12 The first provisioning request fails while the pods are healthy

- **VMaaS:** `Storage tier "..." is not available for tenant "shared"`, or an
  empty `status.storageClasses` field on the `shared` `Tenant`: the storage
  tier that the create-VM playbook requests is not registered. Check
  `oc get sc --show-labels` for the `osac.openshift.io/tenant` and
  `osac.openshift.io/storage-tier` labels and the `status.storageClasses` field
  of the `Tenant`, and align the requested tier with a labeled storage class.
- Networking custom resources stuck in `PROGRESSING` on a cluster with no real
  fabric: set `operator.controllers.networkingProvisioning` to `false`.
- **CaaS:** watch the AAP `cluster-fulfillment` job for the failing task. Netris
  or Route 53 credential errors, and `NETRIS_RESOURCE_CLASS_MAP` errors, appear
  there.

### 11.13 Additional resources

- "Troubleshooting" in the
  [Helm Deployment Guide](https://github.com/osac-project/osac/blob/main/docs/guides/installation/helm-deployment-guide.md#troubleshooting)
- "Troubleshooting" and "Debug Commands" in
  [`osac-installer/README.md`](https://github.com/osac-project/osac/blob/main/osac-installer/README.md)

---

## 12. Glossary

- **Phases** — Phase 1 is the prerequisite Operators (`osac-deps`) and the
  infrastructure layer (`osac-infra`). Phase 2 is the OSAC platform (`osac`
  chart). This guide installs phase 2 only.
- **`osac-deps`, `osac-infra`, `osac`** — The three Helm charts. The `osac`
  chart is published as an OCI artifact; the other two are only in the monorepo.
- **Published-chart install** — Installing only the published `osac` chart
  (phase 2) onto a cluster where the prerequisites already exist. See the
  [Helm Deployment Guide](https://github.com/osac-project/osac/blob/main/docs/guides/installation/helm-deployment-guide.md)
  if you need to set up the prerequisites yourself.
- **Operand** — The custom resource that an Operator reconciles, for example
  `HyperConverged` for OpenShift Virtualization, `LVMCluster` for LVM Storage,
  or `IPAddressPool` for MetalLB. The `osac-infra` chart creates these.
- **ClusterServiceVersion (CSV)** — The OLM record of an installed Operator
  version. A status of `Succeeded` means that the Operator is running.
- **Hub** — The OpenShift Container Platform cluster that the OSAC Operator and
  AAP run on and that provisions resources. This guide assumes that the
  Fulfillment Service runs on the hub, that is, the hub and the cluster
  running the Fulfillment Service are the same cluster.
- **Tenant** — An isolation boundary in OSAC, identified by the
  `osac.openshift.io/tenant` annotation. `shared` is the built-in tenant used
  for resources shared across all tenants.
- **VMaaS, CaaS, BMaaS, MaaS** — Virtual machine, cluster, bare-metal host,
  and raw-metal access as a service (per the chart schema; distinct from
  BMaaS). The service tiers, toggled by `global.services.*`.
- **ComputeInstance, Cluster, BareMetalInstance** — The user-facing resources
  for a virtual machine, a hosted cluster, and a bare-metal machine.
- **Instance group** — An AAP execution group with its own `ConfigMap` and
  Secret of environment variables, for example `cluster-fulfillment`,
  `network-fulfillment`, `storage-fulfillment`, or `publish-templates`.
- **Network backend (`NETWORK_CLASS`)** — How CaaS clusters get networking,
  driven by the `NetworkClass` custom resource's registered fabric and
  Kubernetes managers. Netris is the documented fabric manager.
- **DNS backend (`DNS_CLASS`)** — How CaaS clusters get DNS records:
  `dns.route53.dns` (default, AWS Route 53).
- **config-as-code** — The Ansible content that the `osac-aap-bootstrap` job
  loads into AAP. Its subscription manifest is stored in the
  `config-as-code-manifest-ig` Secret.
- **Bundled compared with external** — Bundled PostgreSQL is for testing, and
  OpenBao is for testing and CI. Production deployments use external services.
