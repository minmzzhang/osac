# Upgrading OSAC

> **Applies to:** OSAC 0.4.x and 0.5.x Helm installations on OpenShift.

This runbook assumes a working OSAC installation prepared as described in the
[customer installation guide](customer-install-guide.md): the platform runs in
the `osac` namespace, and the prerequisite Operators, identity provider,
PostgreSQL, and secret store are provided by your platform administrators.

OSAC has two upgrade paths with different rules:

- **Patch** (0.4.x → 0.4.y) is an in-service upgrade: existing workloads keep
  running. See [§9](#9-patch-upgrade-in-service).
- **Minor** (0.4 → 0.5) is supported only on an empty deployment. See
  [§10](#10-minor-release-upgrade-04--05-empty-deployment) and
  [§2](#2-current-limitations).

---

## Contents

- [Upgrade matrix](#upgrade-matrix)
- [Before you start](#before-you-start)
- [Overview](#overview)
- [1. Upgrade paths and compatibility](#1-upgrade-paths-and-compatibility)
- [2. Current limitations](#2-current-limitations)
- [3. Prerequisites](#3-prerequisites)
  - [3.1 Cluster and client tools](#31-cluster-and-client-tools)
  - [3.2 Platform Operators and components](#32-platform-operators-and-components)
  - [3.3 Certificates and trust](#33-certificates-and-trust)
  - [3.4 Identity provider](#34-identity-provider)
  - [3.5 PostgreSQL, Kafka, and secret store](#35-postgresql-kafka-and-secret-store)
  - [3.6 Provider networking (only when enabled)](#36-provider-networking-only-when-enabled)
- [4. Service enablement and namespace checks](#4-service-enablement-and-namespace-checks)
  - [4.1 Read what the release provisions](#41-read-what-the-release-provisions)
  - [4.2 Namespace checklist](#42-namespace-checklist)
  - [4.3 VMaaS](#43-vmaas)
  - [4.4 CaaS](#44-caas)
  - [4.5 BMaaS](#45-bmaas)
  - [4.6 Networking and storage records](#46-networking-and-storage-records)
- [5. Helm-only snapshot](#5-helm-only-snapshot)
  - [Extract the required values from the live release](#extract-the-required-values-from-the-live-release)
- [6. Required values to read from the current release](#6-required-values-to-read-from-the-current-release)
- [7. Prepare the target chart, images, and values](#7-prepare-the-target-chart-images-and-values)
  - [7.1 Values file](#71-values-file)
  - [7.2 Pin images](#72-pin-images)
  - [7.3 Choose the chart](#73-choose-the-chart)
  - [7.4 Define the shared Helm arguments once](#74-define-the-shared-helm-arguments-once)
- [8. Helm pre-flight](#8-helm-pre-flight)
  - [8.1 Diff your live values against the new chart defaults](#81-diff-your-live-values-against-the-new-chart-defaults)
  - [8.2 Render the new chart and diff manifests](#82-render-the-new-chart-and-diff-manifests)
  - [8.3 Cluster-side dry run (Helm 4)](#83-cluster-side-dry-run-helm-4)
  - [8.4 Hook inventory](#84-hook-inventory)
  - [8.5 What Helm cannot show](#85-what-helm-cannot-show)
- [9. Patch upgrade (in-service)](#9-patch-upgrade-in-service)
- [10. Minor release upgrade (0.4 → 0.5, empty deployment)](#10-minor-release-upgrade-04--05-empty-deployment)
  - [10.1 Guard precondition](#101-guard-precondition)
  - [10.2 Values and schema deltas: 0.4 → 0.5](#102-values-and-schema-deltas-04--05)
  - [10.3 Run the upgrade](#103-run-the-upgrade)
  - [10.4 Reinstall alternative](#104-reinstall-alternative)
  - [10.5 Reconnecting to existing services](#105-reconnecting-to-existing-services)
- [11. Verify after either upgrade](#11-verify-after-either-upgrade)
- [12. Rollback](#12-rollback)
- [13. Gotcha checklist](#13-gotcha-checklist)
- [Appendix A — the few things Helm cannot answer (optional)](#appendix-a--the-few-things-helm-cannot-answer-optional)
  - [A.1 Guard tables (fulfillment DB)](#a1-guard-tables-fulfillment-db)
  - [A.2 Database backup and restore](#a2-database-backup-and-restore)
  - [A.3 `NetworkClass` status, storage records, tenant conditions](#a3-networkclass-status-storage-records-tenant-conditions)
- [Related documentation](#related-documentation)

---

## Upgrade matrix

| Upgrade | Source → target | Workloads and networking records | Method | Data | Downtime | Guide |
|---|---|---|---|---|---|---|
| Patch | 0.4.x → 0.4.y | Any | `helm upgrade` | Database migrated in place; workloads kept | Control plane only; running VMs and clusters continue | [§9](#9-patch-upgrade-in-service) |
| Minor | 0.4.x → 0.5.x | None | `helm upgrade` | Same database migrated; tenants, catalog, and storage registrations kept | Full platform restart | [§10](#10-minor-release-upgrade-04--05-empty-deployment) |
| Minor | 0.4.x → 0.5.x | None | Reinstall `osac` and reconnect services | Same database migrated; chart-managed AAP history lost | Full platform restart | [§10.4](#104-reinstall-alternative)–[§10.5](#105-reconnecting-to-existing-services) |
| Minor | 0.4.x → 0.5.x | Any | **Not supported** — remove the resources first, or reinstall with a new database | Workloads are deleted with the resources | — | [§2](#2-current-limitations) |

"Workloads and networking records" means rows in `virtual_networks`,
`external_ip_pools`, `compute_instances`, `bare_metal_instances`, or
`clusters`.

---

## Before you start

Define these once per shell. Every command below reuses them.

```bash
export VALUES_DIR="${VALUES_DIR:-$HOME/osac-values}"  # your values files
export NS=osac                                        # OSAC release namespace
export WORK_DIR="${WORK_DIR:-$HOME/osac-upgrade}"     # scratch space
```

`VALUES_DIR` holds the values file you pass to Helm. The tools used below are
`helm` 4.x, `yq` (mikefarah) v4, and `jq`; the optional checks in
[Appendix A](#appendix-a--the-few-things-helm-cannot-answer-optional) also
need `oc` or `kubectl`.

---

## Overview

1. **Everything below drives off `helm get values`, `helm get manifest`,
   `helm get hooks`, `helm history`, `helm template`, `helm upgrade`,
   `helm rollback`.**
2. **Do not trust the values file for the deployment identity.** Read
   `global.osacDeploymentId` from `helm get values -a` and cross-check the
   retained ConfigMap in `helm get manifest`. A hand-written values file can
   disagree with the live release, so always pass the value you read with
   `--set-string global.osacDeploymentId=...`; `--set` wins over `-f`.
3. **Never use `--reuse-values`.** Pass the values file + the same `--set`
   flags every time.
4. **Manage only the `osac` release.** Platform services — Operators,
   identity provider, PostgreSQL, and secret store — are outside this guide
   and owned by your platform administrators.
5. **Pin image tags.** A released chart already pins them; a chart built
   from a development commit uses `:latest` placeholders.
6. **Helm 4 (`v4.2.2`):** add `--force-conflicts` to upgrades, dry-runs and
   rollbacks of the `osac` release.

---

## 1. Upgrade paths and compatibility

The matrix above summarizes the supported paths. The fulfillment service
enforces these rules at startup:

- `global.networking.provisioningEnabled` never changes.
- Enabling BMaaS/CaaS while a VirtualNetwork exists is refused.
- A different **major.minor** release is refused while any row exists in
  `virtual_networks`, `external_ip_pools`, `compute_instances`,
  `bare_metal_instances`, or `clusters`. Exact message:

  > in-place upgrade or downgrade between minor releases is not supported;
  > delete all workloads and networking resources or reinstall

---

## 2. Current limitations

These limitations apply to OSAC 0.4 → 0.5 and are enforced by the product, not
by this runbook.

- **Minor upgrades with workloads are not supported.** The fulfillment service
  refuses to start when the stored release and the running release differ at
  the major.minor level and any row exists in `virtual_networks`,
  `external_ip_pools`, `compute_instances`, `bare_metal_instances`, or
  `clusters`. This guide therefore covers an **empty deployment** only.
- **A maintenance window does not change this.** The check runs at service
  startup against stored records, not against runtime availability. Draining
  traffic or scaling down the control plane is not enough.
- **There is no workload migration.** Deleting those resources deletes the
  actual VMs, clusters, and networking state. This release provides no export,
  import, or live-migration path across a minor release.
- **Reinstall does not adopt running workloads.** Backing up values,
  uninstalling, and reinstalling reconnects platform services but not tenant
  workloads. The five guard tables still block startup, and this release has
  no mechanism to adopt VMs, clusters, or networks created by 0.4.
- **Draining is out of scope for this guide.** If any of the five tables has
  rows, stop and plan a reinstall or a separate workload-recreation exercise
  with the service owners.
- **Migrations are forward-only.** The service migrates the database before it
  runs the compatibility check, and there is no downgrade. Restore a backup to
  go back.
- **The guard is version-string based.** It compares semantic versions and
  skips the check for non-semver builds, for example a bare `dev` build. That
  does not make such an upgrade supported.
- **Patch upgrades are different.** 0.4.x → 0.4.y patch upgrades can run in
  place with workloads and a maintenance window.

Enforcement:
[`installation_settings.go`](../../../fulfillment-service/internal/startup/installation_settings.go)
(`ReleaseChangedMessage`, the five-table check, and `releaseChanged`). See
[§1](#1-upgrade-paths-and-compatibility).

---

## 3. Prerequisites

The upgrade runs the same install checks as a fresh installation. Confirm every
prerequisite before you change the release, and ask your platform
administrators for anything you do not own. The authoritative component list
is Table 2.1 of the
[customer installation guide](customer-install-guide.md#table-21-platform-operators-and-components).

### 3.1 Cluster and client tools

| Requirement | Check |
|---|---|
| OpenShift Container Platform 4.17 or later with cluster-admin access | `oc whoami`, `oc get clusterversion version` |
| Helm 3.18 or later, or Helm 4.x | `helm version --short` |
| `yq` (mikefarah) v4 and `jq` | `yq --version`, `jq --version` |
| `psql` (optional database checks) | `psql --version` |

```bash
oc whoami
oc get clusterversion version -o jsonpath='{.status.desired.version}{"\n"}'
oc get namespace "$NS"
helm version --short
```

### 3.2 Platform Operators and components

| Component | Namespace | Required for |
|---|---|---|
| cert-manager Operator for Red Hat OpenShift | `cert-manager-operator` (operands in `cert-manager`) | All services |
| trust-manager | `cert-manager` | All services |
| Configured certificate issuer (default `default-ca` `ClusterIssuer`) | Cluster-scoped | All services |
| Keycloak with the `osac` realm | Provider-defined | All services |
| Red Hat Ansible Automation Platform | `ansible-aap` | All services |
| Streams for Apache Kafka, or an external broker | Provider-defined | Fulfillment events and metering |
| OpenShift Virtualization | `openshift-cnv` | VMaaS |
| MetalLB Operator | `metallb-system` | CaaS and BMaaS publication |
| multicluster engine Operator | `multicluster-engine` | CaaS |
| Kubernetes NMState Operator | `openshift-nmstate` | VMaaS with provider networking |
| EVPN host networking (FRR, VTEP, BGP) | `openshift-frr-k8s` | VMaaS with provider networking |
| Dynamic StorageClass | Provider-defined | VMaaS workloads |
| External PostgreSQL 18 or later | Provider-defined | Production (`dbInit.host`) |
| External Vault or OpenBao | Provider-defined | Production (`service.vault.*`) |

Every prerequisite Operator must report a `Succeeded` CSV:

```bash
oc get csv -A \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,PHASE:.status.phase' \
  | awk 'NR == 1 || $2 ~ /^(aap-operator|cert-manager-operator|kubernetes-nmstate-operator|kubevirt-hyperconverged-operator|lvms-operator|metallb-operator|multicluster-engine)\./'
```

### 3.3 Certificates and trust

The issuer named by `service.certs.issuerRef` must be `Ready`,
trust-manager must run in `cert-manager`, and the `ConfigMap` named by
`service.certs.caBundle.configMap` must contain `bundle.pem` in the release
namespace.

```bash
oc get clusterissuer default-ca \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}{"\n"}'
oc get deploy -n cert-manager trust-manager
oc get configmap ca-bundle -n "$NS" -o jsonpath='{.data.bundle\.pem}' | head -1
```

### 3.4 Identity provider

Keycloak must serve the `osac` realm with the `osac-controller` and `osac-ui`
clients, and the `fulfillment-controller-credentials` Secret must hold
`client-id` and `client-secret`.

```bash
oc get secret -n "$NS" fulfillment-controller-credentials \
  -o jsonpath='{.data.client-id} {.data.client-secret}' | wc -w
```

### 3.5 PostgreSQL, Kafka, and secret store

Production requires an external PostgreSQL 18 or later database reached
through `dbInit.host`, a Kafka broker, and a Vault-compatible secret store
configured through `service.vault.*`. The upgrade applies database migrations,
so take a backup first (Appendix A.2).

```bash
helm get values osac -n "$NS" -a -o yaml \
| yq -o json | jq '{dbInitHost: .dbInit.host,
                    kafkaEnabled: .kafka.enabled,
                    vaultEndpoint: .service.vault.endpoint}'
```

### 3.6 Provider networking (only when enabled)

With `global.networking.provisioningEnabled: true`, the Fabric Manager
registration named by `global.networking.fabricManager` must exist, and with
VMaaS the EVPN host networking (NMState, FRR, VTEP, BGP) and the provider
fabric must be ready. With provisioning disabled, confirm the setting and skip
the fabric checks.

---

## 4. Service enablement and namespace checks

### 4.1 Read what the release provisions

```bash
helm get values osac -n "$NS" -a -o yaml | yq -o json | jq '.global.services'
```

`global.services.maas.enabled` is an enablement signal only; this release has
no MaaS runtime components. Check the components for every service that is
`true`.

### 4.2 Namespace checklist

```bash
for ns in cert-manager-operator cert-manager ansible-aap keycloak \
          openshift-cnv openshift-storage metallb-system \
          multicluster-engine openshift-nmstate openshift-frr-k8s osac-kafka
do
  printf '%-24s ' "$ns"
  oc get namespace "$ns" -o jsonpath='{.status.phase}' 2>/dev/null || echo "not found"
done
```

| Namespace | Needed when | What to check |
|---|---|---|
| `cert-manager-operator`, `cert-manager` | Always | CSVs `Succeeded`; trust-manager ready |
| `ansible-aap` | Always (chart-managed AAP) | AAP CSV `Succeeded`; AAP custom resource ready |
| `keycloak` | Always (bundled Keycloak) | Keycloak ready; `osac` realm present |
| `openshift-cnv` | VMaaS | CNV CSV `Succeeded`; `HyperConverged` available |
| `openshift-storage` | VMaaS storage (LVM or ODF) | `StorageClass` present; storage CSV `Succeeded` |
| `metallb-system` | CaaS, BMaaS publication | MetalLB CSV `Succeeded`; `IPAddressPool` present |
| `multicluster-engine` | CaaS | MCE CSV `Succeeded`; MCE ready |
| `openshift-nmstate` | VMaaS with provider networking | NMState CSV `Succeeded` |
| `openshift-frr-k8s` | VMaaS with provider networking | `FRRConfiguration` present and BGP established |
| Kafka namespace (default `osac-kafka`) | Always | `Kafka` resource `Ready` |

### 4.3 VMaaS

```bash
oc get csv -n openshift-cnv | grep -i kubevirt
oc get hyperconverged -n openshift-cnv
oc get storageclass
oc get lvmcluster -A 2>/dev/null || true
```

### 4.4 CaaS

```bash
oc get csv -n multicluster-engine
oc get ipaddresspool -n metallb-system
helm get values osac -n "$NS" -a -o yaml \
| yq '.aap.instanceGroups.clusterFulfillment.config
      | {"DNS_CLASS": .DNS_CLASS,
         "DNS_ZONE": .DNS_ZONE,
         "EXTERNAL_ACCESS_BASE_DOMAIN": .EXTERNAL_ACCESS_BASE_DOMAIN}'
```

### 4.5 BMaaS

```bash
oc get ipaddresspool -n metallb-system
helm get values osac -n "$NS" -a -o yaml \
| yq '.bmf | {"metal3": .metal3.enabled,
              "netbox": .netbox.enabled,
              "bcm": .bcm.enabled}'
```

### 4.6 Networking and storage records

Use Appendix A.3 to confirm the `NetworkClass`, storage backends and tiers,
and tenant conditions through the fulfillment API.

---

## 5. Helm-only snapshot

```bash
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p "$WORK_DIR/$STAMP"
cd "$WORK_DIR/$STAMP"

# Releases, history, live status
helm list -n "$NS" -o table                     | tee helm-list.txt
helm list -n "$NS" -o json                       | tee helm-list-osac.json
helm history osac -n "$NS"                       | tee helm-history-0.4.txt
helm status osac -n "$NS" -o yaml               | tee helm-status-0.4.txt

# Values (user-supplied and merged-with-defaults)
helm get values osac -n "$NS"     -o yaml > osac-values-user.yaml
helm get values osac -n "$NS" -a  -o yaml > osac-values-all.yaml

# Everything Helm rendered: CRDs, ConfigMaps, workloads (hooks are separate)
helm get manifest osac -n "$NS"   > osac-manifest-0.4.yaml

# Hooks (pre/post-install, pre/post-upgrade) actually in this release
helm get hooks osac -n "$NS"      > osac-hooks-0.4.yaml
```

### Extract the required values from the live release

```bash
# Authoritative deployment identity, as stored in this Helm release
export LIVE_OSAC_DEPLOYMENT_ID=$(helm get values osac -n "$NS" -a -o yaml \
  | yq -r '.global.osacDeploymentId')

# Cross-check against the retained ConfigMap Helm rendered
helm get manifest osac -n "$NS" \
  | yq -r 'select(.kind=="ConfigMap" and (.metadata.name|test("-osac-")))
           | .metadata.name + " => " + .data.osacDeploymentId'

echo "deployment id: $LIVE_OSAC_DEPLOYMENT_ID"
```

If the two disagree, the ConfigMap wins (the chart fails the render on a
mismatch). For release `osac` it is `osac-osac-eedb463a`.

```bash
# Cluster domain + hostnames + service/network selection
helm get values osac -n "$NS" -a -o yaml | yq -o json \
| jq '{
    clusterDomain:  .global.clusterDomain,
    deploymentId:   .global.osacDeploymentId,
    services:       .global.services,
    networking:     .global.networking,
    externalHost:   .service.externalHostname,
    internalHost:   .service.internalHostname,
    uiHost:         .ui.externalHostname,
    oidcIssuer:     .ui.auth.oidcIssuerURL
  }'

# Images actually requested by the current release
helm get values osac -n "$NS" -a -o yaml | yq -o json \
| jq '{
    operator:     .operator.image.tag,
    service:      .service.images.service.tag,
    envoy:        .service.images.envoy.tag,
    ui:           .ui.images.ui,
    metering:     .metering.image.tag,
    bmf:          .bmf.image.tag,
    aapBootstrap: .aap.bootstrap.image,
    aapEE:        .aap.configAsCode.eeImage
  }'

# Chart version and app version
helm list -n "$NS" -o json | jq -r '.[0] | "\(.chart) app=\(.app_version) rev=\(.revision) status=\(.status)"'
```

Save these outputs; they are your "before" state.

---

## 6. Required values to read from the current release

Every value below is mandatory to verify against the live release before
upgrading. A wrong or missing value either fails the rollout (the
fixed-at-install checks) or silently changes the deployment.

Legend: **KEEP** = must match the live release; **READ** = take from Helm
output, not the file; **NEW** = added in 0.5; **NO** = never copy.

| Required value | Read it with | Action |
|---|---|---|
| `global.osacDeploymentId` | `helm get values -a` + manifest ConfigMap (§5) | **READ.** Take it from the live release; never from a hand-written file. |
| `global.clusterDomain` | `helm get values -a` | **READ.** The cluster ingress domain, for example `apps.example.com`. |
| `global.services.*` | `helm get values -a` | **KEEP** `vmaas: true`, others `false`. |
| `global.networking.provisioningEnabled` | `helm get values -a` | **KEEP** `false`; fixed at first install. |
| `global.networking.fabricManager` / `k8sManager` | `helm get values -a` | **KEEP** `netris` / `cudn_evpn`. |
| `global.networking.excludedIpv4Cidrs` | `helm get values -a` | **KEEP** as-is. |
| `global.networking.networkClass.*` | `helm get values -a` | **KEEP** exactly; the installer fails on any change. |
| `global.fulfillmentTrust.enabled` | `helm get values -a` | **KEEP** `false`. |
| `operator.aap.tokenSecret` | `helm get values -a` | **KEEP**; the hook regenerates the Secret. |
| `operator.aap.insecureSkipVerify` | `helm get values -a` | **KEEP** `"true"` only for self-signed AAP; `"false"` in production. |
| `operator.fulfillment.credentials.*` | `helm get values -a` | **KEEP** (Secret is provided with your identity-provider setup). |
| `service.externalHostname` / `internalHostname` | `helm get values -a` | **READ.** Recompute from `$DOMAIN` if missing. |
| `service.certs.*`, `service.auth.*`, `service.idp.*` | `helm get values -a` | **KEEP**. |
| `service.vault.*` | `helm get values -a` | **KEEP.** External secret-store endpoint, namespace, mounts, and roles. |
| `service.database.connection` | `helm get values -a` | **KEEP** secret names only; URLs derive from your PostgreSQL host (`dbInit.host`). |
| `ui.auth.oidcIssuerURL`, `ui.externalHostname` | `helm get values -a` | **READ** from the release. |
| `aap.*` | `helm get values -a` | **KEEP** (no Netris credentials in your file). |
| `metering.*` | `helm get values -a` | **KEEP** (adapters disabled). |
| `lvms.enabled` | `helm get values -a` | **KEEP** the live value. Production guidance is `false` (storage is registered by your administrators). |
| `validation.enabled`, `hubAccess.enabled` | `helm get values -a` | **KEEP** `true`. |
| `image` / `tag` values | `helm get values -a` | **NO.** Do not carry 0.4 tags forward; pin to the new `sha-` (§7). |
| DB client certs, `osac-aap-api-token`, Helm release secrets | — | **NO.** Regenerated. |
| Identity-provider and database state | — | **NO.** Managed by your platform administrators; never re-create or copy. |

---

## 7. Prepare the target chart, images, and values

### 7.1 Values file

```bash
cd "$VALUES_DIR"
cp my-values.yaml my-values.target.yaml
```

Required edit — use the live value from §5:

```yaml
global:
  osacDeploymentId: "<LIVE_OSAC_DEPLOYMENT_ID>"   # likely "osac/osac"
```

Everything else can stay byte-for-byte.

To reuse the live values safely, export the user-supplied values and pass that
file with `-f` instead of `--reuse-values`:

```bash
helm get values osac -n "$NS" -o yaml > "$VALUES_DIR/live-user-values.yaml"
```

`--reuse-values` reuses the old chart's coalesced values, so the new chart's
defaults — including image tags and new keys — are ignored.

### 7.2 Pin images

A chart released from `ghcr.io/osac-project/charts` already pins consistent
image tags; skip this section when you install `osac/v0.5.x`. A chart built
from a development commit uses `:latest` placeholders: pick that commit and
use the same `sha-` for every component.

```bash
export SHA="<short-0.5-commit>"
IMAGE_ARGS=(
  --set operator.image.tag=sha-$SHA
  --set service.images.service.tag=sha-$SHA
  --set ui.images.ui=ghcr.io/osac-project/osac-ui:sha-$SHA
  --set metering.image.tag=sha-$SHA
  --set bmf.image.tag=sha-$SHA
  --set aap.bootstrap.image=ghcr.io/osac-project/osac-aap:sha-$SHA
  --set aap.configAsCode.eeImage=ghcr.io/osac-project/osac-aap:sha-$SHA
)
```

If you install a stamped nightly or a `0.5.x` release chart, its values already
carry consistent tags — set `IMAGE_ARGS=()`.

### 7.3 Choose the chart

```bash
helm show chart oci://ghcr.io/osac-project/charts/osac --version "<0.5.x-or-nightly>"
export CHART=oci://ghcr.io/osac-project/charts/osac
```

### 7.4 Define the shared Helm arguments once

```bash
export NS=osac
export DOMAIN=${DOMAIN:-$(helm get values osac -n "$NS" -a -o yaml \
  | yq -r '.global.clusterDomain')}

COMMON_ARGS=(
  -f "$VALUES_DIR/my-values.target.yaml"
  --set-string global.osacDeploymentId="$LIVE_OSAC_DEPLOYMENT_ID"
  --set global.clusterDomain="$DOMAIN"
  --set service.externalHostname="fulfillment-api-$NS.$DOMAIN"
  --set service.internalHostname="fulfillment-internal-api-$NS.$DOMAIN"
  --set ui.auth.oidcIssuerURL="https://keycloak-keycloak.$DOMAIN/realms/osac"
  --set ui.externalHostname="osac-ui-$NS.$DOMAIN"
)
```

Everything below reuses `"${COMMON_ARGS[@]}" "${IMAGE_ARGS[@]}"`.

---

## 8. Helm pre-flight

### 8.1 Diff your live values against the new chart defaults

```bash
cd "$(ls -dt "$WORK_DIR"/* | head -1)"

# Defaults of the chart you are about to install
helm show values "$CHART" > new-chart-defaults.yaml

# Sorted diff makes renamed/removed defaults obvious
diff -u <(yq -P 'sort_keys(..)' osac-values-all.yaml) \
        <(yq -P 'sort_keys(..)' new-chart-defaults.yaml) | less
```

Everything you intentionally override is expected to show up; anything you
did **not** set but that appears only on the left is a default that 0.5
removed or renamed.

### 8.2 Render the new chart and diff manifests

```bash
helm template osac "$CHART" -n "$NS" \
  "${COMMON_ARGS[@]}" "${IMAGE_ARGS[@]}" > osac-manifest-0.5.yaml

diff -u osac-manifest-0.4.yaml osac-manifest-0.5.yaml | less
```

Expected diff for your VMaaS values: image tags/`app.kubernetes.io/version`
labels, plus the `ClusterOrder` CRD. Any new/removed object or changed
hostname/annotation is worth understanding before you apply.

### 8.3 Cluster-side dry run (Helm 4)

```bash
helm upgrade osac "$CHART" -n "$NS" \
  "${COMMON_ARGS[@]}" "${IMAGE_ARGS[@]}" \
  --dry-run=server --force-conflicts --hide-secret | less
```

This validates the render against the live API (CRDs, RBAC, schema) without
running hooks or changing anything.

### 8.4 Hook inventory

```bash
helm get hooks osac -n "$NS"
```

Successful hooks are deleted by Helm, so this normally shows only retained
failed hooks. On the upgrade run you expect these to succeed:

- `pre-install,pre-upgrade`: `db-init` (creates DBs/roles), validation/CA hooks
- `post-install,post-upgrade`: `create-api-token` (AAP), `create-network-class`,
  `publish-templates`, `register-local-storage` (with `lvms.enabled: true`)

### 8.5 What Helm cannot show

The DB guard counts, `NetworkClass` status, and Secret contents are not in the
Helm release. If you need them, Appendix A has the minimal non-Helm checks.
The Helm-only proxy is: the values in §5/§6 are exactly what the installer and
service compare against, so copying them verbatim is the safe path.

---

## 9. Patch upgrade (in-service)

Use this when the target is a patch release in the same minor line, for
example 0.4.0 → 0.4.1. Existing workloads keep running because the
minor-version guard does not run.

- Expect brief API, console, and UI downtime while the control plane rolls.
  Running VMs and clusters are not restarted.
- The release still runs database migrations; back up first (Appendix A.2).
- Values should not change for a patch. Keep the live values and do not apply
  the 0.4 → 0.5 deltas from §10.2.
- Use a **newer** patch version. Installing an older release is a downgrade,
  and migrations are forward-only.

Confirm the target is in the same line:

```bash
helm list -n "$NS" -o json | jq -r '.[0].chart'
helm show chart "$CHART" | grep '^version:'
```

Run the upgrade with the shared arguments from §7.4:

```bash
helm upgrade --install osac "$CHART" -n "$NS" \
  "${COMMON_ARGS[@]}" "${IMAGE_ARGS[@]}" \
  --force-conflicts --wait --timeout 40m
```

Then continue with §11 to verify and §12 to roll back.

---

## 10. Minor release upgrade (0.4 → 0.5, empty deployment)

Use this when the target changes the minor line, for example 0.4 → 0.5. It is
supported only when the deployment has no ComputeInstances, Clusters,
BareMetalInstances, ExternalIPPools, or VirtualNetworks. See §2.

### 10.1 Guard precondition

This guide targets an empty deployment. Confirm the five guard tables are
empty (Appendix A.1). If any has rows, stop: draining is out of scope, and a
workload-preserving minor upgrade is not supported
([§2](#2-current-limitations)).

### 10.2 Values and schema deltas: 0.4 → 0.5

1. `global.billing` added (optional; `provider: null`).
2. Netris settings moved to
   `aap.instanceGroups.networkFulfillment.{config,secret}`. You set none.
3. `operator.aap.tokenSecret` and every other key in a 0.4 values file still exist.
4. No VMaaS CRDs changed; only `ClusterOrder` (CaaS) did.

### 10.3 Run the upgrade

```bash
export NS=osac
helm upgrade --install osac "$CHART" -n "$NS" --create-namespace \
  "${COMMON_ARGS[@]}" "${IMAGE_ARGS[@]}" \
  --force-conflicts \
  --wait --timeout 40m
```

Why these flags:

- `--force-conflicts` — Helm 4 + AAP CR field ownership; Helm 3 rejects it.
- no `--reuse-values` — it reuses old chart defaults and image tags; see §7.1.
- `--wait` — Helm returns only after hooks and workloads settle (AAP bootstrap
  can take 10–40 min).

### 10.4 Reinstall alternative

Reinstall produces the same 0.5 deployment and reconnects to the same platform
services. It is an alternative to §10.3, not a way around the guard in §10.1:
with the same database, the five tables must still be empty. Use it when you
accept losing chart-managed AAP job history and regenerated secrets.

> **Warning:** `helm uninstall osac` does **not** delete tenant workloads.
> ComputeInstances, VirtualNetworks, ExternalIPPools, ClusterOrders,
> BareMetalInstances, and the provider resources behind them are created at
> runtime, not by Helm. The uninstall removes the operator, so those resources
> become unmanaged and their finalizers cannot run, while their database
> records remain and still block the 0.5 service. Delete the resources through
> the fulfillment API while 0.4 is running, then reinstall.

```bash
helm history osac -n "$NS"           # note the last good revision
helm uninstall osac -n "$NS"         # keeps the namespace, CRDs, retained ConfigMap,
                                     # platform Secrets, and manual Secrets
helm list -n "$NS" | grep -E '^osac[[:space:]]'   # confirm it is gone
```

There is nothing to re-read with Helm after the uninstall: the chart looks up
the retained identity ConfigMap at render time and fails if the value you pass
differs. Just install with the §10.3 command (`helm upgrade --install` works on
an empty release name).

What survives: the platform services (identity provider, PostgreSQL, secret
store, CA bundle), the controller credentials Secret,
`config-as-code-manifest-ig`, and the CRDs.
What is lost: AAP CR + its PVCs (job history), `osac-db-*` and
`osac-aap-api-token` Secrets (recreated).

### 10.5 Reconnecting to existing services

Reinstall replaces only the `osac` release. Platform services are referenced
from values and Secrets, not owned by the release, so the new install
reconnects to them when you keep the same configuration:

| Service | Values to keep | Survives uninstall |
|---|---|---|
| PostgreSQL | `dbInit.host`, `service.database.connection` | Yes (external database); the 0.5 service migrates the schema in place |
| Keycloak / IdP | `service.auth.issuerUrl`, `service.idp.*`, `fulfillment-controller-credentials` | Yes (realm and clients stay in Keycloak) |
| Vault / OpenBao | `service.vault.*` | Yes |
| Kafka | `kafka.enabled: false`, `service.kafka.connection` | Yes (broker and topics) |
| External AAP | `operator.aap.url`, `operator.aap.tokenSecret`, `aap.aap.instance.enabled: false`, `aap.bootstrap.enabled: false`, `aap.apiToken.create: false`, `aap.instanceGroups.publishTemplates.enabled: false` | Yes |
| CA and issuer | `service.certs.issuerRef`, `service.certs.caBundle.configMap` | Yes |
| Fabric Manager registration | The registration `ConfigMap` (not Helm-owned) | Yes |

Chart-managed AAP is the exception: uninstalling `osac` removes the AAP
custom resource and its PVCs, so its job history is lost; the chart recreates
AAP and re-runs the bootstrap and token hooks.

The same guard still applies: the five tables in §10.1 must be empty before
the 0.5 service starts. Reinstalling against a **different, empty database**
reconnects to the services but has no tenant, catalog, storage-registration,
or NetworkClass records; treat it as a new deployment with the same
deployment identity.

---

## 11. Verify after either upgrade

Run these checks after either upgrade path. For a patch upgrade, also confirm
that existing ComputeInstances are still `RUNNING`.

```bash
export NS=osac

helm list -n "$NS" -o json | jq '.[0] | {name, revision, status, chart, app_version}'
helm history osac -n "$NS"
helm status osac -n "$NS" -o yaml | yq '.info.status'

# Failed hooks are retained and shown here; empty = all hooks succeeded
helm get hooks osac -n "$NS"

# Confirm the release now carries the values you intended
helm get values osac -n "$NS" -a -o yaml | yq -o json \
| jq '{deploymentId: .global.osacDeploymentId,
       domain: .global.clusterDomain,
       services: .global.services,
       networking: .global.networking}'

# Confirm the image tags actually deployed
helm get manifest osac -n "$NS" \
| yq -r 'select(.kind=="Deployment") | .metadata.name + " " + (.spec.template.spec.containers[].image)'
```

Then a functional check (UI login at `https://osac-ui-osac.$DOMAIN`, create a
test ComputeInstance) — Helm can tell you the release is `deployed`, not that
the VM booted.

If the release is `failed` / `pending-upgrade`, inspect the retained hook:

```bash
helm get hooks osac -n "$NS"
helm status osac -n "$NS" -o yaml
```

---

## 12. Rollback

```bash
helm history osac -n "$NS"
helm rollback osac "<LAST_GOOD_REVISION>" -n "$NS" \
  --wait --timeout 40m --force-conflicts
```

Caveats:

- Helm rollback reverts Kubernetes objects, **not** the database. Migrations
  are forward-only: restore the DB dump (Appendix A.2) before or after the
  rollback, otherwise the old binary may not match the schema.
- If you took the reinstall path (§10.4), rollback needs the old chart and values;
  `helm install` the 0.4 chart with the saved 0.4 values.

---

## 13. Gotcha checklist

**Both paths**

- [ ] `global.osacDeploymentId` read from `helm get values -a` / manifest.
- [ ] `global.networking.provisioningEnabled` unchanged.
- [ ] `global.services` unchanged.
- [ ] `networkClass.*` defaults unchanged.
- [ ] Fulfillment and metering databases backed up.
- [ ] No `--reuse-values`.
- [ ] Images pinned consistently (or installing a stamped release chart).
- [ ] `--force-conflicts` on Helm 4.

**Patch**

- [ ] The target chart is the same major.minor as the live release.
- [ ] Existing ComputeInstances are still `RUNNING` after the roll.

**Minor (empty deployment)**

- [ ] Five guard tables empty (Appendix A.1); no workloads or networking resources.
- [ ] `global.billing` left unset.
- [ ] Netris keys (if any) under
      `aap.instanceGroups.networkFulfillment.{config,secret}`.

---

## Appendix A — the few things Helm cannot answer (optional)

Helm stores rendered manifests and values, not database rows, Secret contents,
or controller status. Use these only if you need proof.

### A.1 Guard tables (fulfillment DB)

Connect with the fulfillment database credentials your platform
administrators provided (`FULFILLMENT_DB_URL` is the PostgreSQL connection
string):

```bash
psql "$FULFILLMENT_DB_URL" -c "
select 'virtual_networks'      t, count(*) from virtual_networks
union all select 'external_ip_pools',    count(*) from external_ip_pools
union all select 'compute_instances',    count(*) from compute_instances
union all select 'bare_metal_instances', count(*) from bare_metal_instances
union all select 'clusters',             count(*) from clusters;"
```

All five must be `0` for an in-place upgrade.

### A.2 Database backup and restore

```bash
STAMP=$(date +%Y%m%d-%H%M%S)
pg_dump "$FULFILLMENT_DB_URL" -Fc > service-$STAMP.dump
pg_dump "$METERING_DB_URL"    -Fc > metering-$STAMP.dump
```

Restore (rollback):

```bash
pg_restore -d "$FULFILLMENT_DB_URL" --clean --if-exists < service-$STAMP.dump
```

### A.3 `NetworkClass` status, storage records, tenant conditions

```bash
curl -sS "${CURL_CA_ARGS[@]}" -H "Authorization: Bearer $OSAC_ADMIN_TOKEN" \
  "$FULFILLMENT_API/api/private/v1/network_classes" | jq '.items[] | {id, spec, status}'
curl -sS "${CURL_CA_ARGS[@]}" -H "Authorization: Bearer $OSAC_ADMIN_TOKEN" \
  "$FULFILLMENT_API/api/private/v1/storage_backends" | jq '.items[] | {id, spec}'
curl -sS "${CURL_CA_ARGS[@]}" -H "Authorization: Bearer $OSAC_ADMIN_TOKEN" \
  "$FULFILLMENT_API/api/private/v1/storage_tiers" | jq '.items[] | {id, spec}'
curl -sS "${CURL_CA_ARGS[@]}" -H "Authorization: Bearer $OSAC_ADMIN_TOKEN" \
  "$FULFILLMENT_API/api/private/v1/tenants" \
  | jq '.items[] | {name: .metadata.name, conditions: .status.conditions}'
```

`NetworkClass.status.state` must be `NETWORK_CLASS_STATE_READY`.

---

## Related documentation

- [Customer installation guide](customer-install-guide.md) — values, fixed-at-install
  settings, verification, and uninstall.
- [External services support guide](external-services-support-guide.md) — running
  PostgreSQL, Keycloak, Kafka, or OpenBao outside the cluster.
- [Kafka configuration](kafka-configuration.md) — broker settings used by the
  fulfillment service and metering.
