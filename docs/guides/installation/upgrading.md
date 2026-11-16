# Upgrading OSAC: 0.4.0-rc.1 → 0.4.0-rc.3

> **Applies to:** patch upgrades within OSAC 0.4.x on OpenShift.

This runbook assumes a working OSAC installation prepared as described in the
[customer installation guide](customer-install-guide.md): the platform runs in
the `osac` namespace, and the prerequisite Operators, identity provider,
PostgreSQL, and secret store are provided by your platform administrators.

This guide covers the in-service patch upgrade from 0.4.0-rc.1 to 0.4.0-rc.3.
Patch upgrades keep existing workloads running; only the control plane rolls.
Minor upgrades (0.4 → 0.5) are not supported yet and are out of scope; see
[§10](#10-current-limitations).

---

## Contents

- [Upgrade matrix](#upgrade-matrix)
- [Before you start](#before-you-start)
- [Overview](#overview)
- [1. What changes in 0.4.0-rc.3](#1-what-changes-in-040-rc3)
- [2. Prerequisites](#2-prerequisites)
  - [2.1 Cluster and client tools](#21-cluster-and-client-tools)
  - [2.2 Platform Operators and components](#22-platform-operators-and-components)
  - [2.3 Certificates and trust](#23-certificates-and-trust)
  - [2.4 Identity provider](#24-identity-provider)
  - [2.5 PostgreSQL, Kafka, and secret store](#25-postgresql-kafka-and-secret-store)
  - [2.6 Provider networking (only when enabled)](#26-provider-networking-only-when-enabled)
- [3. Service enablement and namespace checks](#3-service-enablement-and-namespace-checks)
  - [3.1 Read what the release provisions](#31-read-what-the-release-provisions)
  - [3.2 Namespace checklist](#32-namespace-checklist)
  - [3.3 VMaaS](#33-vmaas)
  - [3.4 CaaS](#34-caas)
  - [3.5 BMaaS](#35-bmaas)
  - [3.6 Networking and storage records](#36-networking-and-storage-records)
- [4. Snapshot the current release](#4-snapshot-the-current-release)
  - [Extract the required values from the live release](#extract-the-required-values-from-the-live-release)
- [5. Required values to read from the current release](#5-required-values-to-read-from-the-current-release)
- [6. Prepare the target chart and values](#6-prepare-the-target-chart-and-values)
  - [6.1 Values file](#61-values-file)
  - [6.2 Choose the chart](#62-choose-the-chart)
  - [6.3 Define the shared Helm arguments once](#63-define-the-shared-helm-arguments-once)
- [7. Helm pre-flight](#7-helm-pre-flight)
  - [7.1 Diff your live values against the new chart defaults](#71-diff-your-live-values-against-the-new-chart-defaults)
  - [7.2 Render the new chart and diff manifests](#72-render-the-new-chart-and-diff-manifests)
  - [7.3 Cluster-side dry run (Helm 4)](#73-cluster-side-dry-run-helm-4)
  - [7.4 Hook inventory](#74-hook-inventory)
  - [7.5 What Helm cannot show](#75-what-helm-cannot-show)
- [8. Run the patch upgrade](#8-run-the-patch-upgrade)
- [9. Verify](#9-verify)
- [10. Current limitations](#10-current-limitations)
- [11. Rollback](#11-rollback)
- [12. Gotcha checklist](#12-gotcha-checklist)
- [Appendix A — optional checks](#appendix-a--optional-checks)
  - [A.1 Database backup and restore](#a1-database-backup-and-restore)
  - [A.2 NetworkClass, storage records, and tenant conditions](#a2-networkclass-storage-records-and-tenant-conditions)
- [Related documentation](#related-documentation)

---

## Upgrade matrix

| Upgrade | Tenant workloads + networking | Method | Supported |
|---|---|---|---|
| Patch 0.4.x → 0.4.y | Any | `helm upgrade` | Yes — this guide ([§8](#8-run-the-patch-upgrade)) |
| Minor 0.4 → 0.5 | Any | — | Not supported yet ([§10](#10-current-limitations)) |

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
[Appendix A](#appendix-a--optional-checks) also need `oc` or `kubectl` and
`psql`.

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
4. **Workloads keep running.** Patch upgrades do not run the minor-version
   guard. Only the OSAC control plane rolls, so expect brief API, console, and
   UI downtime while VMs and clusters continue.
5. **The release chart pins every image.** Do not carry 0.4.0-rc.1 tags
   forward. Use `--version 0.4.0-rc.3`.
6. **Helm 4 (`v4.2.2`):** add `--force-conflicts` to upgrades, dry-runs and
   rollbacks of the `osac` release.

---

## 1. What changes in 0.4.0-rc.3

- The umbrella chart is `osac-0.4.0-rc.3` (app version `0.4.0-rc.3`).
- The chart pins the operator, fulfillment service, AAP, UI, metering, and
  bare-metal operator images to `0.4.0-rc.3`. Envoy stays at `v1.33.0`.
- AAP config-as-code moves to branch `b900275`.
- No new values keys: the rc.1 and rc.3 chart values differ only in image tags
  and the AAP git branch.
- The new fulfillment service runs database migrations at startup, so take a
  backup first ([Appendix A.1](#a1-database-backup-and-restore)).

Verify the target chart before you start:

```bash
helm show chart oci://ghcr.io/osac-project/charts/osac --version 0.4.0-rc.3
```

The published chart resolves to
`sha256:c5651c1c611b244899cfcb6b92668f2b2163480698c94c866d315795f3ec6e8a`.

---

## 2. Prerequisites

The upgrade runs the same install checks as a fresh installation. Confirm every
prerequisite before you change the release, and ask your platform
administrators for anything you do not own. The authoritative component list
is Table 2.1 of the
[customer installation guide](customer-install-guide.md#table-21-platform-operators-and-components).

### 2.1 Cluster and client tools

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

### 2.2 Platform Operators and components

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

### 2.3 Certificates and trust

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

### 2.4 Identity provider

Keycloak must serve the `osac` realm with the `osac-controller` and `osac-ui`
clients, and the `fulfillment-controller-credentials` Secret must hold
`client-id` and `client-secret`.

```bash
oc get secret -n "$NS" fulfillment-controller-credentials \
  -o jsonpath='{.data.client-id} {.data.client-secret}' | wc -w
```

### 2.5 PostgreSQL, Kafka, and secret store

Production requires an external PostgreSQL 18 or later database reached
through `dbInit.host`, a Kafka broker, and a Vault-compatible secret store
configured through `service.vault.*`. The upgrade applies database migrations,
so take a backup first ([Appendix A.1](#a1-database-backup-and-restore)).

```bash
helm get values osac -n "$NS" -a -o yaml \
| yq -o json | jq '{dbInitHost: .dbInit.host,
                    kafkaEnabled: .kafka.enabled,
                    vaultEndpoint: .service.vault.endpoint}'
```

### 2.6 Provider networking (only when enabled)

With `global.networking.provisioningEnabled: true`, the Fabric Manager
registration named by `global.networking.fabricManager` must exist, and with
VMaaS the EVPN host networking (NMState, FRR, VTEP, BGP) and the provider
fabric must be ready. With provisioning disabled, confirm the setting and skip
the fabric checks.

---

## 3. Service enablement and namespace checks

### 3.1 Read what the release provisions

```bash
helm get values osac -n "$NS" -a -o yaml | yq -o json | jq '.global.services'
```

`global.services.maas.enabled` is an enablement signal only; this release has
no MaaS runtime components. Check the components for every service that is
`true`.

### 3.2 Namespace checklist

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

### 3.3 VMaaS

```bash
oc get csv -n openshift-cnv | grep -i kubevirt
oc get hyperconverged -n openshift-cnv
oc get storageclass
oc get lvmcluster -A 2>/dev/null || true
```

### 3.4 CaaS

```bash
oc get csv -n multicluster-engine
oc get ipaddresspool -n metallb-system
helm get values osac -n "$NS" -a -o yaml \
| yq '.aap.instanceGroups.clusterFulfillment.config
      | {"DNS_CLASS": .DNS_CLASS,
         "DNS_ZONE": .DNS_ZONE,
         "EXTERNAL_ACCESS_BASE_DOMAIN": .EXTERNAL_ACCESS_BASE_DOMAIN}'
```

### 3.5 BMaaS

```bash
oc get ipaddresspool -n metallb-system
helm get values osac -n "$NS" -a -o yaml \
| yq '.bmf | {"metal3": .metal3.enabled,
              "netbox": .netbox.enabled,
              "bcm": .bcm.enabled}'
```

### 3.6 Networking and storage records

Use [Appendix A.2](#a2-networkclass-storage-records-and-tenant-conditions) to
confirm the `NetworkClass`, storage backends and tiers, and tenant conditions
through the fulfillment API.

---

## 4. Snapshot the current release

```bash
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p "$WORK_DIR/$STAMP"
cd "$WORK_DIR/$STAMP"

# Releases, history, live status
helm list -n "$NS" -o table                     | tee helm-list.txt
helm list -n "$NS" -o json                       | tee helm-list-osac.json
helm history osac -n "$NS"                       | tee helm-history.txt
helm status osac -n "$NS" -o yaml               | tee helm-status.txt

# Values (user-supplied and merged-with-defaults)
helm get values osac -n "$NS"     -o yaml > osac-values-user.yaml
helm get values osac -n "$NS" -a  -o yaml > osac-values-all.yaml

# Everything Helm rendered: CRDs, ConfigMaps, workloads (hooks are separate)
helm get manifest osac -n "$NS"   > osac-manifest-rc1.yaml

# Hooks (pre/post-install, pre/post-upgrade) actually in this release
helm get hooks osac -n "$NS"      > osac-hooks-rc1.yaml
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

# Chart and app version
helm list -n "$NS" -o json | jq -r '.[0] | "\(.chart) app=\(.app_version) rev=\(.revision) status=\(.status)"'
```

Save these outputs; they are your "before" state. Every image should report
`0.4.0-rc.1` (Envoy: `v1.33.0`).

---

## 5. Required values to read from the current release

Every value below is mandatory to verify against the live release before
upgrading. A wrong value either fails the rollout or silently changes the
deployment. A patch upgrade should not require new values.

Legend: **KEEP** = must match the live release; **READ** = take from Helm
output, not the file; **NO** = never copy.

| Required value | Read it with | Action |
|---|---|---|
| `global.osacDeploymentId` | `helm get values -a` + manifest ConfigMap (§4) | **READ.** Take it from the live release; never from a hand-written file. |
| `global.clusterDomain` | `helm get values -a` | **READ.** The cluster ingress domain, for example `apps.example.com`. |
| `global.services.*` | `helm get values -a` | **KEEP** unchanged. |
| `global.networking.provisioningEnabled` | `helm get values -a` | **KEEP** unchanged; fixed at first install. |
| `global.networking.fabricManager` / `k8sManager` | `helm get values -a` | **KEEP** unchanged. |
| `global.networking.excludedIpv4Cidrs` | `helm get values -a` | **KEEP** as-is. |
| `global.networking.networkClass.*` | `helm get values -a` | **KEEP** exactly; the installer fails on any change. |
| `global.fulfillmentTrust.enabled` | `helm get values -a` | **KEEP** unchanged. |
| `operator.aap.tokenSecret` | `helm get values -a` | **KEEP**; the hook regenerates the Secret. |
| `operator.aap.insecureSkipVerify` | `helm get values -a` | **KEEP** the live value. |
| `operator.fulfillment.credentials.*` | `helm get values -a` | **KEEP** (Secret is provided with your identity-provider setup). |
| `service.externalHostname` / `internalHostname` | `helm get values -a` | **READ.** Recompute from `$DOMAIN` if missing. |
| `service.certs.*`, `service.auth.*`, `service.idp.*` | `helm get values -a` | **KEEP** unchanged. |
| `service.vault.*` | `helm get values -a` | **KEEP** unchanged. |
| `service.database.connection` | `helm get values -a` | **KEEP** secret names only; URLs derive from your PostgreSQL host (`dbInit.host`). |
| `ui.auth.oidcIssuerURL`, `ui.externalHostname` | `helm get values -a` | **READ** from the release. |
| `aap.*` | `helm get values -a` | **KEEP** unchanged. |
| `metering.*` | `helm get values -a` | **KEEP** unchanged. |
| `lvms.enabled` | `helm get values -a` | **KEEP** the live value. |
| `validation.enabled`, `hubAccess.enabled` | `helm get values -a` | **KEEP** unchanged. |
| `image` / `tag` values | `helm get values -a` | **NO.** Do not carry rc.1 tags forward; the rc.3 chart pins them. |
| DB client certs, `osac-aap-api-token`, Helm release secrets | — | **NO.** Regenerated. |
| Identity-provider and database state | — | **NO.** Managed by your platform administrators; never re-create or copy. |

---

## 6. Prepare the target chart and values

### 6.1 Values file

```bash
cd "$VALUES_DIR"
cp my-values.yaml my-values.rc3.yaml
```

Required edit — use the live value from §4:

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

### 6.2 Choose the chart

```bash
export CHART=oci://ghcr.io/osac-project/charts/osac
export CHART_VERSION=0.4.0-rc.3

helm show chart "$CHART" --version "$CHART_VERSION"
```

### 6.3 Define the shared Helm arguments once

```bash
export NS=osac
export DOMAIN=${DOMAIN:-$(helm get values osac -n "$NS" -a -o yaml \
  | yq -r '.global.clusterDomain')}

COMMON_ARGS=(
  -f "$VALUES_DIR/my-values.rc3.yaml"
  --set-string global.osacDeploymentId="$LIVE_OSAC_DEPLOYMENT_ID"
  --set global.clusterDomain="$DOMAIN"
  --set service.externalHostname="fulfillment-api-$NS.$DOMAIN"
  --set service.internalHostname="fulfillment-internal-api-$NS.$DOMAIN"
  --set ui.auth.oidcIssuerURL="https://keycloak-keycloak.$DOMAIN/realms/osac"
  --set ui.externalHostname="osac-ui-$NS.$DOMAIN"
)
```

The rc.3 chart pins every image. Do not add image overrides unless you
deliberately need to pin one component; if you do, set all of them to
`0.4.0-rc.3`.

---

## 7. Helm pre-flight

### 7.1 Diff your live values against the new chart defaults

```bash
cd "$(ls -dt "$WORK_DIR"/* | head -1)"

helm show values "$CHART" --version "$CHART_VERSION" > new-chart-defaults.yaml

diff -u <(yq -P 'sort_keys(..)' osac-values-all.yaml) \
        <(yq -P 'sort_keys(..)' new-chart-defaults.yaml) | less
```

For rc.1 → rc.3 the only expected differences are the image tags and the AAP
config-as-code branch.

### 7.2 Render the new chart and diff manifests

```bash
helm template osac "$CHART" --version "$CHART_VERSION" -n "$NS" \
  "${COMMON_ARGS[@]}" > osac-manifest-rc3.yaml

diff -u osac-manifest-rc1.yaml osac-manifest-rc3.yaml | less
```

Expected differences: images and `app.kubernetes.io/version` labels move to
`0.4.0-rc.3`, and the AAP config-as-code branch changes. Any other object
change is worth understanding before you apply.

### 7.3 Cluster-side dry run (Helm 4)

```bash
helm upgrade osac "$CHART" --version "$CHART_VERSION" -n "$NS" \
  "${COMMON_ARGS[@]}" \
  --dry-run=server --force-conflicts --hide-secret | less
```

This validates the render against the live API (CRDs, RBAC, schema) without
running hooks or changing anything.

### 7.4 Hook inventory

```bash
helm get hooks osac -n "$NS"
```

Successful hooks are deleted by Helm, so this normally shows only retained
failed hooks. On the upgrade run you expect these to succeed:

- `pre-install,pre-upgrade`: `db-init` (creates DBs/roles), validation/CA hooks
- `post-install,post-upgrade`: `create-api-token` (AAP), `create-network-class`,
  `publish-templates`, `register-local-storage` (with `lvms.enabled: true`)

### 7.5 What Helm cannot show

The `NetworkClass` status and Secret contents are not in the Helm release. If
you need them, [Appendix A](#appendix-a--optional-checks) has the minimal
non-Helm checks. The Helm-only proxy is: the values in §4/§5 are exactly what
the installer and service compare against, so copying them verbatim is the
safe path.

---

## 8. Run the patch upgrade

```bash
helm upgrade --install osac "$CHART" --version "$CHART_VERSION" -n "$NS" \
  "${COMMON_ARGS[@]}" \
  --force-conflicts \
  --wait --timeout 40m
```

Why these flags:

- `--version 0.4.0-rc.3` — the rc.3 chart pins all images to `0.4.0-rc.3`.
- `--force-conflicts` — Helm 4 + AAP CR field ownership; Helm 3 rejects it.
- no `--reuse-values` — it reuses old chart defaults and image tags; see §6.1.
- `--wait` — Helm returns only after hooks and workloads settle (AAP bootstrap
  can take 10–40 min).

Workloads keep running during this roll. The control plane restarts, so the
API, console, and UI are briefly unavailable.

---

## 9. Verify

```bash
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

# Confirm the deployed image tags are 0.4.0-rc.3
helm get manifest osac -n "$NS" \
| yq -r 'select(.kind=="Deployment") | .metadata.name + " " + (.spec.template.spec.containers[].image)'
```

Then:

- Confirm the chart reports `osac-0.4.0-rc.3` and app version `0.4.0-rc.3`.
- Confirm existing ComputeInstances are still `RUNNING` (`oc get computeinstance -A`).
- Log in to the UI at `https://osac-ui-osac.$DOMAIN`.
- Create a test ComputeInstance and confirm it reaches `RUNNING`.

If the release is `failed` / `pending-upgrade`, inspect the retained hook:

```bash
helm get hooks osac -n "$NS"
helm status osac -n "$NS" -o yaml
```

---

## 10. Current limitations

- **Minor upgrades (0.4 → 0.5) are not supported yet.** The fulfillment
  service refuses to start when the stored release and the running release
  differ at the major.minor level and any row exists in
  `virtual_networks`, `external_ip_pools`, `compute_instances`,
  `bare_metal_instances`, or `clusters`. They are out of scope for this guide.
- **A maintenance window does not change this.** The check runs at service
  startup against stored records, not against runtime availability.
- **There is no workload migration.** Deleting those resources deletes the
  actual VMs, clusters, and networking state; the release provides no export,
  import, or live-migration path across a minor release.
- **Migrations are forward-only.** The service migrates the database before it
  runs the compatibility check, and there is no downgrade. Restore a backup to
  go back.
- **Patch upgrades are different.** Patch upgrades within 0.4 run in place with
  workloads and a maintenance window. This guide covers that path.

Enforcement:
[`installation_settings.go`](../../../fulfillment-service/internal/startup/installation_settings.go)
(`ReleaseChangedMessage`, the five-table check, and `releaseChanged`).

---

## 11. Rollback

```bash
helm history osac -n "$NS"
helm rollback osac "<LAST_GOOD_REVISION>" -n "$NS" \
  --wait --timeout 40m --force-conflicts
```

Caveats:

- Helm rollback reverts Kubernetes objects, **not** the database. Migrations
  are forward-only: restore the DB dump ([Appendix A.1](#a1-database-backup-and-restore))
  before or after the rollback, otherwise the rc.1 binary may not match the
  schema.
- Rolling back the chart returns the images to `0.4.0-rc.1`. Confirm the
  previous revision and values before you roll back.

---

## 12. Gotcha checklist

- [ ] `global.osacDeploymentId` read from `helm get values -a` / manifest.
- [ ] `global.networking.provisioningEnabled` unchanged.
- [ ] `global.services` unchanged.
- [ ] `networkClass.*` defaults unchanged.
- [ ] Fulfillment and metering databases backed up.
- [ ] No `--reuse-values`.
- [ ] `--version 0.4.0-rc.3` used, not a `latest` tag.
- [ ] `--force-conflicts` on Helm 4.
- [ ] After the roll, chart is `osac-0.4.0-rc.3` and images are `0.4.0-rc.3`.
- [ ] Existing ComputeInstances still `RUNNING`.

---

## Appendix A — optional checks

Helm stores rendered manifests and values, not database rows, Secret contents,
or controller status. Use these only if you need proof.

### A.1 Database backup and restore

```bash
STAMP=$(date +%Y%m%d-%H%M%S)
pg_dump "$FULFILLMENT_DB_URL" -Fc > service-$STAMP.dump
pg_dump "$METERING_DB_URL"    -Fc > metering-$STAMP.dump
```

Restore (rollback):

```bash
pg_restore -d "$FULFILLMENT_DB_URL" --clean --if-exists < service-$STAMP.dump
```

`FULFILLMENT_DB_URL` and `METERING_DB_URL` are the PostgreSQL connection
strings your platform administrators provided.

### A.2 NetworkClass, storage records, and tenant conditions

```bash
export FULFILLMENT_API="https://$(oc get route fulfillment-internal-api -n "$NS" \
  -o jsonpath='{.spec.host}')"
export OSAC_ADMIN_TOKEN="$(oc create token admin -n "$NS" --duration 1h)"
export CA_FILE=/tmp/osac-ca.pem
oc get configmap ca-bundle -n "$NS" -o jsonpath='{.data.bundle\.pem}' > "$CA_FILE"
CURL_CA_ARGS=(--cacert "$CA_FILE")

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
