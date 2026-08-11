# Introduction

The files in this `acm` directory are sample GitOps Zero Touch Provisioning (ZTP) content for deploying Single Node OpenShift (SNO) clusters with Red Hat Advanced Cluster Management (ACM) and OpenShift GitOps. They follow the usual ZTP repository layout:

* `bootstrap/` — Argo CD Applications that sync site and policy content from this repository to the hub
* `site-configs/` — Per-site install manifests (`ClusterInstance`, secrets, and namespaces) for clusters such as `ibisno1`–`ibisno4` and the seed cluster
* `site-policies/` — Day-2 configuration delivered as ACM policies (for example OADP backup settings used later by image-based upgrade)

Use these manifests as a starting point for your own hub Git repository. Building or operating this environment is **outside the scope of the main image-based upgrade lab**; follow the Red Hat documentation below for the full ACM and GitOps ZTP process.

# References

* [Preparing the hub cluster for GitOps ZTP](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/ztp-preparing-the-hub-cluster) — hub prerequisites, Git repository layout, and Argo CD Applications (`bootstrap/`)
* [Installing managed clusters with RHACM and SiteConfig resources](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/ztp-deploying-far-edge-sites) — deploying fleets of managed sites from Git (`site-configs/`)
* [Managing cluster policies with PolicyGenerator resources](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/managing-cluster-policies-with-policygenerator-resources) — day-2 configuration through ACM policies (`site-policies/`)
* [Image-based installation for single-node OpenShift](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/edge_computing/image-based-installation-for-single-node-openshift) — image-based SNO install flow used by the sample `ClusterInstance` sites in this directory
