# Image-Based Upgrade Lab

This lab shows how to use OpenShift **Image-Based Upgrade (IBU)** to upgrade Single Node OpenShift (SNO) clusters. It has two parts:

1. **Part 1** — Manually upgrade a single cluster with the `ImageBasedUpgrade` API so you understand each stage.
2. **Part 2** — Use ACM, Topology Aware Lifecycle Manager (TALM), and `ImageBasedGroupUpgrade` to upgrade multiple clusters.

This is not an end-to-end beginner lab. You should already be familiar with:

* Image-Based Installer (IBI) and seed images
* ACM managed-cluster concepts (required for Part 2)

This lab upgrades SNO clusters from **4.18.13** to **4.20.16**.

---

## Prerequisites

### Seed image requirements

Your seed image must:

* Include the Lifecycle Agent
* Include the OpenShift API for Data Protection (OADP) Operator
* Use a separate `/var/lib/containers` partition

A seed image such as `registry.example.com/ibu/seed-sno:v42016` should already be available. For how seed images are created, see [Preparing for image-based installation for single-node OpenShift clusters](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/image-based-installation-for-single-node-openshift#ibi-preparing-for-image-based-install).

### ACM hub requirements (Part 2)

* An ACM version that supports your source and target OpenShift versions
* Topology Aware Lifecycle Manager (TALM) installed on the hub

TALM install steps are covered in [Part 2](#part-2-using-acm-to-upgrade-multiple-clusters).

---

## Part 1: Manually upgrading a single cluster

Before you use ACM and `ImageBasedGroupUpgrade` at scale, manually upgrade one cluster. That walkthrough covers the same stages that the group upgrade later automates.

In this lab, example cluster **`ibisno1`** starts at **4.18.13** and is upgraded to **4.20.16** using seed image `registry.example.com/ibu/seed-sno:v42016`.

The Lifecycle Agent manages the upgrade through stages you set on the `ImageBasedUpgrade` custom resource:

| Stage | Purpose |
| --- | --- |
| `Idle` | Default / ready state. Also used to finalize a completed upgrade or rollback. |
| `Prep` | Pull the seed image and required container images without disrupting the running cluster. |
| `Upgrade` | Back up workloads with OADP, pivot to the new stateroot, and restore applications. |
| `Rollback` | Optional. Return to the previous stateroot if something goes wrong after upgrade. |

You move between stages by patching `spec.stage` on the `ImageBasedUpgrade` resource named `upgrade`.

### Local storage

SNO clusters typically use either the Local Storage Operator (LSO) or the LVM Storage Operator. This lab focuses on LSO. If you use LVM Storage, review the backup guidance in [Example OADP CRs for cluster-scoped application artifacts for LSO and LVM Storage](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/image-based-upgrade-for-single-node-openshift-clusters#ztp-image-based-upgrade-prep-resources).

> **Warning:** If you use LSO, use a `LocalVolume` custom resource and ensure that `forceWipeDevicesAndDestroyAllData` is set to `false`. Any other setting will result in loss of data.

### Optional: Deploy the Guestbook example application

Guestbook-PHP is a simple app that uses MariaDB with persistent storage. Deploying it is optional, but recommended so you can confirm that application data survives the upgrade.

Log in to the target cluster (`ibisno1`), then apply the manifests from the root of this repository:

```sh
oc apply -f apps/guestbook-app
```

Get the route:

```sh
oc get route -n guestbookphp
```

Open the application in a browser and create a few guestbook entries. You will use those entries later to verify restore.

### Configure the OADP backup ConfigMap

Image-based upgrade backs up and restores hosted applications with OADP. This lab uses `upgrade/platform-backup-restore.yml`, which defines Backup and Restore CRs for the ACM klusterlet and the Guestbook app.

Do **not** apply that file directly. Create it as a ConfigMap in the `openshift-adp` namespace so the Lifecycle Agent can use it during upgrade:

```sh
oc create configmap oadp-cm-example \
  --from-file=oadp-resources.yaml=upgrade/platform-backup-restore.yml \
  -n openshift-adp
```

If the ConfigMap already exists, delete it first or recreate it with `oc create ... --dry-run=client -o yaml | oc apply -f -`.

Confirm it is present:

```sh
oc get configmap oadp-cm-example -n openshift-adp
```

### Prepare the ImageBasedUpgrade resource

When the Lifecycle Agent is installed, it creates an `ImageBasedUpgrade` resource named `upgrade`. Use `upgrade/manual-ibu.yml` to set the seed image, OADP content, and related options.

Review and adjust these fields before applying:

```yaml
spec:
  stage: Idle
  seedImageRef:
    version: 4.20.16
    image: registry.example.com/ibu/seed-sno:v42016
  #  pullSecretRef: <seed_pull_secret>
  #autoRollbackOnFailure: {}
  #  initMonitorTimeoutSeconds: 1800
  #extraManifests:
  #- name: example-extra-manifests-cm
  #  namespace: openshift-lifecycle-agent
  #- name: example-catalogsources-cm
  #  namespace: openshift-lifecycle-agent
  oadpContent:
  - name: oadp-cm-example
    namespace: openshift-adp
```

Checklist:

* Set `seedImageRef.version` and `seedImageRef.image` so they match the OpenShift version in your seed image.
* Uncomment `autoRollbackOnFailure: {}` if you want automatic rollback when the post-reboot init monitor times out. Optionally set `initMonitorTimeoutSeconds` (default is 1800).
* Uncomment `extraManifests` only if you need extra ConfigMaps applied after upgrade.
* Keep `oadpContent` pointed at the ConfigMap you created so applications can be restored.

Apply the resource from the repository root while logged in to the target cluster:

```sh
oc apply -f upgrade/manual-ibu.yml
```

Confirm the resource is idle and ready for Prep:

```sh
oc get imagebasedupgrade -o custom-columns="NAME:.metadata.name,DESIRED STAGE:.spec.stage,VALID NEXT STAGES:.status.validNextStages"
```

Example output:

```text
NAME      DESIRED STAGE   VALID NEXT STAGES
upgrade   Idle            [Prep]
```

### Move to the Prep stage

Prep pulls the seed image and required container images. It does not disrupt the running workload, but it can take a significant amount of time.

```sh
oc patch imagebasedupgrades.lca.openshift.io upgrade \
  -p='{"spec": {"stage": "Prep"}}' --type=merge
```

Monitor progress:

```sh
oc describe imagebasedupgrade upgrade
```

While Prep is running you should see conditions such as `PrepInProgress`. When Prep finishes successfully, look for `PrepCompleted` with `Status: True`, and `validNextStages` should include `Upgrade` (and usually `Idle` if you want to abort and clean up).

Example while Prep is still in progress:

```text
Spec:
  Stage:  Prep
Status:
  Conditions:
    Type:    PrepInProgress
    Status:  True
    Reason:  InProgress
    Message: Stateroot setup job in progress...
  Valid Next Stages:
    Idle
```

When Prep is complete:

```sh
oc get imagebasedupgrade -o custom-columns="NAME:.metadata.name,DESIRED STAGE:.spec.stage,VALID NEXT STAGES:.status.validNextStages"
```

```text
NAME      DESIRED STAGE   VALID NEXT STAGES
upgrade   Prep            [Idle Upgrade]
```

If Prep fails, move back to `Idle` to clean up before retrying. Collect must-gather first if you need to debug.

### Move to the Upgrade stage

After Prep completes, start the upgrade. During this stage the Lifecycle Agent triggers OADP backup, pivots to the new stateroot (the node reboots), then restores applications.

```sh
oc patch imagebasedupgrades.lca.openshift.io upgrade \
  -p='{"spec": {"stage": "Upgrade"}}' --type=merge
```

The cluster will become temporarily unavailable during reboot. After it recovers, check status:

```sh
oc describe imagebasedupgrade upgrade
```

When the upgrade succeeds, you should see `UpgradeCompleted` with `Status: True`, and `validNextStages` should include `Idle` and optionally `Rollback`.

```sh
oc get imagebasedupgrade -o custom-columns="NAME:.metadata.name,DESIRED STAGE:.spec.stage,VALID NEXT STAGES:.status.validNextStages"
```

```text
NAME      DESIRED STAGE   VALID NEXT STAGES
upgrade   Upgrade         [Idle Rollback]
```

Verify the cluster version:

```sh
oc get clusterversion
```

If you deployed Guestbook, confirm the app and your test entries were restored:

```sh
oc get route -n guestbookphp
```

### Finalize the upgrade

When you are satisfied with the result, move back to `Idle` to finalize. This cleans up the previous stateroot and completes the upgrade. After this step you can no longer roll back.

```sh
oc patch imagebasedupgrades.lca.openshift.io upgrade \
  -p='{"spec": {"stage": "Idle"}}' --type=merge
```

Confirm the resource has returned to Idle:

```sh
oc get imagebasedupgrade -o custom-columns="NAME:.metadata.name,DESIRED STAGE:.spec.stage,VALID NEXT STAGES:.status.validNextStages"
```

```text
NAME      DESIRED STAGE   VALID NEXT STAGES
upgrade   Idle            [Prep]
```

### Optional: Roll back instead of finalizing

If the upgrade completed but you need to revert before finalizing, move to `Rollback` instead of `Idle`:

```sh
oc patch imagebasedupgrades.lca.openshift.io upgrade \
  -p='{"spec": {"stage": "Rollback"}}' --type=merge
```

After a successful rollback, finalize by moving to `Idle` the same way as above.

### Pre-recorded video

Watch an end-to-end walkthrough of this process:

[![Link to prerecorded video demonstrating the upgrade process](https://img.youtube.com/vi/fObik0DuKuM/0.jpg)](https://www.youtube.com/watch?v=fObik0DuKuM "OpenShift Image Based Upgrade - Part 1")

---

## Part 2: Using ACM to upgrade multiple clusters

Part 1 walked through each `ImageBasedUpgrade` stage on a single cluster. Part 2 uses the same stages at scale from the **ACM hub**, with Topology Aware Lifecycle Manager (TALM) driving an `ImageBasedGroupUpgrade` (IBGU) across several managed clusters.

In this lab, `upgrade/ibgu-ibisno.yaml` targets managed clusters **`ibisno1`**, **`ibisno3`**, and **`ibisno4`** (selected by the ManagedCluster `name` label) and upgrades them from **4.18.13** to **4.20.16**.

All commands in this section run on the **hub cluster** unless noted otherwise.

### How ImageBasedGroupUpgrade relates to Part 1

| Part 1 (`ImageBasedUpgrade`) | Part 2 (`ImageBasedGroupUpgrade`) |
| --- | --- |
| Applied on each managed cluster | Applied once on the hub |
| You patch `spec.stage` (`Idle` → `Prep` → `Upgrade` → `Idle`) | You append plan **actions** (`Prep`, `Upgrade`, `FinalizeUpgrade`) |
| One cluster at a time | TALM rolls out to many clusters with `maxConcurrency` and `timeout` |
| You watch one IBU status | IBGU aggregates per-cluster `completedActions`, `currentAction`, and `failedActions` |

IBGU actions map to the stages you used in Part 1:

| Action | Effect on each selected cluster |
| --- | --- |
| `Prep` | Move IBU to `Prep` |
| `Upgrade` | Move IBU to `Upgrade` |
| `FinalizeUpgrade` | Move IBU back to `Idle` after a successful upgrade |
| `Abort` / `AbortOnFailure` | Cancel and return failed or not-yet-upgraded clusters to `Idle` |
| `Rollback` / `FinalizeRollback` | Roll back successfully upgraded clusters, then finalize |

You can put several actions in one plan step (fully automated) or add them one step at a time (more control). This lab uses the **step-by-step** approach so you can evaluate each stage before continuing—the same learning path as Part 1.

> **Note:** `rolloutStrategy.timeout` is in **minutes**.

### Prerequisites for Part 2

Before you start:

* Complete Part 1 on at least one cluster so you understand Prep, Upgrade, and finalize behavior.
* Log in to the **hub** with `cluster-admin`.
* Confirm the managed clusters are available:

```sh
oc get managedclusters
```

* On each target managed cluster, ensure:
  * Lifecycle Agent and OADP are installed (typically via the seed image and hub policies).
  * The OADP backup ConfigMap `oadp-cm-example` exists in `openshift-adp` (same ConfigMap used in Part 1; hub policies under `acm/site-policies` can deploy it).
* If you already finalized an upgrade on `ibisno1` in Part 1, either remove it from `clusterLabelSelectors` in `upgrade/ibgu-ibisno.yaml` or leave it out of this run so you are not re-upgrading a cluster that is already on the target version.

### Install Topology Aware Lifecycle Manager

TALM must be installed on the hub. It reconciles the `ImageBasedGroupUpgrade` resource and creates the underlying rollout resources.

#### Install from the web console

1. In the OpenShift Container Platform web console on the hub, navigate to **Operators → OperatorHub**.
2. Search for **Topology Aware Lifecycle Manager**, then click **Install**.
3. Keep the default installation mode (**All namespaces on the cluster**) and installed namespace (**openshift-operators**).
4. Click **Install**.

#### Install from the CLI

Create a Subscription on the hub:

```sh
oc apply -f - <<'EOF'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-topology-aware-lifecycle-manager-subscription
  namespace: openshift-operators
spec:
  channel: stable
  name: topology-aware-lifecycle-manager
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

#### Validate the install

```sh
oc get deploy -n openshift-operators cluster-group-upgrades-controller-manager
```

Example output:

```text
NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
cluster-group-upgrades-controller-manager   1/1     1            1           14s
```

### Prepare the ImageBasedGroupUpgrade resource

Review `upgrade/ibgu-ibisno.yaml` before applying it:

```yaml
apiVersion: lcm.openshift.io/v1alpha1
kind: ImageBasedGroupUpgrade
metadata:
  name: ibgu-ibisno
  namespace: default
spec:
  clusterLabelSelectors:
    - matchExpressions:
      - key: name
        operator: In
        values:
        - ibisno1
        - ibisno3
        - ibisno4
  ibuSpec:
    seedImageRef:
      image: registry.example.com/ibu/seed-sno:v42016
      version: 4.20.16
    oadpContent:
      - name: oadp-cm-example
        namespace: openshift-adp
  plan:
    - actions: ["Prep"]
      rolloutStrategy:
        maxConcurrency: 2
        timeout: 2400
```

Checklist:

* Adjust `clusterLabelSelectors` if your ManagedCluster names differ.
* Ensure `ibuSpec.seedImageRef.version` and `image` match your seed image.
* Ensure `oadpContent` matches the ConfigMap name and namespace on each managed cluster.
* The initial plan includes only `Prep`. You will append `Upgrade` and `FinalizeUpgrade` after Prep completes.

Apply the IBGU on the hub:

```sh
oc apply -f upgrade/ibgu-ibisno.yaml
```

### Monitor IBGU status

Watch aggregated status from the hub:

```sh
oc get ibgu ibgu-ibisno -o yaml
```

Useful fields under `status.clusters`:

* `completedActions` — actions that finished successfully on that cluster
* `currentAction` — action currently in progress
* `failedActions` — failed actions and error details

Example while Prep is completing:

```yaml
status:
  clusters:
  - name: ibisno1
    completedActions:
    - action: Prep
  - name: ibisno3
    completedActions:
    - action: Prep
  - name: ibisno4
    failedActions:
    - action: Prep
```

TALM also labels ManagedClusters as stages complete or fail, for example:

* `lcm.openshift.io/ibgu-prep-completed`
* `lcm.openshift.io/ibgu-prep-failed`
* `lcm.openshift.io/ibgu-upgrade-completed`
* `lcm.openshift.io/ibgu-upgrade-failed`

Inspect labels:

```sh
oc get managedclusters --show-labels | grep ibgu
```

Wait until the current plan step has completed or failed on all selected clusters before adding the next step. You cannot remove actions from an in-progress plan; you only append new steps.

### Optional: Abort clusters that failed Prep

If some clusters fail Prep, investigate them first. To return failed clusters to `Idle` and clean up upgrade resources, append `AbortOnFailure`:

```sh
oc patch ibgu ibgu-ibisno --type=json -p \
'[{"op": "add", "path": "/spec/plan/-", "value": {"actions": ["AbortOnFailure"], "rolloutStrategy": {"maxConcurrency": 5, "timeout": 10}}}]'
```

Continue monitoring with `oc get ibgu ibgu-ibisno -o yaml` until that step finishes.

### Add the Upgrade action

When Prep has completed successfully on the clusters you want to upgrade, append the Upgrade step:

```sh
oc patch ibgu ibgu-ibisno --type=json -p \
'[{"op": "add", "path": "/spec/plan/-", "value": {"actions": ["Upgrade"], "rolloutStrategy": {"maxConcurrency": 2, "timeout": 30}}}]'
```

This starts the same disruptive upgrade path you saw in Part 1 (OADP backup, pivot/reboot, restore), rolled out with up to two clusters at a time and a 30-minute timeout per action batch.

Monitor until Upgrade completes or fails:

```sh
oc get ibgu ibgu-ibisno -o yaml
```

Optionally abort clusters that failed Upgrade:

```sh
oc patch ibgu ibgu-ibisno --type=json -p \
'[{"op": "add", "path": "/spec/plan/-", "value": {"actions": ["AbortOnFailure"], "rolloutStrategy": {"maxConcurrency": 5, "timeout": 10}}}]'
```

### Finalize the upgrade

When you are satisfied with the upgraded clusters, append `FinalizeUpgrade`. That moves successful clusters back to `Idle` and cleans up the previous stateroot—the group equivalent of finalizing in Part 1.

```sh
oc patch ibgu ibgu-ibisno --type=json -p \
'[{"op": "add", "path": "/spec/plan/-", "value": {"actions": ["FinalizeUpgrade"], "rolloutStrategy": {"maxConcurrency": 10, "timeout": 3}}}]'
```

Verify completion:

```sh
oc get ibgu ibgu-ibisno -o yaml
```

Example successful cluster entry:

```yaml
status:
  clusters:
  - name: ibisno3
    completedActions:
    - action: Prep
    - action: Upgrade
    - action: FinalizeUpgrade
```

On a managed cluster, you can also confirm the version and that IBU is idle again:

```sh
oc get clusterversion
oc get imagebasedupgrade -o custom-columns="NAME:.metadata.name,DESIRED STAGE:.spec.stage,VALID NEXT STAGES:.status.validNextStages"
```

### Optional: Roll back upgraded clusters

If Upgrade succeeded but you need to revert **before** finalizing, create a **new** `ImageBasedGroupUpgrade` that selects the upgraded clusters and uses a rollback plan:

```yaml
apiVersion: lcm.openshift.io/v1alpha1
kind: ImageBasedGroupUpgrade
metadata:
  name: ibgu-ibisno-rollback
  namespace: default
spec:
  clusterLabelSelectors:
    - matchExpressions:
      - key: name
        operator: In
        values:
        - ibisno3
        - ibisno4
  ibuSpec:
    seedImageRef:
      image: registry.example.com/ibu/seed-sno:v42016
      version: 4.20.16
    oadpContent:
      - name: oadp-cm-example
        namespace: openshift-adp
  plan:
    - actions: ["Rollback", "FinalizeRollback"]
      rolloutStrategy:
        maxConcurrency: 2
        timeout: 30
```

`Rollback` applies to clusters that successfully completed Upgrade. After finalize (upgrade or rollback), the related IBGU completion/failure labels are cleared.

### Letting TALM run the full upgrade automatically

The step-by-step patches above give you control between stages. You can also let TALM drive the entire upgrade on its own by putting all actions in a single plan step:

```yaml
plan:
  - actions: ["Prep", "Upgrade", "FinalizeUpgrade"]
    rolloutStrategy:
      maxConcurrency: 2
      timeout: 2400
```

With that plan, TALM moves each selected cluster through Prep, Upgrade, and FinalizeUpgrade without waiting for you to append the next action. This is useful when service interruption timing is less of a concern and you want a hands-off rollout. The trade-off is that you can only troubleshoot failed clusters after the combined step finishes—clusters that fail one action in the step skip the remaining actions in that same step.

The `rolloutStrategy.maxConcurrency` setting controls how many clusters TALM upgrades at the same time. In the plan above, and in `upgrade/ibgu-ibisno.yaml`, `maxConcurrency` is set to `2`, so TALM works on only two of the selected clusters at once. When one of those clusters finishes the current work, TALM can start another, always keeping at most two in flight. Raise this value to upgrade more clusters in parallel, or lower it when you need a more gradual rollout.

To use the fully automated plan with this lab's IBGU, replace the Prep-only `plan` in `upgrade/ibgu-ibisno.yaml` with the combined actions shown above before you apply it (or edit the live resource), instead of starting with Prep-only and patching later.

### Pre-recorded video

Watch an end-to-end walkthrough of the group upgrade process:

[![Link to prerecorded video demonstrating the group upgrade process](https://img.youtube.com/vi/uJ5bnUCVBjU/0.jpg)](https://www.youtube.com/watch?v=uJ5bnUCVBjU "OpenShift Image Based Upgrade - GroupUpgrade - Part 2")

---

## References

Documentation links below are for OpenShift Container Platform 4.20, the target version used in this lab.

### Image-based installation and seed images

* [Preparing for image-based installation for single-node OpenShift clusters](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/image-based-installation-for-single-node-openshift#ibi-preparing-for-image-based-install)
* [Generating a seed image for the image-based upgrade with the Lifecycle Agent](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/image-based-upgrade-for-single-node-openshift-clusters#cnf-image-based-upgrade-generate-seed-image)

### Image-based upgrade

* [Understanding the image-based upgrade for single-node OpenShift clusters](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/image-based-upgrade-for-single-node-openshift-clusters#cnf-understanding-image-based-upgrade)
* [Preparing for an image-based upgrade for single-node OpenShift clusters](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/image-based-upgrade-for-single-node-openshift-clusters#preparing-for-an-image-based-upgrade-for-single-node-openshift-clusters)
* [Performing an image-based upgrade with the Lifecycle Agent](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/image-based-upgrade-for-single-node-openshift-clusters#cnf-image-based-upgrade) (Part 1)
* [Performing an image-based upgrade using GitOps ZTP](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/image-based-upgrade-for-single-node-openshift-clusters#ztp-image-based-upgrade) (`ImageBasedGroupUpgrade` / Part 2)
* [Managing the image-based upgrade at scale using the ImageBasedGroupUpgrade CR](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/image-based-upgrade-for-single-node-openshift-clusters#ztp-image-based-upgrade-concept_ztp-gitops)
* [Performing an image-based upgrade on managed clusters at scale in several steps](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/image-based-upgrade-for-single-node-openshift-clusters#ztp-image-based-upgrade-procedure-steps_ztp-gitops)
* [Performing an image-based upgrade on managed clusters at scale in one step](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/image-based-upgrade-for-single-node-openshift-clusters#ztp-image-based-upgrade-procedure-one-step_ztp-gitops)

### Topology Aware Lifecycle Manager

* [Updating managed clusters with the Topology Aware Lifecycle Manager](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/cnf-talm-for-cluster-updates)
* [Installing the Topology Aware Lifecycle Manager by using the web console](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/cnf-talm-for-cluster-updates#installing-topology-aware-lifecycle-manager-using-web-console_cnf-topology-aware-lifecycle-manager)
* [Installing the Topology Aware Lifecycle Manager by using the CLI](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/cnf-talm-for-cluster-updates#installing-topology-aware-lifecycle-manager-using-cli_cnf-topology-aware-lifecycle-manager)

### Backup and restore (OADP)

* [OADP backup and restore guidelines](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/image-based-upgrade-for-single-node-openshift-clusters#ztp-image-based-upgrade-backup-guide_understanding-image-based-upgrade)
* [Creating OADP ConfigMap objects for the image-based upgrade with Lifecycle Agent](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/image-based-upgrade-for-single-node-openshift-clusters#cnf-image-based-upgrade-prep-oadp_cnf-non-gitops)
* [Example OADP CRs for cluster-scoped application artifacts for LSO and LVM Storage](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/image-based-upgrade-for-single-node-openshift-clusters#ztp-image-based-upgrade-prep-resources)
