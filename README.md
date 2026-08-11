# Introduction

This Lab will take you through the process of leveraging ACM, ImageBasedUpgrade and ImageBasedGroupUpdate to manage the OpenShift upgrade process for multiple managed servers. This lab builds on existing knowledge and is not an end-to-end lab. Reader will need to understand the following concepts to use this lab:

* Image Based Installer
* ACM Server management

## Image Based Installer - Seed Requirements

* Your seed image will need to have the lifecycle agent installed
* Your seed image will need to have OpenShift API for Data Protection (OADP) installed in your seed image 
* Your seed image needs to have a separate /var/lib/containers partition

This lab will discuss upgrading Single Node OpenSHift clusters from 4.18.13 to 4.20.16.

## ACM Hub Requirements

* ACM version installed that supports your source and target OS version
* Topology Aware Lifecycle Manager (TALM) installed

### Installing Topology Aware Lifecycle Manager

Installing from the GUI:

1. In the OpenShift Container Platform web console, navigate to Operators -> OperatorHub.
2. Search for the `Topology Aware Lifecycle Manager` from the list of available Operators, and then click Install.
3. Keep the default selection of Installation mode ["All namespaces on the cluster (default)"] and Installed Namespace ("openshift-operators") to ensure that the Operator is installed properly.
4. Click Install.

Installing from the CLI/gitops

oc apply -f acm/talm

Validate that the TALM operator deployed:

```
$ oc get deploy -n openshift-operators
NAMESPACE                                          NAME                                             READY   UP-TO-DATE   AVAILABLE   AGE
openshift-operators                                cluster-group-upgrades-controller-manager        1/1     1            1           14s
```

## Manually Upgrading One Node

Prior to using ACM and `ImageBasedGroupUpdate` it is suggested that you manually update a node to understand how the process works. You will need to have a target upgrade seed image. Our example cluster `ibisno1` starts with version 4.18.13 and we will be upgrading to 4.20.16. A seed image `registry.example.com/ibu/seed-sno:v42016` has been previously captured following the documentation here: [Preparing for image-based installation for single-node OpenShift clusters ](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/edge_computing/index#ibi-preparing-for-image-based-install) 

### Local Storage

Single Node OpenShift clusters have two primary local storage options. One leverages the LocalStorageOperator (LSO), and the other uses LVM Storage Operator. We will focus on using the LocalStorageOperator. If you are using the LVM Storage Operator be sure to review the documenation on backing up persistent LVM based storage: [Example OADP CRs for cluster-scoped application artifacts for LSO and LVM Storage](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/edge_computing/index#ztp-image-based-upgrade-prep-resources).

> **WARNING**: If you are using the LSO Operator, ensure that you are using `LocalVolume` custom resource, and have NOT enabled `forceWipeDevicesAndDestroyAllData: true` as part of your configuration. Other options may lead to loss of data.

### Example Application

We will be using an example application called "Guestbook-PHP". Guestbook-PHP is a simple application, that leveraged MySQL for persistent storage. More complex applications can be used in place of Guestbook-PHP, but we will rely on this simple app to show the fundamentals of preserving the application over an upgrade. 

#### Installing the app

**This is an optional step**. 

Esnure that you are logged into the cluster from the command line, and then run the following command:

```sh
oc apply -f apps/guestbook-app
```

After the application is created, retrieve the route to access the application:

```sh
oc get route -n guestbookphp
```

Access the application and create entries in the Guestbook in order to test the data restore process later in this lab.

### Configuring OADP Backup

The process of upgrading with ImageBasedUpgrade requires backing up and restoring all hosted applications. This lab will use the backup configuration `upgrade/platform-backup-restore.yml`. This backup and restore configuration should not be directly applied, instead we will place the configuration as ConfigMap in the `openshift-oadp` Project. This will then be leveraged, by the Lifecycle Manager to start a backup before the upgrade, and start the application restore after the upgrade.

`oc create configmap oadp-cm-example --from-file=oadp-resources.yaml=platform-backup-restore.yml -n openshift-adp`

### Prepare ImageBasedUpgrade yaml

Using the example file `upgrade/manual-ibu.yml`, update the following sections

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

* Ensure that the version of OpenSHift that you will be upgrading to matches between `version` and the image version that the `image` is based on.
* Enable AutoRollback by uncommenting that section 
* If there are extra manifests that you want to apply to the cluster after it is upgraded, you can uncomment the `extraManifests` section 
* Ensure that you have included the `oadpContent` as well in order to recover your application(s) after upgrade

### Upgrading a cluster Manually.

Before upgrading mulitple clusters using the ImageBasedGroupUpgrade process we will manually update a single cluster. This will give you a good idea of the process, and how it works before applying at scale. 

Before proceeding ensure that you have configured your OADP ConfigMap for backup and restore:

`oc get cm -n opensshift-adp`

After validating that your backup config map is in place, we can apply the `ImageBasedUpgrade` to our cluster. 

```
oc apply -f manual-ibu.yml
```

Check that the current state is Idle, and that the next valid state is "Prep"

```
$ oc get imagebasedupgrade -o custom-columns="NAME:.metadata.name,DESIRED STAGE:.spec.stage,VALID NEXT STAGES:.status.validNextStages"
NAME      DESIRED STAGE   VALID NEXT STAGES
upgrade   Idle            [Prep]
```

#### Move Upgrade to Prep State

The Prep stage of the update will pull down the new base image and all required container images. This step will take some time, 

`oc patch imagebasedupgrades.lca.openshift.io upgrade -p='{"spec": {"stage": "Prep"}}' --type=merge -n openshift-lifecycle-agent`

We can check on the status of the upgrade by using the following command

```
oc describe imagebasedupgrade
Name:         upgrade
Namespace:    
Labels:       <none>
Annotations:  <none>
API Version:  lca.openshift.io/v1
Kind:         ImageBasedUpgrade
Metadata:
  Creation Timestamp:  2026-08-10T14:27:55Z
  Generation:          3
  Resource Version:    1919579
  UID:                 7ad6f18b-dc69-4eaa-bdc2-5a6f0940ac31
Spec:
  Oadp Content:
    Name:       oadp-cm-example
    Namespace:  openshift-adp
  Seed Image Ref:
    Image:    registry.example.com/ibu/seed-sno:v42016
    Version:  4.20.16
  Stage:      Prep
Status:
  Conditions:
    Last Transition Time:  2026-08-10T14:44:47Z
    Message:               In progress
    Observed Generation:   3
    Reason:                InProgress
    Status:                False
    Type:                  Idle
    Last Transition Time:  2026-08-10T14:44:47Z
    Message:               Stateroot setup job in progress. job-name: lca-prep-stateroot-setup, job-namespace: openshift-lifecycle-agent
    Observed Generation:   3
    Reason:                InProgress
    Status:                True
    Type:                  PrepInProgress
  History:
    Phases:
      Phase:            Stateroot
      Start Time:       2026-08-10T14:44:49Z
    Stage:              Prep
    Start Time:         2026-08-10T14:44:47Z
  Observed Generation:  3
  Valid Next Stages:
    Idle
Events:  <none>
```

### Pre-recorded video 

Watch this process from end to end in the following video:

[![Link to prerecorded video demonstrating the upgrade process](http://img.youtube.com/vi/fObik0DuKuM/0.jpg)](http://www.youtube.com/watch?v=fObik0DuKuM "OpenShift Image Based Upgrade - Part 1")

## Using ACM to upgrade multiple nodes

### Move to prep step
oc patch ibgu ibgu-ibisno --type=json -p \
'[{"op": "add", "path": "/spec/plan/-", "value": {"actions": ["Prep"], "rolloutStrategy": {"maxConcurrency": 2, "timeout": 30}}}]'

### Move to upgrade step
oc patch ibgu ibgu-ibisno --type=json -p \
'[{"op": "add", "path": "/spec/plan/-", "value": {"actions": ["Upgrade"], "rolloutStrategy": {"maxConcurrency": 2, "timeout": 30}}}]'

### Finalize the Upgrade
oc patch ibgu ibgu-ibisno --type=json -p \
'[{"op": "add", "path": "/spec/plan/-", "value": {"actions": ["FinalizeUpgrade"], "rolloutStrategy": {"maxConcurrency": 10, "timeout": 3}}}]'



## References

