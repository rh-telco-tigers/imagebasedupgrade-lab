# 1. Extract the reference container for your OCP version
podman pull --authfile=/home/markd/pull-secret.json registry.redhat.io/openshift4/ztp-site-generate-rhel9:v4.18
podman pull --authfile=/home/markd/pull-secret.json registry.redhat.io/openshift4/ztp-site-generate-rhel9:v4.20
mkdir -p out418
podman run --log-driver=none --rm \
  registry.redhat.io/openshift4/ztp-site-generate-rhel8:v4.18 \
  extract /home/ztp --tar | tar x -C ./out418

mkdir -p out420
podman run --log-driver=none --rm \
  registry.redhat.io/openshift4/ztp-site-generate-rhel8:v4.20 \
  extract /home/ztp --tar | tar x -C ./out420

# 2. Vendor the reference CRs into this repo (see the per-dir READMEs)
cp -r out418/source-crs/*     acm/site-policies/source-crs/v4.18/
cp -r out418/extra-manifest/* acm/site-configs/reference-crs/v4.18/extra-manifests/

# 2. Vendor the reference CRs into this repo (see the per-dir READMEs)
cp -r out420/source-crs/*     acm/site-policies/source-crs/v4.20/
cp -r out420/extra-manifest/* acm/site-configs/reference-crs/v4.20/extra-manifests/

# 3. Patch the ArgoCD instance to enable the SiteConfig + PolicyGenerator plugins
oc patch argocd openshift-gitops -n openshift-gitops --type=merge \
  --patch-file out420/argocd/deployment/argocd-openshift-gitops-patch.json

# 4. Apply the two ArgoCD Applications
oc apply -k acm/bootstrap/